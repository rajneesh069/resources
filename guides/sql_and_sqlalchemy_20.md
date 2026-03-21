# SQL, Relational Databases, and ORMs in a Nutshell

Structured Query Language (SQL) is the standard language for managing data in **relational databases** (RDBMS), which store information in tables of rows and columns. Relational databases enforce ACID properties (Atomicity, Consistency, Isolation, Durability) to guarantee reliable transactions and data integrity. They use **primary keys** and **foreign keys** to explicitly link tables: for example, a user’s data might be split into separate `users` and `orders` tables linked by a foreign key. This normalized design (no redundant data) and the power of SQL’s declarative queries make complex joins and consistent data management possible. SQL databases also benefit from a standardized query language and broad community support. In short, using SQL and a relational DB (like PostgreSQL) means strong data integrity, rich querying, and wide tool support.

# Why Use an ORM (SQLAlchemy)?

An Object-Relational Mapper (ORM) like **SQLAlchemy** bridges Python classes and SQL tables. Rather than writing raw SQL strings, you define Python classes (models) whose attributes map to table columns; SQLAlchemy generates the underlying SQL. This lets you work with familiar Python objects while leveraging SQL’s power. For example, adding a `User` object to a session and committing it will automatically issue the correct `INSERT` statement for the `users` table. Conversely, you can traverse object relationships (e.g. `user.posts`) instead of manually joining tables. Internally, SQLAlchemy synchronizes object state and database columns: assigning related objects in Python before flush automatically sets the foreign key values in SQL on flush. In fact, you _could_ define a foreign key in the database without ever using `relationship()` in SQLAlchemy; but using `relationship()` maps that database-level link into the object world. In short, SQLAlchemy’s ORM provides a Pythonic, declarative way to define and manipulate database schemas and relationships, combining Python’s expressiveness with SQL’s efficiency.

# Defining SQLAlchemy Models (Declarative 2.0)

In SQLAlchemy 2.0, you typically define a base class (using `DeclarativeBase`) and subclass it for each table. Each class should set a `__tablename__` and use **type-annotated attributes** with `mapped_column()` to declare columns. For example:

```python
from sqlalchemy import String, Integer, DateTime, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id:   Mapped[int]               = mapped_column(primary_key=True)
    name: Mapped[str]               = mapped_column(String(50), unique=True, nullable=False)
    # created_at is timezone-aware timestamp (DateTime with timezone)
    created_at: Mapped[datetime.datetime] = mapped_column(DateTime(timezone=True), default=datetime.datetime.utcnow)

    # Relationship: a User has many Posts
    posts: Mapped[list["Post"]]     = relationship("Post", back_populates="author")
```

In this snippet, each `mapped_column()` call defines a column. The first argument (if given) is the SQL type (e.g. `String(50)`, `Integer`, `DateTime`). The attribute name (`id`, `name`, etc.) is by default used as the column name. For `id` we set `primary_key=True`, making it a primary key column. We also set `unique=True` and `nullable=False` on `name` to enforce constraints. The `created_at` column uses `DateTime(timezone=True)` so it stores a timezone-aware timestamp (and by default we use `datetime.utcnow` to populate it).

Under the hood, **every argument accepted by SQLAlchemy’s traditional `Column()`** is also valid in `mapped_column()`. This includes common keywords like `primary_key`, `nullable`, `unique`, `index`, `default`, `server_default`, `ForeignKey(...)`, etc. For instance:

- `primary_key=True` marks the column as part of the primary key.
- `nullable=False` makes the column NOT NULL.
- `unique=True` adds a UNIQUE constraint.
- `index=True` creates an index on this column.
- `ForeignKey("other_table.id")` adds a FOREIGN KEY constraint at the database level.

Thus, `mapped_column(String, ForeignKey("users.id"), default=..., index=True, comment="...")` works just like writing `Column(String, ForeignKey("users.id"), ...)` in earlier SQLAlchemy versions.

