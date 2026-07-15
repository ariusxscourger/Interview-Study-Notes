# DATABASES - SQL FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: Relational Databases & SQL

### WHY? (Problem Statement)

**Before Relational Databases:**
- Hierarchical/Network databases: Rigid structure, hard to query
- File systems: No ACID, no concurrent access, no integrity
- No standard query language

**What SQL Databases Solve:**
| Problem | Solution |
|---------|----------|
| Data consistency | ACID transactions |
| Flexible queries | Declarative SQL |
| Relationships | Foreign keys, joins |
| Concurrency | Locking, MVCC |
| Durability | WAL, replication |
| Standardization | ANSI SQL |

**Real-World Analogy:**
- Spreadsheet = Single table
- SQL Database = Multiple linked spreadsheets with rules (constraints) and a powerful query language

---

### HOW? (Internal Mechanisms)

#### 1. **Storage Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                      DATABASE CLUSTER                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Primary   │◄───│  Replica 1  │    │  Replica 2  │     │
│  │  (Writer)   │    │  (Reader)   │    │  (Reader)   │     │
│  └──────┬──────┘    └─────────────┘    └─────────────┘     │
│         │                                                      │
│         ▼                                                      │
│  ┌─────────────────────────────────────────┐                 │
│  │           STORAGE ENGINE                │                 │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │                 │
│  │  │ Buffer  │  │  WAL    │  │  Data   │  │                 │
│  │  │ Pool    │──│ (Write  │──│  Files  │  │                 │
│  │  │ (RAM)   │  │ Ahead   │  │ (Disk)  │  │                 │
│  │  └─────────┘  │ Log)    │  └─────────┘  │                 │
│  │               └─────────┘               │                 │
│  └─────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **MVCC (Multi-Version Concurrency Control)**

```
TRANSACTION 1 (Read)          TRANSACTION 2 (Write)
─────────────────             ─────────────────
BEGIN                         BEGIN
SELECT * FROM users           UPDATE users SET name='Bob'
WHERE id=1;                   WHERE id=1;
  │                             │
  ▼                             ▼
Sees version 1              Creates version 2
(consistent snapshot)       (invisible to Tx1)
                              COMMIT
COMMIT
  │                             │
  ▼                             ▼
Sees version 1              Version 2 visible
```

**Isolation Levels:**
| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* |
| Serializable | Prevented | Prevented | Prevented |

*MySQL Repeatable Read prevents phantoms via gap locks

---

### WHAT? (SQL Mastery)

#### 1. **DDL (Data Definition Language)**

```sql
-- Create Table with Constraints
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user' 
        CHECK (role IN ('admin', 'user', 'guest')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ  -- Soft delete
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active_role ON users(is_active, role) 
    WHERE is_active = true;  -- Partial index
CREATE UNIQUE INDEX idx_users_email_lower ON users(LOWER(email));

-- Foreign Key
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Composite Index for Query Patterns
CREATE INDEX idx_posts_user_published ON posts(user_id, published_at DESC)
    WHERE published_at IS NOT NULL;
```

#### 2. **DML (Data Manipulation Language)**

```sql
-- Insert with Returning
INSERT INTO users (email, password_hash, name, role)
VALUES ('alice@example.com', '$2b$12$...', 'Alice', 'admin')
RETURNING id, email, name, created_at;

-- Bulk Insert
INSERT INTO posts (user_id, title, content)
SELECT id, 'Welcome ' || name, 'Hello ' || name || '!'
FROM users WHERE role = 'user'
ON CONFLICT DO NOTHING;

-- Update with Join
UPDATE posts p
SET view_count = p.view_count + 1
FROM users u
WHERE p.user_id = u.id AND u.is_active = true AND p.id = $1
RETURNING p.*;

-- Delete (Soft)
UPDATE users SET deleted_at = NOW(), is_active = false
WHERE id = $1;

-- Hard Delete with Cascade
DELETE FROM users WHERE id = $1; -- CASCADE to posts
```

#### 3. **DQL (Data Query Language) - Advanced**

