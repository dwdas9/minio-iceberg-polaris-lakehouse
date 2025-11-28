# Mac Setup

You need Docker Desktop for Mac - get the Apple Silicon version if you're on M1/M2/M3. The Intel version works through Rosetta but runs hotter.

After installing, bump up Docker resources. Settings → Resources → give it 8GB RAM minimum (12GB is better for Spark), 4 CPUs. Default 2GB will make Spark cry.

## Getting It Running

```bash
# One-time setup
docker network create dasnet
docker volume create warehouse_storage

# Start infrastructure
docker-compose up -d
```

Wait 30 seconds, then grab the Polaris credentials (these change on every restart):

```bash
docker logs warehouse-polaris 2>&1 | grep "credentials:"
```

Output looks like: `credentials: a78cf5b1274db27a:aac273037efbfd31ae9b285c7eb206a1`

First part is client ID, second part is secret.

## Create the MinIO Bucket

This trips everyone up. MinIO starts empty, but Polaris expects `s3://warehouse/` to exist.

Go to http://localhost:9001, login with `minioadmin`/`minioadmin`, create a bucket called `warehouse`. Skip this and you'll get 404 errors later.

## Set Up Polaris Catalog

Update `setup-polaris.sh` with your credentials, then:

```bash
chmod +x setup-polaris.sh
./setup-polaris.sh
```

If you get a 401 error, credentials are wrong - grab fresh ones from the logs.

## Start Jupyter

```bash
docker-compose -f spark-notebook-compose.yml up -d
```

First run pulls ~4GB, takes a few minutes. Once done, open http://localhost:8888.

Go to `work/getting_started.ipynb`, update the credentials in the Spark config, run the first cell. First run takes 2-3 minutes to download JARs. After that, if tables are getting created and queries work - you're done.

## Quick Reference

- Jupyter: http://localhost:8888
- MinIO Console: http://localhost:9001 (minioadmin/minioadmin)
- Spark UI: http://localhost:4040 (only when running queries)
- Polaris API: http://localhost:8181

## Nuke Everything

```bash
docker-compose down -v
docker-compose -f spark-notebook-compose.yml down -v
docker network rm dasnet
docker volume rm warehouse_storage
```

Then start from scratch.