With type annotations, SQLAlchemy can even **infer columns automatically**. For example, if you wrote `fullname: Mapped[str]` without a `mapped_column()`, SQLAlchemy will auto-create a column named `fullname` with a suitable SQL type (e.g. TEXT) and default nullability. You can override this by explicitly calling `mapped_column(...)`. This integration of Python typing (`Mapped[...]`) and `mapped_column` was introduced in SQLAlchemy 2.0 for concise syntax.

# Primary Keys and Foreign Keys

A **primary key** uniquely identifies each row. In SQLAlchemy models, you designate it by `mapped_column(primary_key=True)`. Every table should have a primary key (usually a single column like `id`), which is critical for identity and relationships.

A **foreign key** is a column (or set of columns) that references another table’s primary key, enforcing referential integrity. You add it in SQLAlchemy via `ForeignKey`. For instance, in a blog example:

```python
class Post(Base):
    __tablename__ = "posts"
    id:         Mapped[int]   = mapped_column(primary_key=True)
    title:      Mapped[str]   = mapped_column(String(100))
    author_id:  Mapped[int]   = mapped_column(ForeignKey("users.id"))
    author:     Mapped[User]  = relationship("User", back_populates="posts")
```

Here `author_id` is a foreign key pointing to `users.id`. Notice we put the foreign key on the “many” side of the relationship (the `Post` table, which is many posts for one user). This is a general rule: **for one-to-many, put the foreign key on the “many” side**. In this example, each `Post` row refers to one `User`, and the `User` class has `posts` as a collection of its related posts.

# One-to-Many and Many-to-One Relationships

To model **one-to-many** (e.g. one User → many Posts), we combine a foreign key column with ORM `relationship()`. In the example above, `User.posts = relationship("Post", back_populates="author")` and `Post.author = relationship("User", back_populates="posts")` link the two sides. SQLAlchemy sees that `Post.author_id` is the only foreign key connecting to `User.id` and infers the join condition. By default, SQLAlchemy treats this as a list on the “one” side: `user.posts` will be a Python list of `Post` objects (since one user can have many posts), while `post.author` is a scalar `User`. (If you needed exactly one post per user, you would enforce that at the schema level and use `uselist=False` to make it a scalar, effectively modeling a one-to-one.)

```python
class User(Base):
    # ... as above ...
    posts: Mapped[list["Post"]] = relationship("Post", back_populates="author")
class Post(Base):
    # ... as above ...
    author: Mapped["User"] = relationship("User", back_populates="posts")
```

This `back_populates` setup tells SQLAlchemy that `User.posts` and `Post.author` are two ends of the same bidirectional relationship. Changes to one side will be reflected on the other in Python state. (The older `backref="..."` shortcut could auto-create the reverse relationship, but modern SQLAlchemy prefers explicit `back_populates`.)

Key points on one-to-many:

- **Foreign key on child/many side**: as above, `posts` table has the foreign key to `users`.
- **relationship() on parent**: typically you put `relationship()` on the “one” side to get a collection. But you can define it on either or both; just be sure to link them with `back_populates`.
- **Automatic inference**: If there is only one foreign key linking two tables, SQLAlchemy will automatically figure out the join. If multiple FKs or custom joins are needed, you may have to specify them (via `foreign_keys=...`, `primaryjoin=...`, etc.), but in the simple case no extra config is needed.
- **Collection vs. scalar**: By default `relationship()` will use a list for one-to-many and many-to-many, and a scalar for many-to-one. This is automatic when using annotations. You only use `uselist=False` for a one-to-one when you truly want a single object rather than a list.

# One-to-One Relationships

A one-to-one relationship is simply a special case of one-to-many where you constrain the relationship to at most one row. In SQLAlchemy, you still place the foreign key on one side, but you enforce a uniqueness (or use `uselist=False`) to ensure only one match. For example, each `User` might have exactly one `Profile`. You could do:

```python
class Profile(Base):
    __tablename__ = "profiles"
    id:      Mapped[int]  = mapped_column(primary_key=True)
    user_id: Mapped[int]  = mapped_column(ForeignKey("users.id"), unique=True)
    user:    Mapped[User] = relationship("User", back_populates="profile")
class User(Base):
    # ... as above ...
    profile: Mapped["Profile"] = relationship("Profile", back_populates="user", uselist=False)
```

