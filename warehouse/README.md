# On-Premise Data Warehouse (Spark + Iceberg)

Fully on-premises data warehouse with **Spark**, **Polaris**, and **MinIO**. No cloud, no external dependencies.

## Setup

```bash
cd warehouse

# One-time setup
docker network create dasnet
docker volume create warehouse_storage

# Start services
docker-compose up -d
docker-compose -f spark-notebook-compose.yml up -d
```

Access:
- **Jupyter**: http://localhost:8888
- **MinIO**: http://localhost:9001 (minioadmin/minioadmin)
- **Spark UI**: http://localhost:4040 (during queries)



## Quick Start in Jupyter

1. Open http://localhost:8888 in your browser
2. Create a new Python notebook (or use `getting_started.ipynb`)
3. The **Spark session is auto-created** — just use `spark` directly:

```python
# Create schema
spark.sql("CREATE NAMESPACE IF NOT EXISTS mydb")

# Create Iceberg table
spark.sql("""
    CREATE TABLE mydb.users (
        id INT,
        name STRING,
        created_at TIMESTAMP
    )
    USING ICEBERG
    PARTITIONED BY (CAST(created_at AS DATE))
""")

# Insert data
spark.sql("""
    INSERT INTO mydb.users VALUES
    (1, 'Alice', CURRENT_TIMESTAMP()),
    (2, 'Bob', CURRENT_TIMESTAMP())
""")

# Query data (simple pattern: schema.tablename)
spark.sql("SELECT * FROM mydb.users").show()

# DataFrame API
df = spark.table("mydb.users")
df.filter(df.id > 1).show()
```

## Key Commands

| Task | Command |
|------|---------|
| List tables | `spark.sql("SHOW TABLES IN mydb").show()` |
| Describe schema | `spark.sql("DESCRIBE TABLE mydb.users").show()` |
| Count rows | `spark.sql("SELECT COUNT(*) FROM mydb.users").show()` |
| Time travel | `spark.sql("SELECT * FROM mydb.users VERSION AS OF 0").show()` |
| Add column | `spark.sql("ALTER TABLE mydb.users ADD COLUMN email STRING")` |
| Update | `spark.sql("UPDATE mydb.users SET name='Alice2' WHERE id=1")` |
| Delete | `spark.sql("DELETE FROM mydb.users WHERE id=1")` |

## Features

✅ **Spark** - Query engine with SQL & DataFrame API  
✅ **Iceberg** - ACID, time travel, schema evolution, partitioning  
✅ **Polaris** - Metadata catalog (via PostgreSQL)  
✅ **MinIO** - On-premise S3-compatible storage  
✅ **Jupyter** - Interactive notebooks  
✅ **No cloud** - Everything runs locally  
✅ **Data persists** - Tables survive container restarts  

## Where Is My Data?

- **Parquet files**: MinIO (http://localhost:9001)  
  Navigate: `warehouse` → `polaris` → `mydb` → `users` → `data`
- **Metadata**: PostgreSQL (managed by Polaris)

## Credentials

- MinIO: `minioadmin` / `minioadmin`
- Polaris: `admin` / `polaris`

## Shutdown

```bash
# Safe (keeps data)
docker-compose down
docker-compose -f spark-notebook-compose.yml down

# Full cleanup (deletes data)
docker-compose down -v
docker-compose -f spark-notebook-compose.yml down -v
docker network rm dasnet
docker volume rm warehouse_storage
```

## Examples

See `workspace/notebooks/getting_started.ipynb` for complete working examples.

---

**Ready to query! Open Jupyter and start creating tables. 🚀**
