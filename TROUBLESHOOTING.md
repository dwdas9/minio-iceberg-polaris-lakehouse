# Troubleshooting

Common issues you'll hit and how to fix them.

---

## "Bucket does not exist" (404 error)

MinIO starts with zero buckets. Polaris is configured to store data in `s3://warehouse/`, but that bucket doesn't exist yet.

**Fix:** Open MinIO console at <http://localhost:9001>, login with `minioadmin`/`minioadmin`, create a bucket called `warehouse`.

Or via command line:

```bash
docker exec warehouse-minio mc mb local/warehouse
```

---

## "401 Unauthorized" from Polaris

Polaris credentials change on every restart. Your old credentials are now invalid.

**Fix:**

```bash
# Mac/Linux
docker logs warehouse-polaris 2>&1 | grep "credentials:"

# Windows
docker logs warehouse-polaris 2>&1 | Select-String -Pattern "credentials:"
```

Update the setup script and notebook with the new credentials, then re-run the setup script.

---

## "Failed to get subscoped credentials" / STS 403 Error

This is the big one. Polaris tries to use AWS STS for temporary credentials, but MinIO doesn't support STS.

```text
The security token included in the request is invalid. (Service: Sts, Status Code: 403)
```

**Why it's confusing:** Namespace and table creation work fine because they only touch Polaris metadata. Data operations (INSERT/SELECT) trigger the STS logic and fail.

**Fix:** The catalog needs `stsUnavailable: true` in its storage config. The setup scripts already include this, so just make sure you've run `setup-polaris.sh` or `setup-polaris.ps1` after starting Polaris.

---

## "No FileSystem for scheme 's3'"

Spark doesn't know how to read `s3://` URIs.

**Fix:** Already configured in the notebooks, but if you're rolling your own Spark session:

```python
.config("spark.hadoop.fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
```

---

## "Unable to load region from providers"

AWS SDK v2 is stricter than v1 - it requires `AWS_REGION` to be set.

**Fix:** At the top of your notebook:

```python
import os
os.environ['AWS_REGION'] = 'us-east-1'
os.environ['AWS_ACCESS_KEY_ID'] = 'minioadmin'
os.environ['AWS_SECRET_ACCESS_KEY'] = 'minioadmin'
```

---

## "NoClassDefFoundError: software/amazon/awssdk"

Missing AWS SDK v2 classes. Iceberg uses SDK v2 internally, but you only have SDK v1.

**Fix:** Add `iceberg-aws-bundle` to your JARs:

```python
"org.apache.iceberg:iceberg-aws-bundle:1.4.3"
```

---

## Spark session hangs forever

Usually means Polaris isn't reachable.

**Fix:** Check if containers are running:

```bash
docker ps
```

If warehouse-polaris isn't there, bring up infrastructure again with `docker-compose up -d`.

---

## "Connection refused" errors

Docker network issue.

**Fix:** Make sure all containers are on the same network:

```bash
docker network inspect dasnet
```

You should see all containers listed. If not, recreate the network and restart containers.

---

## Out of memory / container killed

Docker Desktop needs more RAM. Spark is memory hungry.

**Fix:** Docker Desktop → Settings → Resources → bump Memory to 10-12GB.

---

## Container networking basics

- Containers can't reach `localhost` on your host machine
- Use `host.docker.internal` from containers to reach host services
- Use container names (like `minio`, `polaris`) for container-to-container communication on the same Docker network

---

## Nuclear option - start fresh

When nothing's working:

```bash
docker-compose down -v
docker-compose -f spark-notebook-compose.yml down -v
docker network rm dasnet
docker volume rm warehouse_storage
```

Then start from scratch. Your data will be gone, but sometimes that's what you need.
