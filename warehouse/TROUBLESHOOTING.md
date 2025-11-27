# Troubleshooting Guide

## 🔍 Debugging Journey: The AWS STS Problem

This document captures the **real debugging process** we went through to get Polaris + Iceberg + MinIO working. If you're hitting similar issues, this shows you exactly how we solved them.

---

## 🚨 The Main Problem: AWS STS Credential Vending

### Initial Error

```
org.apache.iceberg.exceptions.ForbiddenException: Failed to get subscoped credentials:
The security token included in the request is invalid. (Service: Sts, Status Code: 403)
```

### Why This Happened

**The Architecture Assumption**:
- Polaris was designed for AWS S3 in cloud environments
- In AWS, you use IAM roles and STS (Security Token Service) for temporary credentials
- Polaris assumed it could call AWS STS to get temporary, scoped credentials for each operation

**The Reality with MinIO**:
- MinIO is S3-compatible but **doesn't implement AWS STS**
- MinIO only supports static access keys (like `minioadmin`/`minioadmin`)
- When Polaris tried to call STS, MinIO returned 403 Forbidden

### The Confusion

This was confusing because:
1. **Namespaces worked fine** - creating `polaris.test_db` succeeded
2. **Table creation worked** - `CREATE TABLE` succeeded
3. **Only writes/reads failed** - `INSERT` and `SELECT` triggered the STS error

**Why?** Namespace and table creation only touch Polaris metadata (PostgreSQL). Data operations require actual S3 access, which triggered Polaris's STS credential vending logic.

---

## 🔧 Solutions We Tried (In Order)

### ❌ Attempt 1: Add `credential-vending-enabled: false` to Catalog Properties

```json
{
  "name": "my_catalog",
  "properties": {
    "credential-vending-enabled": "false",
    "default-base-location": "s3://warehouse/"
  }
}
```

**Result**: ❌ Property not recognized by Polaris. No effect.

**Lesson**: Not all configuration properties are documented or supported.

---

### ❌ Attempt 2: Set `roleArn: null` in storageConfigInfo

```json
{
  "storageConfigInfo": {
    "storageType": "S3",
    "roleArn": null,
    "allowedLocations": ["s3://warehouse/"]
  }
}
```

**Result**: ❌ Error: "ARN must not be empty"

**Lesson**: Polaris validates that `roleArn` is a valid AWS ARN string if provided.

---

### ❌ Attempt 3: Set `roleArn: ""`

```json
{
  "storageConfigInfo": {
    "storageType": "S3",
    "roleArn": "",
    "allowedLocations": ["s3://warehouse/"]
  }
}
```

**Result**: ❌ Error: "ARN must not be empty"

**Lesson**: Empty string is not a valid ARN.

---

### ❌ Attempt 4: Change `storageType: "FILE"`

```json
{
  "storageConfigInfo": {
    "storageType": "FILE",
    "allowedLocations": ["s3://warehouse/"]
  }
}
```

**Result**: ❌ Error: "Unsupported storage type: FILE"

**Lesson**: Polaris only supports `S3` storage type for remote object storage.

---

### ❌ Attempt 5: Add Environment Variable `POLARIS_FEATURE_CATALOG_CREDENTIAL_VENDING_ENABLED=false`

```yaml
polaris:
  environment:
    POLARIS_FEATURE_CATALOG_CREDENTIAL_VENDING_ENABLED: "false"
```

**Result**: ❌ Environment variable not recognized. No effect.

**Lesson**: This environment variable doesn't exist in Polaris 1.2.0. May exist in future versions.

---

### ❌ Attempt 6: Use `EXTERNAL` Catalog Type

```json
{
  "name": "my_catalog",
  "type": "EXTERNAL",
  "remoteUrl": "unused",
  "properties": {
    "default-base-location": "s3://warehouse/"
  }
}
```

**Result**: ❌ Error: "Bad Request (400)"