This way, `User.profile` is a scalar (not a list) and SQLAlchemy will enforce the one-to-one nature. Alternatively, you can omit `uselist=False` but put a `unique=True` on the foreign key column in `Profile`; the end result in the database is the same (the foreign key will refer to at most one user).

# Many-to-Many Relationships and `secondary`

For many-to-many, you need an **association table**. This is a table that contains (at minimum) foreign keys to two other tables. For example, if posts can have many tags and tags can belong to many posts, you might define:

```python
from sqlalchemy import Table

post_tags = Table(
    "post_tags",
    Base.metadata,
    mapped_column("post_id", ForeignKey("posts.id"), primary_key=True),
    mapped_column("tag_id",  ForeignKey("tags.id"),  primary_key=True)
)

class Post(Base):
    __tablename__ = "posts"
    id:   Mapped[int]   = mapped_column(primary_key=True)
    title: Mapped[str]  = mapped_column(String(100))
    tags: Mapped[list["Tag"]] = relationship("Tag", secondary=post_tags, back_populates="posts")

class Tag(Base):
    __tablename__ = "tags"
    id:   Mapped[int]   = mapped_column(primary_key=True)
    name: Mapped[str]   = mapped_column(String(50), unique=True, nullable=False)
    posts: Mapped[list["Post"]] = relationship("Post", secondary=post_tags, back_populates="tags")
```

Here, `post_tags` is a bare association `Table`. Notice it has two columns: `post_id` and `tag_id`, each a foreign key to its parent table, and together they form a composite primary key. In the `relationship()`, we use the `secondary` argument to refer to this table. This tells SQLAlchemy to do a join through `post_tags` when accessing `Post.tags` or `Tag.posts`. By default `secondary` should be either the `Table` object or a string name of a table in the same metadata; using the `Table` object is more direct, but you could also write `secondary="post_tags"` if needed.

In SQLAlchemy’s docs: “Many to Many adds an association table between two classes” and you pass it via `secondary`. The ORM will manage inserts/deletes in that table for you when you append to the list. For example, `post.tags.append(tag)` will insert into `post_tags` on flush.

# Association Object (Many-to-Many with Extra Columns)

If you need to store extra data on the association (e.g. an assignment date, a quantity, a price, etc.), use the **association object pattern**. Instead of a bare table, you map a class to the association table and give it its own columns. For example, suppose we have users and groups, and we want to record when a user joined a group:

```python
class UserGroup(Base):
    __tablename__ = "user_group"
    user_id:  Mapped[int] = mapped_column(ForeignKey("users.id"), primary_key=True)
    group_id: Mapped[int] = mapped_column(ForeignKey("groups.id"), primary_key=True)
    joined_at: Mapped[datetime.datetime] = mapped_column(DateTime(timezone=True), default=datetime.datetime.utcnow)

    user:  Mapped[User]  = relationship("User", back_populates="group_links")
    group: Mapped["Group"] = relationship("Group", back_populates="user_links")

class User(Base):
    __tablename__ = "users"
    id:      Mapped[int] = mapped_column(primary_key=True)
    name:    Mapped[str] = mapped_column(String(50))
    group_links: Mapped[list["UserGroup"]] = relationship("UserGroup", back_populates="user")

class Group(Base):
    __tablename__ = "groups"
    id:      Mapped[int] = mapped_column(primary_key=True)
    name:    Mapped[str] = mapped_column(String(50))
    user_links: Mapped[list["UserGroup"]] = relationship("UserGroup", back_populates="group")
```

Now `UserGroup` is a first-class mapped class representing the join table, and has its own `joined_at` column. Users and Groups each have a `group_links` or `user_links` list referring to these objects. You can still easily navigate: `user.group_links` gives a list of `UserGroup` entries (including timestamps), or `group.user_links` likewise. You could also create a convenience property to get all `user.groups` from `user.group_links`.

