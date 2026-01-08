# Rails Backend: SQLite3 (Local) vs PostgreSQL (RDS) Tips

## Overview

When developing Rails applications with **SQLite3 for local development** and **PostgreSQL for production (AWS RDS)**, there are critical differences that must be addressed to avoid runtime errors.

---

## 1. UUID Primary Keys

### Problem

PostgreSQL has native UUID support with `gen_random_uuid()` function, but SQLite3 does NOT auto-generate UUIDs.

### Error Example

```
SQLite3::ConstraintException: NOT NULL constraint failed: users.id
ActiveRecord::NotNullViolation
```

This occurs when:
- Migration uses `id: :uuid`
- SQLite3 tries to INSERT without an ID value
- PostgreSQL would auto-generate, but SQLite3 cannot

### Solution

Add UUID auto-generation callback in `ApplicationRecord`:

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  # Auto-generate UUID for SQLite3 compatibility
  before_create :set_uuid

  private

  def set_uuid
    self.id ||= SecureRandom.uuid if self.class.columns_hash['id']&.type == :string
  end
end
```

### Alternative: Environment-specific Approach

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  before_create :set_uuid, unless: -> { Rails.configuration.database_configuration[Rails.env]['adapter'] == 'postgresql' }

  private

  def set_uuid
    self.id ||= SecureRandom.uuid
  end
end
```

---

## 2. Database Configuration (database.yml)

### Recommended Configuration

```yaml
# config/database.yml
default: &default
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  adapter: sqlite3
  database: storage/development.sqlite3

test:
  <<: *default
  adapter: sqlite3
  database: storage/test.sqlite3

production:
  primary:
    adapter: postgresql
    encoding: unicode
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
    database: <%= ENV['DB_NAME'] %>
    username: <%= ENV['DB_USERNAME'] %>
    password: <%= ENV['DB_PASSWORD'] %>
    host: <%= ENV['DB_HOST'] %>
    port: <%= ENV['DB_PORT'] || 5432 %>
```

### Gemfile Configuration

```ruby
# Gemfile
group :development, :test do
  gem "sqlite3", ">= 2.1"
end

group :production do
  gem "pg"
end
```

---

## 3. JSON vs JSONB Column Types

### Difference

| Feature | SQLite3 | PostgreSQL |
|---------|---------|------------|
| `json` | Stored as TEXT | Native JSON |
| `jsonb` | NOT SUPPORTED | Binary JSON (indexed, faster queries) |

### Solution

Use `json` type in migrations for cross-database compatibility:

```ruby
# Migration
create_table :workflows do |t|
  t.json :parameters_schema  # Use 'json', not 'jsonb'
  t.json :constraints
end
```

If you need PostgreSQL-specific `jsonb` features in production, use conditional migrations:

```ruby
# Migration with adapter check
def change
  create_table :workflows do |t|
    if ActiveRecord::Base.connection.adapter_name == 'PostgreSQL'
      t.jsonb :parameters_schema
    else
      t.json :parameters_schema
    end
  end
end
```

---

## 4. Integer Types (bigint)

### Difference

| Type | SQLite3 | PostgreSQL |
|------|---------|------------|
| `integer` | Up to 8 bytes (flexible) | 4 bytes |
| `bigint` | Same as integer | 8 bytes |

### Recommendation

Always use `bigint` for large values (file sizes, timestamps):

```ruby
# Migration
t.bigint :size_bytes  # Explicit bigint for file sizes
```

SQLite3 handles this transparently, but PostgreSQL needs explicit `bigint` for values > 2^31.

---

## 5. UUID Foreign Key References

### Problem

When using UUID primary keys, foreign key references must also use UUID type.

### Correct Migration

```ruby
create_table :runs, id: :uuid do |t|
  t.references :user, null: false, foreign_key: true, type: :uuid
  t.references :workflow, null: false, foreign_key: true, type: :uuid
end

create_table :contribution_assertions, id: :uuid do |t|
  t.references :contributor, null: false, foreign_key: { to_table: :users }, type: :uuid
  t.references :block, null: true, foreign_key: true, type: :uuid
end
```

### Key Points

- Always add `type: :uuid` to `t.references` when referencing UUID tables
- Use `foreign_key: { to_table: :table_name }` for non-standard foreign key names

---

## 6. Polymorphic Associations with UUID

```ruby
# Migration
t.references :subject, polymorphic: true, null: false, type: :uuid

# This creates:
# - subject_type (string)
# - subject_id (uuid)
```

---

## 7. Common Pitfalls Checklist

Before running migrations, verify:

- [ ] All `id: :uuid` tables have UUID auto-generation in ApplicationRecord
- [ ] All `t.references` include `type: :uuid` when referencing UUID tables
- [ ] JSON columns use `json` (not `jsonb`) for SQLite3 compatibility
- [ ] File size columns use `bigint`
- [ ] `database.yml` correctly separates SQLite3 (dev/test) and PostgreSQL (production)
- [ ] Gemfile has `sqlite3` in dev/test group and `pg` in production group

---

## 8. Testing Both Databases

### Local Development (SQLite3)

```bash
RAILS_ENV=development bin/rails db:create db:migrate
bin/rails c
# Test CRUD operations
```

### Production Simulation (PostgreSQL via Docker)

```bash
# docker-compose.yml with PostgreSQL for local testing
docker-compose up -d db
RAILS_ENV=production DATABASE_URL=postgres://user:pass@localhost:5432/app_prod bin/rails db:create db:migrate
```

---

## 9. Migration Rollback Safety

When rolling back migrations with UUID:

```bash
# SQLite3: Delete database files
rm -f storage/development.sqlite3 storage/test.sqlite3

# Recreate
bin/rails db:create db:migrate
```

---

## Summary

| Feature | SQLite3 (Dev/Test) | PostgreSQL (Production) |
|---------|-------------------|-------------------------|
| UUID Generation | Manual (SecureRandom.uuid) | Native (gen_random_uuid) |
| JSON Type | `json` only | `json` or `jsonb` |
| Integer Range | Flexible | Explicit `bigint` needed |
| Performance | File-based, simple | Network, connection pool |

**Key Rule**: Always design for the lowest common denominator (SQLite3), then optimize for PostgreSQL in production if needed.