**Lesson**: `EXTERNAL` catalog type is for federated catalogs that delegate to another catalog service, not for direct S3 storage.

---

### ✅ Attempt 7: Use `stsUnavailable: true` (SOLUTION!)

After reviewing an online article using Polaris with MinIO, we found the magic flag:

```json
{
  "catalog": {
    "name": "my_catalog",
    "type": "INTERNAL",
    "properties": {
      "default-base-location": "s3://warehouse/"
    },
    "storageConfigInfo": {
      "storageType": "S3",
      "allowedLocations": ["s3://warehouse/*"],
      "region": "us-east-1",
      "endpoint": "http://minio:9000",
      "pathStyleAccess": true,
      "stsUnavailable": true  // ← THE FIX!
    }
  }
}
```

**Result**: ✅ **IT WORKED!**

**What `stsUnavailable: true` does**:
- Tells Polaris: "This S3 endpoint doesn't support AWS STS"
- Polaris then uses **static credentials** instead of trying to vend temporary tokens
- Static credentials come from environment variables: `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`

---

## 🐛 Secondary Issues We Hit

### Issue 1: "No FileSystem for scheme 's3'"

**Error**:
```
org.apache.hadoop.fs.UnsupportedFileSystemException: No FileSystem for scheme "s3"
```

**Cause**: Polaris returns `s3://` URIs, but Spark didn't know how to read them.

**Solution**: Configure S3A filesystem in Spark:
```python
.config("spark.hadoop.fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
.config("spark.hadoop.fs.s3a.endpoint", "http://minio:9000")
.config("spark.hadoop.fs.s3a.access.key", "minioadmin")
.config("spark.hadoop.fs.s3a.secret.key", "minioadmin")
.config("spark.hadoop.fs.s3a.path.style.access", "true")
.config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")
```

**Lesson**: Spark needs explicit S3A configuration even though Iceberg uses its own S3 client.

---

### Issue 2: "NoClassDefFoundError: software/amazon/awssdk"

**Error**:
```
java.lang.NoClassDefFoundError: software/amazon/awssdk/services/s3/model/S3Exception
```

**Cause**: Iceberg uses **AWS SDK v2** internally, but we only had AWS SDK v1 (aws-java-sdk-bundle).

**Solution**: Add `iceberg-aws-bundle`:
```python
"spark.jars.packages": 
    "org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.4.3,"
    "org.apache.hadoop:hadoop-aws:3.3.4,"
    "com.amazonaws:aws-java-sdk-bundle:1.12.262,"  # AWS SDK v1
    "org.apache.iceberg:iceberg-aws-bundle:1.4.3"   # AWS SDK v2 ← ADDED
```

**Lesson**: Iceberg needs both SDK versions. AWS SDK v1 for S3A, AWS SDK v2 for Iceberg's own S3 operations.

---

### Issue 3: "Unable to load region from providers"

**Error**:
```
Unable to load region from any of the providers in the chain
```

**Cause**: AWS SDK v2 requires `AWS_REGION` environment variable (unlike SDK v1).

**Solution**: Set environment variables in notebook:
```python
import os
os.environ['AWS_REGION'] = 'us-east-1'
os.environ['AWS_ACCESS_KEY_ID'] = 'minioadmin'
os.environ['AWS_SECRET_ACCESS_KEY'] = 'minioadmin'
```

**Lesson**: AWS SDK v2 is stricter about configuration than v1.

---

### Issue 4: "Unable to load credentials from providers"

**Error**:
```
Unable to load credentials from any of the providers in the chain
```

**Cause**: AWS SDK v2 credential providers check:
1. System properties (`aws.accessKeyId`)
2. Environment variables (`AWS_ACCESS_KEY_ID`)
3. AWS credentials file (`~/.aws/credentials`)
4. Container credentials (ECS/EKS)
5. EC2 instance profile

None of these were set in the Spark container.

**Solution**: Set environment variables (same as Issue 3 above).

**Lesson**: Always set AWS credentials when using SDK v2, even with MinIO.