```sql
-- Window Functions
SELECT 
    u.name,
    p.title,
    ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY p.created_at DESC) as rn,
    LAG(p.title) OVER (PARTITION BY u.id ORDER BY p.created_at) as prev_post,
    COUNT(*) OVER (PARTITION BY u.id) as total_posts
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE p.published_at IS NOT NULL
QUALIFY ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY p.created_at DESC) <= 3;

-- CTEs (Common Table Expressions)
WITH RECURSIVE category_tree AS (
    -- Anchor
    SELECT id, name, parent_id, 1 as level, name::text as path
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    -- Recursive
    SELECT c.id, c.name, c.parent_id, ct.level + 1, 
           ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.level < 10
)
SELECT * FROM category_tree ORDER BY path;

-- Lateral Join (Cross Apply)
SELECT u.name, recent_posts.title
FROM users u
JOIN LATERAL (
    SELECT title FROM posts p 
    WHERE p.user_id = u.id AND p.published_at IS NOT NULL
    ORDER BY p.published_at DESC LIMIT 3
) recent_posts ON true
WHERE u.role = 'author';

-- JSON Operations (PostgreSQL)
SELECT 
    id,
    data->>'name' as name,
    data->'address'->>'city' as city,
    jsonb_agg(orders) as user_orders
FROM users u
JOIN LATERAL (
    SELECT jsonb_build_object(
        'id', o.id, 'total', o.total, 'items', 
        (SELECT jsonb_agg(jsonb_build_object('product', i.product, 'qty', i.qty))
         FROM order_items i WHERE i.order_id = o.id)
    ) as orders
    FROM orders o WHERE o.user_id = u.id
) orders ON true
WHERE data @> '{"premium": true}'::jsonb
GROUP BY u.id;

-- Full Text Search
SELECT title, ts_rank_cd(to_tsvector('english', content), query) as rank
FROM posts, plainto_tsquery('english', 'database optimization') query
WHERE to_tsvector('english', content) @@ query
ORDER BY rank DESC LIMIT 10;
```

#### 4. **Advanced Analytics**

```sql
-- Pivot / Crosstab
SELECT * FROM crosstab(
    'SELECT user_id, date_trunc(''month'', created_at) as month, COUNT(*)
     FROM posts WHERE created_at > NOW() - INTERVAL ''1 year''
     GROUP BY user_id, month ORDER BY 1,2',
    'SELECT generate_series(date_trunc(''month'', NOW() - INTERVAL ''1 year''),
                            date_trunc(''month'', NOW()), ''1 month'')'
) AS ct(user_id UUID, jan INT, feb INT, mar INT, ...);

-- Time-Series Aggregation
SELECT 
    time_bucket('1 hour', created_at) as bucket,
    COUNT(*) as posts_per_hour,
    AVG(LENGTH(content)) as avg_length
FROM posts
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY bucket ORDER BY bucket;

-- Gap Detection
SELECT 
    id,
    created_at,
    LAG(created_at) OVER (ORDER BY created_at) as prev_created,
    created_at - LAG(created_at) OVER (ORDER BY created_at) as gap
FROM events
WHERE created_at > NOW() - INTERVAL '1 day'
HAVING created_at - LAG(created_at) OVER (ORDER BY created_at) > INTERVAL '1 hour';
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Parameterized Queries** | String concatenation | SQL Injection |
| **Proper Indexing** | No indexes / Too many indexes | Slow queries / Slow writes |
| **Connection Pooling** | New connection per request | Exhaustion, latency |
| **Migrations** | Direct schema changes | No rollback, no history |
| **Soft Deletes** | Hard deletes | Data loss, audit trail |
| **UUIDs for PK** | Auto-increment in distributed | Collision, enumeration |
| **Read Replicas** | All queries to primary | Bottleneck |
| **Prepared Statements** | Ad-hoc queries | Plan cache misses |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Explain ACID with examples"

```
ATOMICITY: All or nothing
- Transfer $100: Debit A AND Credit B, or neither
- If crash after debit: Rollback on recovery

CONSISTENCY: Valid state to valid state
- Foreign keys, checks, triggers enforced
- Balance never negative (constraint)

ISOLATION: Concurrent transactions don't interfere
- MVCC: Readers don't block writers, writers don't block readers
- Serializable: Transactions appear sequential

