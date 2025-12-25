---
title: "Tracing FK delete dependencies in PostgreSQL"
date: 2025-03-28T00:00:00Z
description: PostgreSQL query that recursively traces foreign key (FK) relationships to find all tables that indirectly or directly depend on a specific table and displays their ON DELETE actions.
tags:
  - postgresql
  - foreign keys
  - sql
categories:
  - Postgres
draft: false
---

### TL;DR

```sql
WITH RECURSIVE fk_chain AS (SELECT c.oid         AS constraint_oid,
                                   c.conname     AS constraint_name,
                                   c.conrelid    AS table_oid,
                                   c.confrelid   AS referenced_table_oid,
                                   c.confdeltype,
                                   ARRAY [c.oid] AS path -- prevent cycles
                            FROM pg_constraint c
                            -- THIS IS A PLACE WHERE YOU ADJUST THE TABLE
                            WHERE c.confrelid = 'TODO_TABLE_NAME'::regclass
                              AND c.contype = 'f'
                              -- THIS IS A PLACE WHERE YOU ADJUST THE DESIRED RELATIONS
                              and c.confdeltype in ('a', 'r', 'n')

                            UNION ALL

                            SELECT child_c.oid,
                                   child_c.conname,
                                   child_c.conrelid,
                                   child_c.confrelid,
                                   child_c.confdeltype,
                                   path || child_c.oid
                            FROM pg_constraint child_c
                                     JOIN fk_chain parent ON child_c.confrelid = parent.table_oid
                            WHERE child_c.contype = 'f'
                              AND NOT child_c.oid = ANY (path)
                              and child_c.confdeltype in ('a', 'r', 'n'))
SELECT child_table.relname      AS table_name,
       referenced_table.relname AS parent_table,
       CASE confdeltype
           WHEN 'a' THEN 'NO ACTION'
           WHEN 'r' THEN 'RESTRICT'
           WHEN 'c' THEN 'CASCADE'
           WHEN 'n' THEN 'SET NULL'
           WHEN 'd' THEN 'SET DEFAULT'
           ELSE 'UNKNOWN'
           END                  AS on_delete_action
FROM fk_chain c
         JOIN pg_class child_table ON child_table.oid = c.table_oid
         JOIN pg_class referenced_table ON referenced_table.oid = c.referenced_table_oid;
```

### Use Cases

- Understanding cascade risk before deleting data.
- Visualizing dependency trees for cleanup/migrations.
- Preventing unintended orphaned rows.

### What It Does

- Starts with all FKs pointing to the desired table (`confrelid = 'TODO_TABLE_NAME'::regclass`).
- Follows the chain of FKs recursively, capturing tables that depend on tables that depend on tables, and so on.
- Prevents cycles using a path array of visited constraints.
- Filters to only include certain delete actions: 'a' (NO ACTION), 'r' (RESTRICT), 'n' (SET NULL).
- Outputs each dependent table, its parent table, and the delete behavior.

### Output Columns

- `table_name`: The dependent (child) table.
- `parent_table`: The referenced (parent) table.
- `on_delete_action`: Human-readable delete rule.

### Customization

- Replace `'TODO_TABLE_NAME'::regclass` with any target table.
- Add 'c' (CASCADE), 'd' (SET DEFAULT) to include other delete actions.
