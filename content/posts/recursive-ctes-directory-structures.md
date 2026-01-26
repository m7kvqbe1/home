+++
title = 'Recursive SQL CTEs for Directory Structures'
date = 2026-01-26T12:00:00+00:00
draft = false
+++

Recently, I worked on efficiently querying nested directory structures stored in a database. This led me to explore recursive Common Table Expressions (CTEs) and how they can elegantly solve hierarchical data problems.

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

## The Key Insight: Filter-Aware Directory Listing

The interesting challenge was creating a function that only shows directories containing files matching active filters. If you're filtering by file extension `.pdf`, you don't want to show empty directories or directories containing only `.jpg` files.

Here's the approach:

```sql
CREATE OR REPLACE FUNCTION browse_directory(
  p_parent_id INT,
  p_recursive BOOLEAN DEFAULT FALSE,
  p_extension TEXT DEFAULT NULL,
  p_min_size BIGINT DEFAULT NULL,
  p_start_date TIMESTAMPTZ DEFAULT NULL
)
RETURNS TABLE (
  item_type TEXT,
  item_id TEXT,
  name TEXT,
  size BIGINT,
  extension TEXT,
  created_at TIMESTAMPTZ,
  sort_order INT
) AS $$
DECLARE
  has_filters BOOLEAN;
BEGIN
  -- Determine if any filters are active
  has_filters := (
    p_extension IS NOT NULL OR
    p_min_size IS NOT NULL OR
    p_start_date IS NOT NULL
  );

  RETURN QUERY
  -- CTE: Get all files matching the filter criteria
  WITH filtered_files AS (
    SELECT
      f.id,
      f.name,
      f.size,
      f.extension,
      f.created_at,
      f.directory_id
    FROM files f
    LEFT JOIN directories d ON f.directory_id = d.id
    WHERE
      -- RECURSIVE MODE: Include files from all subdirectories
      (NOT p_recursive OR p_parent_id IS NULL OR
       f.directory_id = p_parent_id OR
       d.path LIKE (SELECT path || '/%' FROM directories WHERE id = p_parent_id))

      -- Apply filters
      AND (p_extension IS NULL OR f.extension = p_extension)
      AND (p_min_size IS NULL OR f.size >= p_min_size)
      AND (p_start_date IS NULL OR f.created_at >= p_start_date)
  )

  -- Return directories at current level
  SELECT
    'directory'::TEXT,
    d.id::TEXT,
    d.name,
    d.total_size,
    NULL::TEXT,
    d.created_at,
    0  -- Sort directories first
  FROM directories d
  WHERE NOT p_recursive
    AND d.parent_id IS NOT DISTINCT FROM p_parent_id
    AND (
      NOT has_filters  -- No filters? Show all directories
      OR EXISTS (
        -- Only show if contains matching files
        SELECT 1 FROM filtered_files WHERE directory_id = d.id
      )
    )

  UNION ALL

  -- Return files at current level
  SELECT
    'file'::TEXT,
    f.id,
    f.name,
    f.size,
    f.extension,
    f.created_at,
    1  -- Sort files after directories
  FROM filtered_files f
  WHERE NOT p_recursive
    AND f.directory_id IS NOT DISTINCT FROM p_parent_id

  UNION ALL

  -- RECURSIVE: Return ALL descendant files
  SELECT
    'file'::TEXT,
    f.id,
    f.name,
    f.size,
    f.extension,
    f.created_at,
    1
  FROM filtered_files f
  WHERE p_recursive;
END;
$$ LANGUAGE plpgsql STABLE;
```

## Key Design Decisions

### 1. Reusable CTE for Filtered Files

The `filtered_files` CTE is defined once and reused three times in the UNION query. This eliminates duplication and ensures consistent filtering logic across all query branches.

### 2. Path-Based Recursive Filtering

Instead of using a recursive CTE to find all subdirectories, I leveraged the `path` field:

```sql
d.path LIKE (SELECT path || '/%' FROM directories WHERE id = p_parent_id)
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

The combination of these techniques created a flexible function that handles both browsing and filtered searching efficiently.
