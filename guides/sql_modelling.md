# SQL Relationship Modeling

- Disclaimer: It is ChatGPT generated, though I've read it and it has correct data from what I know of.
- Website to model: https://dbdiagram.io/

## 1. Identify the Relationship Type

1. **One-to-Many**

   - Example: An **Organization** can have many **Employees**, but each **Employee** belongs to exactly one **Organization**.

   - Modeling rule: Place the foreign key (FK) in the “many” table pointing back to the “one.”

   - In dbdiagram.io, that means in `Table org_employee`, you write:

   ```sql
   organization_id uuid [not null, ref: > organization.id]
   ```

   - You **do not** put any `employee_ids` array or list in the `organization` table.

2. **One-to-One**

   - Example: A **Company** has exactly one **HeadquarterAddress** (and vice versa).

   - Modeling rule: You still place the FK in one of the two tables, but you also add a `unique` constraint on that FK column.
   - **Either**:

   ```sql
   Table company {
     id uuid [pk]
     headquarter_id uuid [not null, unique, ref: > address.id]
   }
   ```

   - **or**:
     you could put `company_id uuid [not null, unique, ref: > company.id]` inside `Table address`. Either way is fine—just ensure the FK is marked `unique` so that each row on both sides pairs to exactly one row on the other.

3. **Many-to-Many**

   - Example: A **Company** can have many **ExternalAuditors**, and a given **ExternalAuditor** might audit multiple **Companies**.

   - Modeling rule: You _never_ put an “array of IDs” inside either main table. Instead, you create a **join (associative) table** that has two FKs—one to each parent table—and a composite PK (or a unique constraint) on those two columns.

   - In dbdiagram.io:

   ```sql
   Table company {
     id uuid [pk]
   }

   Table external_auditor {
     id uuid [pk]
   }

   Table company_external_auditor {
     company_id uuid [not null, ref: > company.id]
     auditor_id uuid [not null, ref: > external_auditor.id]

     Indexes {
       (company_id, auditor_id) [pk]
     }
   }
   ```

   - Now each row in `company_external_auditor` links one company ↔ one auditor. To find all auditors for a given company, you simply join on that table.

---

## 2. “Where to Put the `ref: > other_table.id`”

- **Always** place the FK in the “child” or “many-side” of a relationship.

  - For One-to-Many: The table that can appear multiple times gets the FK.
  - For One-to-One: Pick one side; that table has a unique‐constrained FK pointing to the other.

- **You should not** try to store an “array” of foreign keys in a single column. That doesn’t enforce referential integrity the way a proper FK does—and normal forms tell us to avoid “repeating groups.”

---

### ✔️ Example: Fixing “Company ↔ ExternalAuditor” case

In `Table company`:

```sql
external_auditors uuid [null, ref: > external_auditor.id]
```

but that can hold only one UUID, and it also creates ambiguity: “Does it mean one auditor or an array?” In fact, it models a **many-to-one** (each company may point to one auditor), not a one-to-many or many-to-many.

**Two correct options** (depending on your true cardinality):

1. **If a Company can have _many_ ExternalAuditors, but each ExternalAuditor audits exactly one Company** (a one-to-many):

   ```sql
   Table company {
     id uuid [pk]
     name varchar [not null, unique]
     organization_id uuid [not null, ref: > organization.id]
     address varchar [not null]
     gstin varchar [not null, ref: > GSTIN.id]
     pan varchar [not null, ref: > GSTIN.pan]
     -- **REMOVE** external_auditors from here
   }

   Table external_auditor {
     id uuid [pk]
     name varchar [not null]
     email varchar [not null, unique]
     phone varchar [not null, unique]
     company_id uuid [null, ref: > company.id]
     organization_id uuid [null, ref: > organization.id]
   }
   ```

   - Now, if you want _all_ auditors for Company X, you simply query `external_auditor WHERE company_id = X.id`.
   - You do **not** need an `external_auditors` column in `company`.

