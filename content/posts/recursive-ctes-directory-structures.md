+++
title = 'Recursive CTEs for Directory Structures'
date = 2026-01-26T09:00:00+00:00
draft = false
+++

Recently, I worked on efficiently querying nested directory structures stored in a database. This led me to explore recursive Common Table Expressions (CTEs) and how they can solve hierarchical data problems.

## The Problem

When modeling a file system in a relational database, you have a classic hierarchical structure: directories contain files and other directories. This creates a parent-child relationship that can be arbitrarily deep.

The challenge becomes more interesting when you need to filter this structure. For example, showing only directories that contain files matching certain criteria (file size, creation date, etc.) while maintaining the hierarchical relationship.

## Database Schema

Here's a simplified schema representing directories and files:

```sql
CREATE TABLE directories (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id INT REFERENCES directories(id),
  path TEXT UNIQUE NOT NULL,
  total_files INT DEFAULT 0,
  total_size BIGINT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE files (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  directory_id INT REFERENCES directories(id),
  size BIGINT,
  extension TEXT,
  created_at TIMESTAMPTZ,
  metadata JSONB
);

CREATE INDEX idx_files_directory ON files(directory_id);
CREATE INDEX idx_directories_parent ON directories(parent_id);
```

The `parent_id` in the `directories` table creates the tree structure. The `path` field stores the full hierarchical path, which proves useful for recursive queries.

## Understanding Recursive CTEs

A recursive CTE is a query that references itself. It consists of two parts:

1. **Base case**: The initial query that starts the recursion
2. **Recursive case**: The query that references the CTE itself

Here's a simple example that finds all subdirectories of a given directory:

```sql
WITH RECURSIVE subdirectories AS (
  -- Base case: start with the target directory
  SELECT id, name, parent_id, path, 0 AS depth
  FROM directories
  WHERE id = 123

  UNION ALL

  -- Recursive case: find children of previous results
  SELECT d.id, d.name, d.parent_id, d.path, sd.depth + 1
  FROM directories d
  INNER JOIN subdirectories sd ON d.parent_id = sd.id
)
SELECT * FROM subdirectories;
```

This query starts with directory 123 and recursively finds all its descendants, tracking the depth as it goes.

## Filter-Aware Directory Listing

The interesting challenge was a listing that only shows directories containing files matching active filters. If you're filtering by file extension `.pdf`, you don't want to show empty directories or directories containing only `.jpg` files.

The shape is a single query with three parts. First, a CTE that collects the files matching the active filters, walking into subdirectories by matching on the stored path:

```sql
WITH filtered_files AS (
  SELECT f.id, f.name, f.size, f.directory_id
  FROM files f
  JOIN directories d ON d.id = f.directory_id
  WHERE d.path LIKE (SELECT path || '/%' FROM directories WHERE id = :parent_id)
    AND (:extension IS NULL OR f.extension = :extension)
    AND (:min_size IS NULL OR f.size >= :min_size)
)
```

Then the directories at the current level, kept only if a matching file exists somewhere beneath them:

```sql
SELECT 'directory' AS item_type, d.id, d.name, 0 AS sort_order
FROM directories d
WHERE d.parent_id IS NOT DISTINCT FROM :parent_id
  AND (NOT :has_filters OR EXISTS (
    SELECT 1 FROM filtered_files f WHERE f.directory_id = d.id
  ))
```

And finally the matching files at the current level, unioned on after the directories:

```sql
UNION ALL
SELECT 'file', f.id, f.name, 1
FROM filtered_files f
WHERE f.directory_id IS NOT DISTINCT FROM :parent_id
```

## Key Design Decisions

### 1. Reusable CTE for Filtered Files

The `filtered_files` CTE is defined once and reused by both branches of the union. This eliminates duplication and ensures consistent filtering logic across all query branches.

### 2. Path-Based Recursive Filtering

Instead of using a recursive CTE to find all subdirectories, I used the `path` field:

```sql
d.path LIKE (SELECT path || '/%' FROM directories WHERE id = :parent_id)
```

This is more efficient than recursion for the common case of filtering files in a subtree.

### 3. Conditional Directory Filtering

The `EXISTS` subquery is only executed when filters are active:

```sql
AND (
  NOT has_filters
  OR EXISTS (SELECT 1 FROM filtered_files WHERE directory_id = d.id)
)
```

This means browsing without filters (the common case) remains fast, while filtered browsing only shows relevant directories.

### 4. Sort Order via Union

By assigning `sort_order=0` to directories and `sort_order=1` to files, the client can order results to show directories first without additional sorting logic.

## Performance Optimization

A composite index on the files table significantly improves the filtered queries:

```sql
CREATE INDEX idx_files_composite
ON files(directory_id, extension, size, created_at)
WHERE created_at IS NOT NULL;
```

The column order matters: `directory_id` is most selective for the common case of browsing a single directory, followed by the filter columns.

The partial index (`WHERE created_at IS NOT NULL`) reduces index size by excluding incomplete records.

## When to Use Recursive CTEs

Recursive CTEs are powerful for:

- Traversing hierarchical data (org charts, category trees, etc.)
- Bill of materials (parts containing other parts)
- Graph traversal with cycle detection
- Computing aggregates across a tree

However, they're not always necessary. In this case, storing the full path allowed for efficient filtering without full recursion.

## Conclusion

Modeling directory structures in a database revealed some interesting patterns:

1. Recursive CTEs aren't always needed for hierarchical queries
2. Storing computed paths can enable efficient filtering
3. CTEs can eliminate code duplication in complex queries
4. Conditional logic (like `has_filters`) can optimize common vs. filtered cases
5. Composite indexes should match your query patterns

The combination of these techniques gives a single query that handles both browsing and filtered searching efficiently.
