# Lakehouse Engine Setup with Docker Compose

This document provides a detailed guide to set up a lakehouse engine for testing. The solution leverages **MinIO**, **Apache Iceberg**, **Apache Nessie**, **Trino**, **Apache Spark**, and **Taipy** for a fully open-source and Python-centric architecture.

---

## Prerequisites

1. **Docker** and **Docker Compose** installed on your machine.
2. Access to an existing **S3-compatible MinIO bucket** or create one as described in this guide.

---

## Directory Structure

```
lakehouse/
├── docker-compose.yml
├── minio_data/               # Data directory for MinIO
├── trino_config/             # Configuration for Trino
│   ├── etc
│       ├── catalog
│           ├── iceberg.properties
├── spark_config/             # Configuration for Spark
│   ├── spark-defaults.conf
├── taipy/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── taipy_app.py
```

---

## Step 1: Docker Compose File

Create a `docker-compose.yml` file to define the services:

```yaml
version: '3.8'

services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    environment:
      MINIO_ROOT_USER: "minioadmin"
      MINIO_ROOT_PASSWORD: "minioadmin"
    command: server /data
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./minio_data:/data

  nessie:
    image: projectnessie/nessie
    container_name: nessie
    ports:
      - "19120:19120"
    environment:
      QUARKUS_PROFILE: "prod"

  trino:
    image: trinodb/trino:latest
    container_name: trino
    ports:
      - "8080:8080"
    volumes:
      - ./trino_config:/etc/trino
    environment:
      JAVA_TOOL_OPTIONS: "-Xmx4g -XX:+ExitOnOutOfMemoryError"

  spark:
    image: bitnami/spark:latest
    container_name: spark
    environment:
      - SPARK_MODE=master
    volumes:
      - ./spark_config:/opt/bitnami/spark/conf
    ports:
      - "7077:7077"
      - "8081:8081"

  taipy:
    build: ./taipy
    container_name: taipy
    ports:
      - "5000:5000"
    depends_on:
      - trino
      - nessie
      - minio
      - spark
```

---

## Step 2: MinIO Configuration

MinIO provides S3-compatible object storage.

1. **Start MinIO**:
   After running the Docker Compose setup, MinIO will be available at:
   - **Console**: [http://localhost:9001](http://localhost:9001)
   - **API**: [http://localhost:9000](http://localhost:9000)

2. **Create a Bucket**:
   Use the MinIO Client (mc) to create a bucket:
   ```bash
   mc alias set local http://localhost:9000 minioadmin minioadmin
   mc mb local/lakehouse-bucket
   ```

---

## Step 3: Trino Configuration

Trino is the query engine for the lakehouse.

1. Create `trino_config/etc/catalog/iceberg.properties`:
   ```properties
   connector.name=iceberg
   catalog.type=nessie
   nessie.uri=http://nessie:19120/api/v1
   iceberg.file-format=parquet
   hive.s3.endpoint=http://minio:9000
   hive.s3.aws-access-key=minioadmin
   hive.s3.aws-secret-key=minioadmin
   hive.s3.path-style-access=true
   ```

2. Trino will be accessible at [http://localhost:8080](http://localhost:8080).

---

## Step 4: Spark Configuration

Apache Spark enables large-scale distributed data processing.

1. **Create `spark_config/spark-defaults.conf`**:
   ```properties
   spark.hadoop.fs.s3a.access.key=minioadmin
   spark.hadoop.fs.s3a.secret.key=minioadmin
   spark.hadoop.fs.s3a.endpoint=http://minio:9000
   spark.sql.catalog.nessie=org.apache.iceberg.spark.SparkCatalog
   spark.sql.catalog.nessie.catalog-impl=org.apache.iceberg.nessie.NessieCatalog
   spark.sql.catalog.nessie.uri=http://nessie:19120/api/v1
   spark.sql.catalog.nessie.ref=main
   spark.sql.catalog.nessie.warehouse=s3a://lakehouse-bucket/warehouse/
   ```

2. Spark will be accessible at [http://localhost:8081](http://localhost:8081).

---

## Step 5: Taipy Application

1. **Create `taipy/Dockerfile`**:
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   # Install dependencies
   COPY requirements.txt .
   RUN pip install -r requirements.txt

   # Copy application
   COPY taipy_app.py .

   EXPOSE 5000
   CMD ["python", "taipy_app.py"]
   ```

2. **Create `taipy/requirements.txt`**:
   ```plaintext
   taipy
tableview
   pynessie
   requests
   pandas
   ```

3. **Create `taipy/taipy_app.py`**:
   ```python
   from taipy.gui import Gui
   import requests

   def query_trino(sql):
       url = "http://trino:8080/v1/statement"
       headers = {
           "X-Trino-User": "admin",
           "X-Trino-Catalog": "iceberg",
           "X-Trino-Schema": "default",
       }
       response = requests.post(url, data=sql, headers=headers)
       if response.status_code == 200:
           results_url = response.json()["nextUri"]
           return requests.get(results_url).json()["data"]
       else:
           return []

   page = """
   # Lakehouse Dashboard

   <|layout|columns=2 1|gap=10px|

   ### Query Editor
   <|Input|value=sql_query|label="SQL Query"|>

   <|Run Query|button|on_action=run_query|style=primary|>

   ### Query Results
   <|Table|value=query_results|width=800px|height=400px|>

   |>
   """

   sql_query = ""
   query_results = []

   def run_query(state):
       global sql_query, query_results
       sql_query = state.sql_query
       query_results = query_trino(sql_query)

   gui = Gui(page)
   gui.run(host="0.0.0.0", port=5000)
   ```

---

## Step 6: Start Services

Run the following command to start all services:
```bash
docker-compose up --build
```

Verify:
- MinIO: [http://localhost:9001](http://localhost:9001)
- Nessie: [http://localhost:19120](http://localhost:19120)
- Trino: [http://localhost:8080](http://localhost:8080)
- Spark: [http://localhost:8081](http://localhost:8081)
- Taipy: [http://localhost:5000](http://localhost:5000)

---

## Testing the Setup

1. **Create an Iceberg Table**:
   Use Trino or Taipy’s SQL input:
   ```sql
   CREATE TABLE iceberg.default.sample_table (
       id BIGINT,
       name STRING,
       timestamp TIMESTAMP
   )
   PARTITIONED BY (timestamp);
   ```

2. **Insert Data**:
   ```sql
   INSERT INTO iceberg.default.sample_table VALUES
   (1, 'Alice', TIMESTAMP '2023-01-01 10:00:00'),
   (2, 'Bob', TIMESTAMP '2023-01-02 11:00:00');
   ```

3. **Query Data**:
   ```sql
   SELECT * FROM iceberg.default.sample_table;
   ```

4. **Process Data with Spark**:
   Run a Spark job to process the data:
   ```bash
   docker exec -it spark /opt/bitnami/spark/bin/spark-shell \
       --conf spark.hadoop.fs.s3a.access.key=minioadmin \
       --conf spark.hadoop.fs.s3a.secret.key=minioadmin \
       --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000
   
   val df = spark.read.format("iceberg").load("nessie.default.sample_table")
   df.show()
   ```

---

## Notes

1. **Custom Credentials**: Replace `minioadmin:minioadmin` with secure credentials in production.
2. **Logs**: Check logs for troubleshooting using:
   ```bash
   docker logs <service-name>
   ```
3. **Cleanup**: Stop and remove all containers:
   ```bash
   docker-compose down
   ```

This setup enables you to test a fully functional lakehouse engine integrating MinIO, Nessie, Iceberg, Trino, Spark, and Taipy.