2. **If an ExternalAuditor can work for multiple Companies**, and each Company can hire multiple Auditors (a many-to-many):

   ```sql
   Table company {
     id uuid [pk]
     name varchar [not null, unique]
     organization_id uuid [not null, ref: > organization.id]
     address varchar [not null]
     gstin varchar [not null, ref: > GSTIN.id]
     pan varchar [not null, ref: > GSTIN.pan]
     -- remove external_auditors here too
   }

   Table external_auditor {
     id uuid [pk]
     name varchar [not null]
     email varchar [not null, unique]
     phone varchar [not null, unique]
     organization_id uuid [null, ref: > organization.id]
     -- no company_id here either
   }

   Table company_external_auditor {
     company_id uuid [not null, ref: > company.id]
     auditor_id uuid [not null, ref: > external_auditor.id]
     Indexes {
       (company_id, auditor_id) [pk]
     }
   }
   ```

   - When you want to add a link “Auditor A audits Company B,” you insert a row `(B.id, A.id)` into `company_external_auditor`.
   - To list “all auditors of Company B,” you join `company_external_auditor` → `external_auditor` on `auditor_id`.

---

## 3. General Checklist

1. **Ask yourself**:

   - “Can multiple rows in Table A reference a single row in Table B?”
     • If **yes**, then it’s a **One-to-Many**: put `b_id ref: > table_b.id` in `table_a`.
     - If **no** (each row in A can point to at most one row in B, and each row in B to at most one row in A), it’s **One-to-One**: pick one table to hold a `unique` FK.

2. **Check for Many-to-Many**:

   - “Is it possible that one row in A is linked to multiple rows in B, and vice versa?”
     - If yes, create a **third (join) table** with two FKs—one to A, one to B—and a composite PK or unique index on those two.

3. **Never store “arrays of IDs” in a single column** if you care about proper referential integrity. Always normalize to a separate table or to a FK field in the child table.

4. **Every FK lives in the table that holds the “many side.”**

   - A user-does-not-own many posts? FKs go on `post.user_id`.
   - A book-can-belong-to-one-category? FK goes on `book.category_id` (assuming Category → many Books).

5. **When you’re in doubt**:

   - Draw a quick diagram on paper: label each table and draw lines “one” → “many.” The arrow always points to the child (the “many”) side.
   - Then translate into dbdiagram.io by putting `ref: > parent_table.id` on that child side.

---

## 4. A Few More Mini-Examples (dbdiagram.io DSL)

1. **One-to-Many (Department ↔ Employee)**

   ```sql
   Table department {
     id int [pk]
     name varchar
   }

   Table employee {
     id int [pk]
     name varchar
     department_id int [not null, ref: > department.id]
   }
   ```

2. **One-to-One (User ↔ UserProfile)**

   ```sql
   Table users {
     id int [pk]
     email varchar [not null, unique]
   }

   Table user_profile {
     id int [pk]
     user_id int [not null, unique, ref: > users.id]
     bio text
   }
   ```

   - Here `user_id` is both a FK **and** `unique` so each `user_profile` row pairs to exactly one `users` row, and no two `user_profile` rows can point to the same user.

3. **Many-to-Many (Student ↔ Course)**

   ```sql
   Table student {
     id int [pk]
     name varchar
   }

   Table course {
     id int [pk]
     title varchar
   }

   Table student_course {
     student_id int [not null, ref: > student.id]
     course_id int [not null, ref: > course.id]

     Indexes {
       (student_id, course_id) [pk]
     }
   }
   ```

---

### 🔑 Take-Home Points

- **Always put the FK column in the “child” (or many-side) table.**
- **Mark it `unique` only if it truly is one-to-one.**
- **Use a separate join table for any many-to-many.**
- **dbdiagram.io’s DSL syntax for a foreign key is**

  ```sql
  column_name <type> [ref: > referenced_table.primary_key_column]
  ```

  and you can tack on modifiers like `not null` or `unique`.

Once you practice drawing “one” vs “many” arrows on paper or a whiteboard, translating to dbdiagram.io will become second nature. If you still feel unsure, sketch out:

1. **Who is the parent?**
2. **Who is the child?**
3. **Is there ever a scenario where A can link to multiple B’s, and B to multiple A’s?**

Answer those, then place the FK accordingly. That way, you’ll never accidentally create “arrays of IDs” where a proper normalized relationship belongs.

# Indexing and its benefits