---

## 📊 Debugging Timeline

| Attempt | Approach | Time Spent | Result |
|---------|----------|------------|--------|
| 1 | Add catalog property | 15 min | ❌ Failed |
| 2-3 | Try roleArn variations | 20 min | ❌ Failed |
| 4 | Change storage type | 10 min | ❌ Failed |
| 5 | Environment variable | 15 min | ❌ Failed |
| 6 | EXTERNAL catalog | 20 min | ❌ Failed |
| 7 | Research online articles | 30 min | ✅ Found solution |
| 8 | Fix S3A filesystem | 10 min | ✅ Fixed |
| 9 | Add AWS SDK v2 bundle | 15 min | ✅ Fixed |
| 10 | Set AWS environment vars | 10 min | ✅ Fixed |
| **Total** | | **~2.5 hours** | ✅ **Working!** |

---

## 🎓 Key Lessons Learned

### 1. **MinIO ≠ AWS S3**
MinIO is S3-compatible for data operations, but doesn't implement AWS-specific services like STS, IAM, or CloudWatch. Always check if your storage backend supports the features your catalog expects.

### 2. **Polaris Assumes AWS by Default**
Polaris was built for cloud environments and assumes:
- AWS STS is available
- IAM roles work
- Temporary credentials can be vended

When using with MinIO, you must explicitly disable these assumptions.

### 3. **Configuration Is Not Documented**
The `stsUnavailable` flag is **not in official Polaris documentation**. We only found it by:
- Reading GitHub issues
- Examining example configurations from blog posts
- Trial and error

**Takeaway**: Community resources are often more valuable than official docs for cutting-edge features.

### 4. **Iceberg Uses Its Own S3 Client**
Even though Spark uses S3A (hadoop-aws) for filesystem operations, Iceberg has its own S3 client (AWS SDK v2). This means:
- You need **both** SDK versions
- You need **both** sets of configurations
- Credentials must be set in **multiple places**

### 5. **Environment Variables Matter**
AWS SDK v2 is stricter than v1:
- Requires `AWS_REGION` (v1 had defaults)
- Requires explicit credential providers
- Doesn't fall back gracefully

Always set: `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

### 6. **Container Networking Is Tricky**
- Spark runs **inside** a container
- Polaris/MinIO run in **other** containers
- `localhost` means different things in each container
- Solution: Use `host.docker.internal` from Spark to reach host services
- Alternative: Use container names if on same Docker network (e.g., `http://minio:9000`)

### 7. **Polaris Credentials Change on Restart**
Every time Polaris restarts:
- New bootstrap credentials are generated
- Previous credentials become invalid
- You must update `setup-polaris.ps1` and re-run it
- You must update notebook with new credentials

**Mitigation**: Don't restart Polaris unnecessarily. When you do, immediately get new credentials from logs.

---

## 🔬 How to Debug Similar Issues

### Step 1: Check Container Logs

```powershell
# Polaris logs (most important)
docker logs warehouse-polaris

# MinIO logs
docker logs warehouse-minio

# PostgreSQL logs
docker logs warehouse-postgres

# Spark logs (from notebook cell output)
```

### Step 2: Test Each Component Independently

**Test MinIO**:
```powershell
# Using MinIO client
docker run --rm --network dasnet minio/mc alias set myminio http://minio:9000 minioadmin minioadmin
docker run --rm --network dasnet minio/mc ls myminio
```

**Test Polaris REST API**:
```powershell
# Get OAuth token
$token = (Invoke-RestMethod -Uri "http://localhost:8181/api/catalog/v1/oauth/tokens" -Method Post -ContentType "application/x-www-form-urlencoded" -Body @{grant_type="client_credentials"; client_id="<ID>"; client_secret="<SECRET>"; scope="PRINCIPAL_ROLE:ALL"}).access_token

# List catalogs
Invoke-RestMethod -Uri "http://localhost:8181/api/management/v1/catalogs" -Headers @{Authorization="Bearer $token"}
```