DURABILITY: Committed = permanent
- WAL (Write-Ahead Log): Log before data pages
- fsync on commit (or group commit)
- Replication for disaster recovery
```

#### 2. **Code**: "Fix the N+1 Query Problem"

```sql
-- PROBLEM: N+1 (ORM typical)
-- 1 query: SELECT * FROM users LIMIT 10
-- N queries: SELECT * FROM posts WHERE user_id = ?

-- SOLUTION 1: JOIN (if small result)
SELECT u.*, p.* FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.id IN (1,2,3...);

-- SOLUTION 2: Batch Load (DataLoader pattern)
-- 1. SELECT * FROM users LIMIT 10
-- 2. SELECT * FROM posts WHERE user_id IN (1,2,3...)
-- 3. Group in memory by user_id

-- SOLUTION 3: Lateral Join (PostgreSQL)
SELECT u.*, recent_posts.*
FROM users u
JOIN LATERAL (
    SELECT * FROM posts p 
    WHERE p.user_id = u.id 
    ORDER BY p.created_at DESC LIMIT 5
) recent_posts ON true
WHERE u.role = 'author';

-- SOLUTION 4: Subquery with Array Aggregation
SELECT u.*, 
    COALESCE(
        (SELECT jsonb_agg(jsonb_build_object('id', p.id, 'title', p.title))
         FROM posts p WHERE p.user_id = u.id
         ORDER BY p.created_at DESC LIMIT 5),
        '[]'::jsonb
    ) as recent_posts
FROM users u WHERE u.role = 'author';
```

#### 3. **Design**: "Model a Many-to-Many with Attributes"

```sql
-- Users <-> Projects (with role)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    owner_id UUID REFERENCES users(id)
);

-- Junction table with extra attributes
CREATE TABLE project_members (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member' 
        CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    invited_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, project_id)
);

-- Query: User's projects with role
SELECT p.*, pm.role, pm.joined_at
FROM projects p
JOIN project_members pm ON p.id = pm.project_id
WHERE pm.user_id = $1;

-- Query: Project members with user info
SELECT u.id, u.name, u.email, pm.role, pm.joined_at
FROM users u
JOIN project_members pm ON u.id = pm.user_id
WHERE pm.project_id = $1
ORDER BY 
    CASE pm.role WHEN 'owner' THEN 1 WHEN 'admin' THEN 2 ELSE 3 END,
    pm.joined_at;
```

#### 4. **Performance**: "Query Plan Analysis"

```sql
-- EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT u.name, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id AND p.published_at IS NOT NULL
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(p.id) > 5
ORDER BY post_count DESC LIMIT 20;

-- KEY THINGS TO LOOK FOR:
-- 1. Seq Scan vs Index Scan
--    Seq Scan on large table = BAD (unless small/majority)
-- 2. Index Condition vs Filter
--    Index Cond: used during scan
--    Filter: applied after (rows removed)
-- 3. Rows Estimated vs Actual
--    Large difference = stale statistics (ANALYZE)
-- 4. Buffers: shared hit/read
--    hit = memory, read = disk
-- 5. Join Type
--    Nested Loop: small outer, indexed inner
--    Hash Join: large, no index
--    Merge Join: both sorted
-- 6. Sort Method
--    external merge (disk) vs quicksort (memory)
--    work_mem too small?

-- FIXES:
-- CREATE INDEX ON users(created_at) WHERE created_at > '2024-01-01';
-- CREATE INDEX ON posts(user_id, published_at) WHERE published_at IS NOT NULL;
-- SET work_mem = '256MB'; (per query)
```

#### 5. **Code**: "Implement Optimistic Locking"

```sql
-- Add version column
ALTER TABLE users ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Update with version check
UPDATE users 
SET name = $1, email = $2, version = version + 1, updated_at = NOW()
WHERE id = $3 AND version = $4
RETURNING version;

-- Application logic:
-- 1. Read user with version
-- 2. Modify
-- 3. UPDATE ... WHERE id = ? AND version = ?
-- 4. If 0 rows affected → ConflictError (retry or notify user)
```

#### 6. **Architecture**: "Read Replica Lag Handling"

```sql
-- Primary: Write
INSERT INTO users (...) VALUES (...);

-- Replica: Read (might be stale)
-- Strategy 1: Read from primary for own writes
SELECT * FROM users WHERE id = $1;  -- Route to primary if user just wrote