An **index** is a database structure (usually a B-tree in PostgreSQL) that lets the database quickly locate rows matching certain column values, without scanning the entire table. In your join-table example, we wrote:

```sql
Table company_external_auditor {
  company_id uuid       [not null, ref: > company.id]
  auditor_id uuid       [not null, ref: > external_auditor.id]

  Indexes {
    (company_id, auditor_id) [pk]
  }
}
```

Here’s what’s happening:

1. **“Indexes { (company_id, auditor_id) \[pk] }”**

   - By declaring `(company_id, auditor_id)` as a **primary key**, you are implicitly creating a **composite unique index** on those two columns.
   - In PostgreSQL, a primary-key constraint automatically builds a B-tree index behind the scenes.
   - That means whenever you query something like:

     ```sql
     SELECT *
       FROM company_external_auditor
      WHERE company_id = 'some‐uuid'
        AND auditor_id = 'some‐other‐uuid';
     ```

     the database can use that composite index to jump straight to the matching row, instead of scanning every row in `company_external_auditor`.

---

## 1. Why Indexes Matter for Performance

1. **Faster LOOKUPS**

   - Without an index, a query like

     ```sql
     SELECT * FROM company_external_auditor
      WHERE company_id = 'X-UUID';
     ```

     would require a **sequential scan**—PostgreSQL reads every row from top to bottom, checking `company_id` until it finds matches. On large tables, that can be very slow (O(N) time).

   - With an index on `company_id`, PostgreSQL can do a **B-tree search** (O(log N) time) to find all rows where `company_id = 'X-UUID'` almost instantly.

2. **JOINs Become Cheaper**

   - If you frequently join `company_external_auditor` back to `company` via

     ```sql
     … JOIN company
        ON company.id = company_external_auditor.company_id
     ```

     then having `company_id` indexed means PostgreSQL can rapidly find all matching join-rows instead of scanning the entire join table.

3. **ORDER BY and GROUP BY Can Use Indexes**

   - Suppose you do

     ```sql
     SELECT auditor_id
       FROM company_external_auditor
      WHERE company_id = 'X-UUID'
      ORDER BY auditor_id;
     ```

     If `(company_id, auditor_id)` is indexed in that order, PostgreSQL can produce results already sorted by `auditor_id` for that given `company_id`. That avoids a separate sort step.

---

## 2. What Kinds of Indexes Are There?

1. **Primary Key Index (Unique, Clustered by Default in Some Engines)**

   - Whenever you declare a column (or set of columns) as `[pk]`, the database automatically creates a **unique B-tree index**.
   - In dbdiagram’s DSL, writing `(company_id, auditor_id) [pk]` means:

     1. Enforce uniqueness of that pair.
     2. Create a B-tree index on `(company_id, auditor_id)`.

2. **Unique Index**

   - If you write, for example,

     ```sql
     email varchar [not null, unique, ref: > users.email]
     ```

     that creates a unique index on `email`. It both speeds up lookups by `email` and prevents duplicates.

3. **Non-Unique (Regular) Index**

   - You can add an index on a column (or expression) even if it’s not declared `pk` or `unique`. In raw SQL:

     ```sql
     CREATE INDEX idx_company_external_by_company
       ON company_external_auditor(company_id);
     ```

   - In dbdiagram DSL, you’d write:

     ```sql
     Indexes {
       company_id
     }
     ```

     That means “please build a regular B-tree index on `company_id`.”

4. **Partial Indexes, Expression Indexes, GIN/GINdexes, etc.**

   - PostgreSQL supports more advanced index types, but for most OLTP schemas you’ll use ordinary B-tree indexes (the default) on columns you filter or join by.

---

## 3. How to Decide **What to Index** (and What **Not** to)

### A. Always-Indexed Columns

1. **Primary Keys**

   - Every table should have a primary key; by definition, that column is indexed.
   - In dbdiagram DSL, marking `[pk]` automatically creates the index.

2. **Foreign Keys**

   - If you frequently join on a foreign key (e.g., `company_external_auditor.company_id` → `company.id`), you almost always want an index on that FK column.
   - That way, when you run

     ```sql
     SELECT …
       FROM company
     ```