The key point is that when the association has extra fields, you **map a class to it** instead of just using `secondary`. SQLAlchemy’s docs explain this as a variant of many-to-many used when “the association table contains additional columns beyond those foreign keys,” in which case you map a new class rather than using `secondary`. If no extra columns are needed, a plain association table is simpler. In other words: _use a simple `Table` + `secondary` for pure many-to-many; use an “association object” (mapped class) when you need extra columns_.

# `back_populates`, `backref`, and Bidirectional Relationships

When you define relationships on both sides of a pair (like `User.posts` and `Post.author`), link them with `back_populates`. The `back_populates` string should match the attribute name on the opposite class. For example, in our `Post`/`User` example above, we set `relationship("Post", back_populates="author")` on `User` and `relationship("User", back_populates="posts")` on `Post`. This ensures SQLAlchemy keeps both ends in sync: adding a post to `user.posts` will automatically set `post.author = user` in memory, and vice versa.

The older `backref="other_attr"` parameter would auto-create the reverse relationship for you, but it’s now considered a more “legacy” approach. Modern SQLAlchemy encourages explicitly using `back_populates` on both sides. This is more explicit and works better with type annotations and static analysis. (If you do use `backref`, you can customize it with `sqlalchemy.orm.backref`, but it is essentially sugar over `back_populates`.)

# Important `mapped_column()` and `relationship()` Parameters

Below are some key parameters of these constructs:

- **`mapped_column(type_, ..., primary_key, nullable, unique, index, default, server_default, ForeignKey, comment, etc.)`**: As noted, `mapped_column()` accepts all the usual Column arguments. You can specify column type (like `String(100)` or `Integer`), constraints (`primary_key=True`, `nullable=False`), and SQL defaults. For example, `mapped_column(String, index=True, default="")` creates a text column with an index and a default value. You can also pass `ForeignKey("other.id")` here to add a foreign key constraint. In SQLAlchemy 2.0, the attribute name (like `id` or `name`) is used as the column name by default. If you omit the first argument, SQLAlchemy will infer the SQL type from the Python type in `Mapped[...]`. Attributes with no explicit `mapped_column` but annotated `Mapped[str]` will automatically get a column created (with type and nullability inferred).

- **`relationship(argument, secondary, back_populates, backref, uselist, cascade, lazy, foreign_keys, passive_deletes, order_by, etc.)`**: The `relationship()` function has many options. Key ones include:

  - `argument`: the target class, given as a class or string (e.g. `"User"` or `User`), or even a lambda for forward references.
  - `secondary`: for many-to-many, the intermediate `Table` or table name (as shown with `post_tags`).
  - `back_populates`: the name of the attribute on the related class to link with (for bi-directional relationships).
  - `backref`: a legacy shortcut to auto-generate the reverse relationship; not recommended in new code.
  - `uselist`: boolean (usually inferred) – set to `False` to force a scalar instead of a list (used for one-to-one or many-to-one).
  - `cascade`: a string of cascade rules for the session (e.g. `"all, delete-orphan"`).
  - `lazy`: loading strategy (`"select"` (default), `"joined"`, `"subquery"`, `"dynamic"`, etc.) determining how related items are loaded.
  - `foreign_keys`: to explicitly list which columns are treated as foreign keys if the join is ambiguous.
  - `passive_deletes`, `viewonly`, `order_by`, etc.: more advanced behaviors.

For example, `relationship("Post", back_populates="author", cascade="all, delete-orphan")` would mean that when you delete a `User`, all its `posts` are also deleted. The default `cascade` is `"save-update, merge"`.

# Which Side Gets the Foreign Key (Thumb Rule)

The _child_ or _“many”_ side holds the `ForeignKey`. In a one-to-many (or many-to-one) relationship, the table on the “many” side includes a column like `parent_id = Column(ForeignKey("parent.id"))`. The _parent_ side typically has the `relationship()`. For example, as above, `posts` table has `author_id = ForeignKey("users.id")`, and `User` has `relationship("Post", ...)`. This is the convention: place FKs on the many side. In SQLAlchemy, as long as there is exactly one linking foreign key, it will match it to the relationship on the other class automatically. If there are multiple candidate foreign keys, you’d need to use `foreign_keys=` or `primaryjoin=` to disambiguate.