-- Strategy 2: Synchronous commit for critical reads
-- ALTER SYSTEM SET synchronous_commit = remote_apply;

-- Strategy 3: Application-level retry
async function getUserWithConsistency(userId) {
    for (let i = 0; i < 3; i++) {
        const user = await replica.query('SELECT * FROM users WHERE id = $1', [userId]);
        if (user.updated_at <= await getPrimaryMaxLSN()) return user;
        await sleep(50 * (i + 1));
    }
    return primary.query('SELECT * FROM users WHERE id = $1', [userId]);
}
```

#### 7. **Security**: "SQL Injection Prevention"

```sql
-- VULNERABLE
-- query = "SELECT * FROM users WHERE email = '" + email + "'";

-- SAFE: Parameterized (Prepared Statements)
PREPARE get_user(text) AS 
SELECT * FROM users WHERE email = $1;
EXECUTE get_user('alice@example.com');

-- ORM (Parameterized automatically)
const user = await db.users.findUnique({ where: { email } });

-- Dynamic SQL (when needed) - Use format() with %L/%I
CREATE OR REPLACE FUNCTION search_users(search_term text, role_filter text)
RETURNS SETOF users LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT * FROM users WHERE %s AND role = %L',
        CASE 
            WHEN search_term ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
            THEN 'email ILIKE %L'
            ELSE 'name ILIKE %L'
        END,
        search_term,
        role_filter
    );
END;
$$;
```

---

### GOTCHAS & EDGE CASES

- [ ] **COUNT(*) vs COUNT(col)**: COUNT(*) counts rows, COUNT(col) excludes NULLs
- [ ] **NULL in NOT IN**: `WHERE x NOT IN (1, 2, NULL)` returns empty! Use NOT EXISTS
- [ ] **BETWEEN inclusive**: `BETWEEN 1 AND 10` includes both ends
- [ ] **Timestamp vs Timestamptz**: Always use TIMESTAMPTZ (stores UTC)
- [ ] **CHAR vs VARCHAR**: CHAR pads with spaces, wastes space
- [ ] **Float for money**: Never! Use DECIMAL/NUMERIC(19,4) or integer cents
- [ ] **LIMIT without ORDER BY**: Non-deterministic results
- [ ] **DELETE without WHERE**: Truncates table (use TRUNCATE for speed)
- [ ] **Transaction too long**: Holds locks, blocks vacuum, increases WAL
- [ ] **Vacuum/Autovacuum**: Essential for MVCC cleanup, index stats

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete User Repository (TypeScript/PostgreSQL)**
```typescript
class UserRepository {
  constructor(private pool: Pool) {}

  async create(data: CreateUserDTO): Promise<User> {
    const result = await this.pool.query(
      `INSERT INTO users (email, password_hash, name, role)
       VALUES ($1, $2, $3, $4)
       RETURNING id, email, name, role, created_at`,
      [data.email, data.passwordHash, data.name, data.role || 'user']
    );
    return result.rows[0];
  }

  async findByEmail(email: string): Promise<User | null> {
    const result = await this.pool.query(
      'SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL',
      [email.toLowerCase()]
    );
    return result.rows[0] || null;
  }

  async findById(id: string): Promise<User | null> {
    const result = await this.pool.query(
      'SELECT id, email, name, role, created_at FROM users WHERE id = $1 AND deleted_at IS NULL',
      [id]
    );
    return result.rows[0] || null;
  }

  async update(id: string, data: Partial<User>, expectedVersion: number): Promise<User> {
    const result = await this.pool.query(
      `UPDATE users 
       SET name = COALESCE($1, name),
           email = COALESCE($2, email),
           role = COALESCE($3, role),
           version = version + 1,
           updated_at = NOW()
       WHERE id = $4 AND version = $5 AND deleted_at IS NULL
       RETURNING id, email, name, role, version, updated_at`,
      [data.name, data.email?.toLowerCase(), data.role, id, expectedVersion]
    );
    if (result.rowCount === 0) throw new OptimisticLockError();
    return result.rows[0];
  }

  async softDelete(id: string): Promise<void> {
    await this.pool.query(
      'UPDATE users SET deleted_at = NOW(), is_active = false WHERE id = $1',
      [id]
    );
  }

