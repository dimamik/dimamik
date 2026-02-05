---
title: "BM25 Search in Ecto with Torus"
date: 2026-01-27T00:00:00Z
description: BM25 (Best Matching 25) is a modern ranking function for full-text search that provides better relevance than traditional TF-IDF algorithms. Torus integrates BM25 search into your Ecto queries through the pg_textsearch PostgreSQL extension.
tags:
  - torus
  - search
  - elixir
categories:
  - Torus
draft: false
ShowToc: true
---

## What is Torus?

[Torus](https://hexdocs.pm/torus/Torus.html) bridges Ecto and PostgreSQL, simplifying building search queries.

With Torus, instead of writing complex macros, you could just add any preferred search type in a single query:

```elixir
# For example - semantic search
embedding_vector = Torus.to_vector("A magic school in the UK")

Post
|> Torus.semantic([p], p.embedding, embedding_vector)
|> select([p], p.title)
|> Repo.all()
["Diagon Bombshell"]
```

Currently, Torus supports pattern match, similarity, full-text ([TF-IDF](https://en.wikipedia.org/wiki/Tf%E2%80%93idf#:~:text=In%20information%20retrieval%2C%20tf%E2%80%93idf,appear%20more%20frequently%20in%20general.) and [BM25](https://en.wikipedia.org/wiki/Okapi_BM25)), semantic search, with plans to expand further. In addition to abstracting complex PostgreSQL functions, Torus provides full guides on tradeoffs and ways to add them to your application straight in the docs.

You could see all available search types in action [on the demo page](https://torus.dimamik.com/?method=bm25)

## Full text search

Full-text search finds documents containing specific words or phrases, handling linguistic variations like stemming ("running" matches "run") and stop word removal.

There are two issues with full text search. Relevance, and speed. PostgreSQL's default `ts_rank` uses TF-IDF (Term Frequency–Inverse Document Frequency), which works but has known limitations:

- **No term saturation**: TF-IDF keeps rewarding repeated terms linearly. A document mentioning "database" 50 times scores much higher than one mentioning it 5 times, even though both are clearly about databases.
- **Weak length normalization**: Short documents and long documents compete on unequal footing without careful tuning.
- **No top-k optimization**: PostgreSQL must score _every_ matching document before sorting, even if you only want the top 10.

In terms of speed: GIN indexes on `tsvector` columns are efficient for filtering (finding documents that match), but ranking still requires computing `ts_rank` for every matched row. With millions of documents and broad queries, this becomes a bottleneck.

BM25 (Best Matching 25) addresses both problems. It's the same algorithm powering Elasticsearch and other modern search engines. The key improvements:

- **Term frequency saturation**: Diminishing returns for repeated terms. The 5th occurrence of a word contributes less than the 1st.
- **Document length normalization**: Built into the formula via the `b` parameter - short documents aren't unfairly penalized.
- **Block-Max WAND**: The pg_textsearch implementation can skip scoring documents that can't make the top-k, making `ORDER BY score LIMIT 10` queries significantly faster on large tables.

## When to Use BM25

**Use [`Torus.bm25/5`][bm25] when:**

- You need state-of-the-art relevance ranking
- You're searching a single column
- You have top-k queries (ORDER BY + LIMIT)
- You're on PostgreSQL 17+
- Performance is critical for large result sets

**Use [`Torus.full_text/5`][full_text] instead when:**

- You need multi-column search with different weights
- You want to use stored tsvector columns
- You're on PostgreSQL < 17
- You need the `:concat` filter type

## How to use BM25

### 1. Add Torus to Your Mix Dependencies

Add Torus to your `mix.exs` dependencies:

```elixir
def deps do
  [
    {:torus, ">= 0.6"}
  ]
end
```

### 2. Install pg_textsearch Extension

First, install the `pg_textsearch` extension in your database. For full installation steps (including
building from source), see the [pg_textsearch installation guide](https://github.com/timescale/pg_textsearch#installation):

Next, we need to enable the extension via a migration:

```elixir
defmodule MyApp.Repo.Migrations.InstallPgTextsearch do
  use Ecto.Migration

  def change do
    execute "CREATE EXTENSION IF NOT EXISTS pg_textsearch",
      "DROP EXTENSION IF EXISTS pg_textsearch"
  end
end
```

### 3. Create BM25 Indexes

BM25 requires indexes on the columns you want to search. Create them in a migration:

```elixir
defmodule MyApp.Repo.Migrations.CreateBm25Indexes do
  use Ecto.Migration

  def up do
    # Basic index with default parameters
    execute """
    CREATE INDEX posts_body_bm25_idx ON posts
    USING bm25(body) WITH (text_config='english')
    """

    # Custom parameters (optional)
    execute """
    CREATE INDEX posts_title_bm25_idx ON posts
    USING bm25(title)
    WITH (text_config='english', k1=1.5, b=0.8)
    """
  end

  def down do
    execute "DROP INDEX IF EXISTS posts_body_bm25_idx"
    execute "DROP INDEX IF EXISTS posts_title_bm25_idx"
  end
end
```

#### BM25 Index Parameters

- **`text_config`** (required): PostgreSQL text search configuration (english, french, german, etc.)
- **`k1`** (default: `1.2`): Term frequency saturation parameter (range: 0.1-10.0)
  - Higher values = more emphasis on term frequency
  - Lower values = saturation kicks in sooner
- **`b`** (default: `0.75`): Length normalization parameter (range: 0.0-1.0)
  - 1.0 = full length normalization
  - 0.0 = no length normalization

## Basic Usage

### Simple Search

```elixir
# Find top 10 most relevant posts - blazingly fast
Post
|> Torus.bm25([p], p.body, "database system")
|> limit(10)
|> Repo.all()
```

### With Score Selection

```elixir
# Include relevance scores in results
Post
|> Torus.bm25([p], p.body, "database", score_key: :relevance)
|> limit(10)
|> select([p], %{id: p.id, body: p.body})
|> Repo.all()
# => [%{id: 1, body: "...", relevance: -5.2}, ...]
```

Note: BM25 scores are **negative** - lower scores indicate better matches.

### With Pre-filtering

```elixir
# Filter by category before searching
Post
|> where([p], p.category_id == 123)
|> Torus.bm25([p], p.body, "database")
|> limit(10)
|> Repo.all()
```

### With Score Threshold (Post-filtering)

```elixir
# Only return results with score better than -5.0
# (i.e., scores like -4.0, -3.0, -2.0 which indicate better matches)
Post
|> Torus.bm25([p], p.body, "database", score_threshold: -5.0)
|> Repo.all()
```

Note: This is post-filtering - applied after `ORDER BY`, and it may return fewer results than LIMIT

## Advanced Usage

### Explicit Index Name

When using BM25 inside PL/pgSQL functions or stored procedures, you must specify the index name explicitly:

```elixir
Post
|> Torus.bm25([p], p.body, "database", index_name: "posts_body_bm25_idx")
|> limit(10)
|> Repo.all()
```

### Multi-word Queries

BM25 handles multi-word queries naturally:

```elixir
Post
|> Torus.bm25([p], p.body, "relational database system")
|> limit(5)
|> Repo.all()
```

### Phrase Search

Use quotes for exact phrase matching:

```elixir
Post
|> Torus.bm25([p], p.body, "\"relational database\"")
|> limit(5)
|> Repo.all()
```

### Exclude Non-Matches

Use `pre_filter: true` to add a WHERE clause that excludes rows that don't match at all
(non-matches have score = 0, matches have negative scores):

```elixir
Post
|> Torus.bm25([p], p.body, "database", pre_filter: true)
|> limit(10)
|> Repo.all()
```

## Multi-Column Search

BM25 indexes work on **single columns only**. For multi-column search, use a generated column:

### Option 1: Generated Column

```elixir
defmodule MyApp.Repo.Migrations.AddSearchableColumn do
  use Ecto.Migration

  def up do
    # Add generated column combining title and body
    execute """
    ALTER TABLE posts
    ADD COLUMN searchable_text TEXT
    GENERATED ALWAYS AS (title || ' ' || body) STORED
    """

    # Create BM25 index on generated column
    execute """
    CREATE INDEX posts_searchable_bm25_idx ON posts
    USING bm25(searchable_text) WITH (text_config='english')
    """
  end

  def down do
    execute "DROP INDEX IF EXISTS posts_searchable_bm25_idx"
    execute "ALTER TABLE posts DROP COLUMN IF EXISTS searchable_text"
  end
end
```

Then search the generated column:

```elixir
Post
|> Torus.bm25([p], p.searchable_text, "search term")
|> limit(10)
|> Repo.all()
```

### Option 2: Multiple Searches + Score Combination

For more control, search each column separately and combine scores:

```elixir
# This requires indexes on both columns
defmodule MyApp.Posts do
  def multi_column_search(query_text) do
    from p in Post,
      select: %{
        id: p.id,
        title: p.title,
        body: p.body,
        combined_score:
          fragment("(? <@> to_bm25query(?, 'posts_title_bm25_idx')) * 1.0 +
                    (? <@> to_bm25query(?, 'posts_body_bm25_idx')) * 0.4",
            p.title, ^query_text,
            p.body, ^query_text
          )
      },
      order_by: fragment("combined_score ASC"),
      limit: 10
  end
end
```

If this use-case becomes more common, we could extend `Torus` implementation to handle such cases.

## Performance Optimization

### 1. Create Appropriate Indexes

BM25 requires indexes to work efficiently:

```sql
-- Basic index
CREATE INDEX posts_body_bm25_idx ON posts
USING bm25(body) WITH (text_config='english');

-- For pre-filtering, add B-tree index on filter columns
CREATE INDEX posts_category_idx ON posts (category_id);
```

### 2. Use LIMIT for Top-k Optimization

BM25 is most efficient with `ORDER BY + LIMIT` (Block-Max WAND optimization):

```elixir
# Efficient - uses top-k optimization
Post
|> Torus.bm25([p], p.body, "database")
|> limit(10)
|> Repo.all()

# Less efficient - scans all matches
Post
|> Torus.bm25([p], p.body, "database")
|> Repo.all()
```

### 3. Pre-filtering Strategy

**Best case:** Selective pre-filter (<10% of rows) + BM25 with LIMIT

```elixir
# Good: Category filter is selective
Post
|> where([p], p.category_id == 123)  # Matches 5% of rows
|> Torus.bm25([p], p.body, "database")
|> limit(10)
|> Repo.all()
```

**Watch out:** Non-selective pre-filter (>50% of rows) + no LIMIT

```elixir
# Less efficient: Filter matches many rows, no LIMIT
Post
|> where([p], p.published == true)  # Matches 90% of rows
|> Torus.bm25([p], p.body, "database")
|> Repo.all()
```

### 4. Score Thresholds

Using `score_threshold` applies post-filtering - you may get fewer results than LIMIT.

Since BM25 scores are negative (lower = better match), `score_threshold: -3.0` keeps only
results with score < -3.0 (i.e., -4.0, -5.0, -6.0 are kept; -2.0, -1.0 are filtered out):

```elixir
# May return fewer than 10 results
Post
|> Torus.bm25([p], p.body, "database", score_threshold: -3.0)
|> limit(10)
|> Repo.all()
```

To guarantee 10 results, increase LIMIT and re-limit in application:

```elixir
# Get more results, then trim
results =
  Post
  |> Torus.bm25([p], p.body, "database", score_threshold: -3.0)
  |> limit(50)
  |> Repo.all()
  |> Enum.take(10)
```

## Language Support

BM25 language/stemming is configured **at index creation time** via the `text_config` parameter,
not at query time. Create separate indexes for different languages:

```sql
-- English index (with stemming: "searching" matches "search")
CREATE INDEX posts_body_en_idx ON posts
USING bm25(body) WITH (text_config='english');

-- French index
CREATE INDEX posts_body_fr_idx ON french_posts
USING bm25(body) WITH (text_config='french');

-- Simple index (no stemming: exact word matches only)
CREATE INDEX posts_body_simple_idx ON posts
USING bm25(body) WITH (text_config='simple');
```

Then query with the appropriate index:

```elixir
# Uses English stemming from the index
Post
|> Torus.bm25([p], p.body, "searching", index_name: "posts_body_en_idx")
|> limit(10)
|> Repo.all()
```

List available text search configurations:

```sql
SELECT cfgname FROM pg_ts_config;
```

## Debugging and Monitoring

### View Query Plan

```elixir
Post
|> Torus.bm25([p], p.body, "database")
|> limit(10)
|> Torus.QueryInspector.tap_explain_analyze()
|> Repo.all()
```

### View Generated SQL

```elixir
Post
|> Torus.bm25([p], p.body, "database", score_key: :score)
|> limit(10)
|> Torus.QueryInspector.substituted_sql()
|> IO.puts()
```

### Monitor Index Usage

```sql
-- Check if indexes are being used
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE indexrelid::regclass::text LIKE '%bm25%';
```

## Comparison: BM25 vs Full Text (TF-IDF)

| Feature            | [`bm25/5`][bm25]           | [`full_text/5`][full_text] |
| ------------------ | -------------------------- | -------------------------- |
| Algorithm          | BM25                       | TF-IDF (ts_rank)           |
| Relevance          | Generally better           | Good                       |
| Multi-column       | Single (use generated col) | Native support             |
| Weights per column | Manual combination         | Built-in                   |
| Top-k optimization | Yes (Block-Max WAND)       | No                         |
| Stored tsvector    | No                         | Yes                        |
| PostgreSQL version | 17+                        | 12+                        |
| Index type         | `bm25(column)`             | GIN/GiST on tsvector       |
| Query syntax       | Direct text                | Requires tsquery           |

## Common Patterns

### Search with Pagination

```elixir
def search_posts(query_text, page, per_page) do
  offset = (page - 1) * per_page

  Post
  |> Torus.bm25([p], p.body, query_text, score_key: :relevance)
  |> limit(^per_page)
  |> offset(^offset)
  |> select([p], %{id: p.id, body: p.body})
  |> Repo.all()
end
```

### Search with Faceted Filters

```elixir
def search_with_facets(query_text, filters) do
  query = from p in Post

  query =
    if filters[:category_id] do
      where(query, [p], p.category_id == ^filters[:category_id])
    else
      query
    end

  query =
    if filters[:published] do
      where(query, [p], p.published == ^filters[:published])
    else
      query
    end

  query
  |> Torus.bm25([p], p.body, query_text, score_key: :relevance)
  |> limit(20)
  |> Repo.all()
end
```

### Combining BM25 with Other Search Types

```elixir
# First try BM25 (exact matches)
bm25_results =
  Post
  |> Torus.bm25([p], p.body, query_text, score_threshold: -5.0)
  |> limit(10)
  |> Repo.all()

# If not enough results, fall back to similarity search
if length(bm25_results) < 5 do
  similarity_results =
    Post
    |> where([p], p.id not in ^Enum.map(bm25_results, & &1.id))
    |> Torus.similarity([p], p.body, query_text)
    |> limit(5)
    |> Repo.all()

  bm25_results ++ similarity_results
else
  bm25_results
end
```

## Troubleshooting

### Extension Not Found

```
** (Postgrex.Error) ERROR 42704 (undefined_object) type "bm25query" does not exist
```

**Solution:** Install the pg_textsearch extension:

```sql
CREATE EXTENSION pg_textsearch;
```

### Index Not Found

```
** (Postgrex.Error) ERROR 42883 (undefined_function) operator does not exist: text <@> unknown
```

**Solution:** Create a BM25 index on the column:

```sql
CREATE INDEX posts_body_bm25_idx ON posts
USING bm25(body) WITH (text_config='english');
```

### No Results Returned

**Check:**

1. Is the index created? `\di posts_body_bm25_idx`
2. Does the table have data? `SELECT COUNT(*) FROM posts;`
3. Is the search term empty? Try with a known term
4. Is `score_threshold` too restrictive? Remove or adjust it

### Slow Queries

**Check:**

1. Is the query using the index? Run `EXPLAIN ANALYZE`
2. Add LIMIT clause for top-k optimization
3. Create B-tree indexes on pre-filter columns
4. Consider if pre-filter is too broad (>50% of rows)

## Further Reading

- [pg_textsearch GitHub](https://github.com/timescale/pg_textsearch)
- [BM25 Algorithm on Wikipedia](https://en.wikipedia.org/wiki/Okapi_BM25)
- [PostgreSQL Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [Torus Documentation](https://hexdocs.pm/torus)

[bm25]: https://hexdocs.pm/torus/Torus.html#bm25/5
[full_text]: https://hexdocs.pm/torus/Torus.html#full_text/5
