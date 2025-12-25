---
title: "Let constraints drive consistency"
date: 2024-05-10T00:00:00Z
description: Best practices, derived from my own experience, that help us to enforce data consistency in a large production database (PostgreSQL and Ecto).
pinned: true
tags:
  - general
  - data consistency
  - postgresql
categories:
  - General
draft: false
cover:
  image: /img/posts/general/let_constraints_drive_consistency.webp
---

# TL;DR

- [When adding a new column (when it's not too late yet)](<#when-adding-a-new-column-(when-it's-not-too-late)%3A>)
  - [Always add non-null db constraints (unless completely infeasible)](#non_null_constraints)
  - [Think of ON DELETE policies](#on_delete)
  - [Make sure time is always UTC](#utc_time)
  - [Make sure dates and times are always of the same data type](#same_data_type)
  - [Do not prematurely create indexes (unless completely necessary)](#premature_indexes)
  - [Use timestamps instead of (or alongside) status flags](#timestamps_vs_status_flags)
  - [Mirror validations in database](#mirror_validations_in_database)
  - [Avoid using Ecto's embedded schemas for data managed by the application (opinionated)](#embedded_schemas)
- [When creating a new table](#when-creating-a-new-table%3A)
  - [Consider using soft deletes by adding `deleted_at` column](#soft_deletes)
- [When updating an existing table](#when-updating-an-existing-table%3A)

# About me

I've been working with a large production database containing hundreds of millions of rows for more than 3 years now, and the biggest pain point has always been an inconsistency in the data shape:

- A user without an email, even though we fetch and log in users by email — no problem, **5k+** of these
- A user with an empty (but validated) phone number — no problem, your database has you covered
- A many-to-many relationship table row that has no relations — we have one as well
- Breaking a non-spoken (but obvious) unique constraint — most definitely, almost everywhere
- And many, many more. I'm sure you know them!

So how do we prevent this? What should you do if you already have this? How can you gradually enforce data consistency in a large production database? That's what we'll cover in this post.

## When adding a new column (when it's not too late):

- <span id="non_null_constraints" /> **Always add non-null db constraints (unless completely infeasible)**

  Non-null constraints ensure that critical fields are never left empty, improving data integrity and preventing unexpected errors. For example, a nullable boolean introduces a third state, so always enforce non-null constraints at the database level when possible.

- <span id="on_delete" /> **Think of ON DELETE policies**

  Make sure to make a conscious decision about when and how your records should be deleted if the associated records are deleted. For example, if you have a user, and a user has many posts, what should happen to the posts when the user is deleted? Should they be deleted as well? Should they be orphaned? Should they be reassigned to another user?

- <span id="utc_time" /> **Make sure time is always UTC**

  Always store time in UTC. When fetching the timestamp for comparison, it will be fetched in the server's timezone, not UTC, so be aware of that.

- <span id="same_data_type" /> **Make sure dates and times are always of the same data type**

  Pick the most suitable type and use it everywhere. And by everywhere, I mean everywhere. I personally prefer using `timestamp` for everything.

- <span id="premature_indexes" /> **Do not prematurely create indexes (unless absolutely necessary)**

  Due to performance, storage, and complexity overheads, adding an index should always be a deliberate and well-considered decision. I prefer adding an index only when querying by that column becomes a bottleneck.

- <span id="timestamps_vs_status_flags" /> **Use timestamps instead of (or alongside) status flags**

  Imagine a post that can be in 3 states: draft, published, archived. Instead of using a status flag, use timestamps for each state, and an XOR constraint to ensure only one of them is set at a time.

  ```
  id, title, content, draft_at, published_at, archived_at
  ```

  You can also add a status flag for convenience, with the necessary integrity constraints added to it as well.

- <span id="mirror_validations_in_database" /> **Mirror validations in database**

  When using Ecto's `unique_constraint` or `validate_*****` in changesets, ensure that the same constraints exist in the database. For bulk updates or inserts, changeset-level validations aren't applied, so always begin with database-level validations and either handle errors gracefully with `unique_constraint` or duplicate them using `validate_****` inside the application.

- <span id="embedded_schemas" /> **Avoid using Ecto's embedded schemas for data managed by the application (opinionated)**

  Use embedded schemas only when storing raw data whose schema isn’t controlled by your application. For example, a correctly used embedded schema might represent a response from a 3rd party API. Embedded schemas are not typed by Postgres, and default values are stored in a schema struct, so when filtering on the DB level, they're not applied. This can lead to hard-to-catch inconsistencies, so it’s better to omit embeds when possible.

  Additionally, it's hard to create constraints on a JSONB column, so if possible, don't use it.

## When creating a new table:

- <span id="soft_deletes" /> **Consider using soft deletes by adding `deleted_at` column**

  Use soft deletes (marking records as deleted without actually removing them) for audit trails and to maintain data consistency.

## When updating an existing table:

So here's the hardest part: you already have an inconsistent table. What should you do? The good news is you've accepted the inconsistency and want to fight it — that's the first step! The rest is easier:

1. Identify all inconsistent records
2. Find out why they're inconsistent and **FIX** all places (either code or processes) that create them
3. Fix the inconsistency by backfilling the data
4. **Add a constraint that would prevent this inconsistency from happening again**. (See the previous section for ideas)