  async list(filters: UserFilters): Promise<Paginated<User>> {
    const conditions = ['deleted_at IS NULL'];
    const params: any[] = [];
    let paramIndex = 1;

    if (filters.role) {
      conditions.push(`role = $${paramIndex++}`);
      params.push(filters.role);
    }
    if (filters.isActive !== undefined) {
      conditions.push(`is_active = $${paramIndex++}`);
      params.push(filters.isActive);
    }

    const whereClause = conditions.join(' AND ');
    
    // Count
    const countResult = await this.pool.query(
      `SELECT COUNT(*) FROM users WHERE ${whereClause}`, params
    );
    const total = parseInt(countResult.rows[0].count);

    // Data
    params.push(filters.limit, filters.offset);
    const dataResult = await this.pool.query(
      `SELECT id, email, name, role, created_at 
       FROM users WHERE ${whereClause}
       ORDER BY created_at DESC
       LIMIT $${paramIndex++} OFFSET $${paramIndex}`,
      params
    );

    return { data: dataResult.rows, total, page: filters.page, limit: filters.limit };
  }
}
```

#### 2. **Migration Best Practices**
```sql
-- 001_create_users.up.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user',
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_users_email ON users(LOWER(email));

-- 002_add_soft_delete.up.sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;

-- 003_add_profile_json.up.sql (Backward compatible)
ALTER TABLE users ADD COLUMN profile JSONB NOT NULL DEFAULT '{}';
-- No DEFAULT needed for JSONB, empty object is fine

-- DOWNS (if needed for rollback)
-- 003_add_profile_json.down.sql
ALTER TABLE users DROP COLUMN profile;

-- SAFE MIGRATION RULES:
-- 1. Never drop column/table in same deploy as code using it
-- 2. Add columns with DEFAULT (or nullable)
-- 3. Backfill in batches for large tables
-- 4. Create index CONCURRENTLY (PostgreSQL)
-- 5. Use advisory locks for coordination
```

#### 3. **Connection Pool Configuration**
```typescript
// Node.js pg pool
const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,                    // Max connections
  min: 2,                     // Min idle
  idleTimeoutMillis: 30000,   // Close idle after 30s
  connectionTimeoutMillis: 2000,
  statement_timeout: 30000,   // Query timeout
  query_timeout: 30000,
  application_name: 'nanolite-api',
});

// Monitor
pool.on('error', (err) => console.error('Unexpected pool error', err));

// Health check
async function checkDb() {
  const client = await pool.connect();
  try {
    await client.query('SELECT 1');
    return true;
  } finally {
    client.release();
  }
}
```

---

### PRACTICE PROBLEMS

1. **Design** a schema for: E-commerce (products, categories, orders, payments, inventory)
2. **Optimize** a slow query: 10-table join with subqueries, 500ms → <50ms
3. **Implement** advisory locking**: Implement** advisory locks for distributed mutex
4. **Write** a migration to split a JSONB column into normalized tables
5. **Debug** deadlock: Two transactions updating rows in different order

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| ACID Properties | Atomicity, Consistency, Isolation, Durability |
| MVCC | Multi-Version Concurrency Control - readers don't block writers |
| Isolation Levels | Read Uncommitted < Read Committed < Repeatable Read < Serializable |
| N+1 Problem | One query + N queries for relations. Fix: JOIN, batch, lateral |
| Optimistic Locking | Version column, UPDATE ... WHERE version = X |
| Pessimistic Locking | SELECT FOR UPDATE / FOR SHARE |
| Soft Delete | deleted_at timestamp + WHERE deleted_at IS NULL |
| Index Types | B-tree (default), Hash, GiST, GIN, BRIN |
| Partial Index | WHERE clause on index (smaller, faster) |
| Connection Pool | Reuse connections, limit max, timeout idle |
| Prepared Statement | Parse/plan once, execute many (security + perf) |
| WAL | Write-Ahead Log - durability, replication, crash recovery |

---

## NEXT TOPIC: `03-DATABASES-NOSQL.md`

> **Study Tip**: Run `EXPLAIN (ANALYZE, BUFFERS)` on your 5 slowest queries. Add indexes, rewrite queries, measure improvement.