LEFT JOIN company_external_auditor
ON company.id = company_external_auditor.company_id
WHERE company.id = 'some-uuid';
\`\`\`
the database can quickly “probe” the join table.

> **Rule of thumb**: If you have a column `X_id` that references another table, and you’ll ever query “WHERE X_id = …” or join on `X_id`, index it.

### B. Index Columns Used in `WHERE` Clauses

- If you frequently do searches like

  ```sql
  SELECT *
    FROM external_auditor
   WHERE email = 'alice@example.com';
  ```

  you want an index on `email`. If that column is already `unique`, you get it for free.

- Columns used in **range filters** (`<`, `≤`, `BETWEEN`) can also benefit, as long as you construct an appropriate B-tree index.

### C. Index Columns in `ORDER BY` or `GROUP BY`

- If you sort or group by a column, an index can help avoid a separate sort step.
- E.g.,

  ```sql
  SELECT *
    FROM org_employee
   WHERE organization_id = 'X-UUID'
   ORDER BY name;
  ```

  Indexes on `(organization_id, name)` would let the database traverse rows in the right order.

### D. Avoid Over-Indexing

- Every index takes up space on disk. More importantly, **every write (INSERT/UPDATE/DELETE)** now has to update each relevant index.
- If you create an index on a column you never query by, you’ve just slowed down writes without any read-performance gain.

> **Rule of thumb**: Only index columns you actually filter, join, or sort by in real queries. If you’re not sure, monitor your slow queries (e.g., using `EXPLAIN`/`EXPLAIN ANALYZE`) and see which predicates are doing sequential scans.

---

## 4. Back to Your Example

```sql
Table company_external_auditor {
  company_id uuid       [not null, ref: > company.id]
  auditor_id uuid       [not null, ref: > external_auditor.id]

  Indexes {
    (company_id, auditor_id) [pk]
  }
}
```

1. **Why `(company_id, auditor_id) [pk]`?**

   - You want to ensure **no duplicate pairs**—i.e., “Auditor A audits Company B” can only appear once.
   - By designating those two columns as a composite primary key, PostgreSQL will:

     1. Enforce uniqueness of `(company_id, auditor_id)`.
     2. Build a combined B-tree index on `(company_id, auditor_id)` so lookups are fast.

2. **What if you also frequently query all auditors of a given company?**

   - You might run:

     ```sql
     SELECT auditor_id
       FROM company_external_auditor
      WHERE company_id = 'some-uuid';
     ```

   - The composite index on `(company_id, auditor_id)` can serve that query, because it’s ordered first by `company_id`, then by `auditor_id`. PostgreSQL will find the start of the `company_id = 'some-uuid'` partition and walk the index to retrieve all matching `auditor_id` values in sorted order.
   - No need for a separate index on just `company_id` in that scenario—**the left‐most column of the composite index is enough**.

3. **What if you ran queries like “find all companies for a given auditor”?**

   - That is,

     ```sql
     SELECT company_id
       FROM company_external_auditor
      WHERE auditor_id = 'ABC-UUID';
     ```

   - In that case, a composite index `(company_id, auditor_id)` is **not** optimal, because the index is sorted first by `company_id`. To search by `auditor_id`, PostgreSQL can’t do a fast index range scan on `(company_id, auditor_id)`—it would have to scan every `company_id` partition and filter by `auditor_id`.
   - If you expect to do “lookup by `auditor_id`” often, you could add a separate **single-column** index on `auditor_id`:

     ```sql
     Indexes {
       (company_id, auditor_id) [pk]
       auditor_id
     }
     ```

   - That way, when your WHERE clause is `auditor_id = 'ABC-UUID'`, the DB can jump right to the matching rows in the `auditor_id` index.

---

## 5. Visualizing Index Impact

1. **With No Index**

   - Table has 1 million rows.
   - Query:

     ```sql
     SELECT *
       FROM org_employee
      WHERE organization_id = 'X';
     ```

   - Without an index on `organization_id`, PostgreSQL will perform a **sequential scan**: it checks `organization_id` on all 1 million rows before returning matches. If “X” appears in 10 000 rows, that’s still an O(N) operation.

2. **With an Index on `organization_id`**

   - PostgreSQL uses the B-tree index to find the first occurrence of “X” and then follows the index pointers to each of the 10 000 matching row locations on disk.
   - Runtime drops from O(N) to roughly O(log N + M), where N = total rows, M = matching rows. For large N, that’s a huge speedup.

> **Bottom line**: Indexes trade **storage + slightly slower writes** for **much faster reads** on indexed columns.

---

## 6. Quick “Indexing Checklist” for Your Next Schema

- **Every table** should have a primary key (`[pk]`). That automatically gives you exactly one index (on the PK column(s)).
- **Every foreign-key column** that you plan to join on or filter by should be indexed. In DSL, that means after `ref: > other_table.id`, add it under `Indexes` if you expect heavy use. (In practice, you might rely on the database to auto-index FKs, but explicitly listing helps you think it through.)
- **If you frequently filter or sort** on a column (or combination of columns), create an index on exactly those columns, in the same order you use in `WHERE`/`ORDER BY`.
- **Avoid indexing columns that change constantly**, or that have extremely high cardinality but are seldom used in queries. For instance, “last_updated_timestamp” on a busy table may not benefit you if you never search by that column.
- **Watch out for redundant indexes**—if you already have a composite index on `(A, B, C)`, you often don’t need separate indexes on `(A)` or `(A, B)` unless you have queries that filter on `(B)` alone (in which case you might need an index starting with `B`).

---

### Example: Applying Indexing to Your Schema

```sql
Table organization {
  id       uuid [pk, unique]       // PK implies a unique index on id
  name     varchar [not null, unique] // unique → automatically indexed
  address  varchar [not null, unique] // unique → automatically indexed
}