**Test PostgreSQL**:
```powershell
docker exec -it warehouse-postgres psql -U polaris -d polaris_db -c "SELECT COUNT(*) FROM POLARIS.CATALOG;"
```

### Step 3: Enable Verbose Logging

In Spark notebook:
```python
# Set Spark log level
spark.sparkContext.setLogLevel("DEBUG")

# Enable Iceberg logging
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Step 4: Check Network Connectivity

```powershell
# From Spark container, test connectivity
docker exec -it <spark-container> curl http://host.docker.internal:8181/healthcheck
docker exec -it <spark-container> curl http://minio:9000/minio/health/live
```

### Step 5: Verify Configuration

```python
# In Spark notebook, check catalog config
spark.sql("DESCRIBE EXTENDED polaris").show(truncate=False)

# Check Iceberg metadata
spark.sql("SELECT * FROM polaris.test_db.test_table.metadata").show()
```

---

## 📋 Checklist When Things Don't Work

### Polaris Issues
- [ ] Container is running: `docker ps | grep polaris`
- [ ] Health check passing: `curl http://localhost:8181/healthcheck`
- [ ] Bootstrap credentials obtained: `docker logs warehouse-polaris | grep credentials`
- [ ] Catalog created: Check setup-polaris.ps1 output
- [ ] PostgreSQL accessible: `docker exec -it warehouse-postgres psql -U polaris -d polaris_db`

### MinIO Issues
- [ ] Container running: `docker ps | grep minio`
- [ ] Console accessible: http://localhost:9001
- [ ] Bucket exists: Check console or use `mc ls`
- [ ] Credentials correct: `minioadmin` / `minioadmin`

### Spark Issues
- [ ] Container running: `docker ps | grep pyspark`
- [ ] Jupyter accessible: http://localhost:8888
- [ ] JARs downloaded: Check first cell output (2-3 minutes first time)
- [ ] Environment variables set: `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- [ ] Credentials match Polaris: Check `setup-polaris.ps1` output

### Network Issues
- [ ] `dasnet` network exists: `docker network ls`
- [ ] All containers on `dasnet`: `docker network inspect dasnet`
- [ ] Ports not conflicting: Check `docker ps` ports column
- [ ] Firewall not blocking: Check Windows Firewall / antivirus

---

## 🆘 Still Having Issues?

### Collect Debug Information

```powershell
# Save all logs
docker logs warehouse-postgres > postgres.log 2>&1
docker logs warehouse-minio > minio.log 2>&1
docker logs warehouse-polaris > polaris.log 2>&1

# Export configuration
docker inspect warehouse-polaris > polaris-config.json
docker network inspect dasnet > network-config.json

# Check volumes
docker volume inspect warehouse_postgres_data
docker volume inspect warehouse_minio_data
```

### Nuclear Option: Start Fresh

```powershell
# Stop everything
docker-compose down -v
docker-compose -f spark-notebook-compose.yml down -v

# Remove network
docker network rm dasnet

# Remove all containers
docker rm -f $(docker ps -aq)

# Remove all volumes (WARNING: Deletes all data!)
docker volume prune -f

# Start from Step 1 of setup guide
```

---

## 📚 Resources That Helped

1. **Alex Merced's Blog**: Found the `stsUnavailable` flag
2. **Polaris GitHub Issues**: Similar MinIO integration problems
3. **Iceberg Documentation**: Understanding metadata structure
4. **Stack Overflow**: AWS SDK v1 vs v2 differences
5. **Docker Documentation**: Container networking patterns

---

## 🎉 Success Indicators

You know it's working when:
- ✅ `CREATE TABLE` succeeds
- ✅ `INSERT` succeeds without STS errors
- ✅ `SELECT` returns data
- ✅ MinIO console shows `.parquet` files
- ✅ No errors about credentials or STS
- ✅ Time travel queries work
- ✅ Data persists after container restart