# Date/Time Columns and Time Zones

For timestamps, use SQLAlchemy’s `DateTime` type. If you want **time zone–aware** timestamps, set `timezone=True`. For example, `created_at = mapped_column(DateTime(timezone=True), default=datetime.datetime.utcnow)` will store the UTC time and recall it with `tzinfo=None` or UTC depending on DB support. (Note: in PostgreSQL this maps to `TIMESTAMP WITH TIME ZONE`.) As one reference notes, “SQLAlchemy’s `DateTime` type allows for a `timezone=True` argument to save a non-naive datetime object to the database”. It’s common to store all times in UTC. You can use Python’s `datetime.now(timezone.utc)` or `utcnow()` for defaults, or the SQL function `func.now()` for server-side defaults. Also consider adding columns like `updated_at = mapped_column(DateTime(timezone=True), onupdate=datetime.datetime.utcnow)` to track modifications (with `server_default=func.now()` as well, if desired). The key is: use `timezone=True` and consistent UTC handling to avoid confusion.

# Best Practices and Thumb Rules

- **Table and column naming**: Always define `__tablename__` on your models. When referring to a table in `ForeignKey` or `secondary`, use the table name (or class’ `__tablename__`). For clarity, many developers pluralize table names (e.g. `"users"`) but consistency is key.
- **Place FKs on the “many” side** of one-to-many relationships. On the “one” side define the `relationship()`, on the “many” side define the FK column.
- **Bidirectional `back_populates`**: When you want both classes to see the relationship, define `back_populates` on both sides (or use one `backref`). This keeps object state in sync in Python.
- **`secondary` vs association object**: If the join table has _no extra data_, use a `Table` and `secondary`. If it needs extra columns (timestamps, quantities, etc.), map a class (association object) instead.
- **Use `uselist=False` for one-to-one** (or a unique FK). By default, one-to-many gives lists and many-to-one gives scalars.
- **Cascades**: Decide how related objects should be saved/removed. The default is `cascade="save-update, merge"`, but you can add `"delete"`, `"delete-orphan"`, etc. (e.g. `cascade="all, delete-orphan"` means children are deleted when removed from the parent).
- **Loading strategy**: The `lazy` parameter (or now just using `select` vs `joined`) controls how related items are loaded. The default (`select`/lazy load) is fine for most cases.
- **Index FKs**: It’s often wise to index foreign key columns for performance (e.g. `mapped_column(ForeignKey("users.id"), index=True)`).
- **Timestamps**: As noted, use `DateTime(timezone=True)` for timestamp columns and default to UTC. Add `created_at` and `updated_at` columns with defaults/`onupdate` to audit changes.
- **Creating Tables**: After defining models, you can create the tables by calling `Base.metadata.create_all(engine)`. In real projects, use Alembic migrations for schema evolution.
- **Consistency**: Keep naming consistent (e.g. use `__tablename__ = "user_groups"` not `"userGroup"`). Many choose snake_case table names and CamelCase model classes.

Overall, the “thumb rule” is: **model the real-world relationships in your schema**, put the correct foreign keys on the many side, use `mapped_column()` to declare columns, and `relationship()` to navigate them. Link both sides with `back_populates` for bidirectional relationships. Use `secondary` for simple many-to-many, and use an association class if you need extra fields on that link. And always remember: the ORM layer (`relationship()`) is separate from the database constraint (`ForeignKey`). Both are important: the FK enforces referential integrity in the database, and `relationship()` tells SQLAlchemy how to load related objects in Python.

This setup – defined tables, columns, keys, and relationships – gives you a **fully fledged SQLAlchemy schema** (for PostgreSQL) using the SQLAlchemy 2.0 style. The example above (Users, Posts, Tags, Groups) demonstrates one-to-many, many-to-many, association objects, primary keys, foreign keys, and timestamp columns all together. By following these patterns and parameters, you can design complex schemas with SQLAlchemy while ensuring clarity and data integrity.

**Sources:** SQLAlchemy 2.0 Documentation (Declarative Mapping, Relationships); StackOverflow Q/As on relationships and foreign keys; SQL concepts references.