Table company {
  id               uuid  [pk]        // PK → index on id
  name             varchar [not null, unique] // unique → index
  organization_id  uuid  [not null, ref: > organization.id]
  address          varchar [not null]
  gstin            varchar [not null, ref: > GSTIN.id]
  pan              varchar [not null, ref: > GSTIN.pan]
  // We removed external_auditors array—use join table instead
  Indexes {
    organization_id   // index to speed up “WHERE organization_id = …” or JOIN
    gstin
    pan
  }
}

Table external_auditor {
  id               uuid   [pk]      // PK → index on id
  name             varchar [not null]
  email            varchar [not null, unique] // → index
  phone            varchar [not null, unique] // → index
  company_id       uuid   [null, ref: > company.id]
  organization_id  uuid   [null, ref: > organization.id]
  Indexes {
    company_id       // if you often do: “WHERE company_id = …”
    organization_id  // if you need: “WHERE organization_id = …”
  }
}

Table company_external_auditor {
  company_id uuid   [not null, ref: > company.id]
  auditor_id uuid   [not null, ref: > external_auditor.id]

  Indexes {
    (company_id, auditor_id) [pk] // composite PK → index
    // If you also need to query by auditor_id alone:
    // auditor_id
  }
}
```

- **Primary keys** (e.g. `id` on every table) and **unique** columns (`name`, `email`, `phone`, `pan`, etc.) are automatically indexed.
- We explicitly added indexes on `organization_id`, `gstin`, and `pan` in `company` because you might filter or join by them.
- In `external_auditor`, indexing `company_id` speeds up “find all auditors of X company.” If you also ever need “find all companies an auditor works for,” you’d add a single-column index on `auditor_id` inside the join table.

---

## 7. Summing Up

- **Index = B-tree (in PostgreSQL) behind the scenes** that speeds up lookups on one or more columns.
- **Primary key** and **UNIQUE** constraints automatically build indexes—no extra work required.
- For any other column you frequently use in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY`, consider adding a normal (non-unique) index.
- **Don’t overdo it:** each index slightly slows down `INSERT`/`UPDATE`/`DELETE` (because the index must be updated), and consumes extra disk space.
- **In dbdiagram.io**, the `Indexes { … }` block is where you declare which columns (or column-sets) should be indexed. If you mark `(col1, col2) [pk]`, that makes a composite primary key _and_ a composite index automatically.

Once you identify your common query patterns—“Which columns appear in my `WHERE` clauses?” or “Which FKs will I join on?”—you’ll know exactly which indexes to define. That way, your most frequent lookups stay fast, and you avoid unnecessary overhead on less-used fields.
