# Moving data between Snowflake and Databricks

Only the small handoff tables listed in `docs/data_contract.md` move between platforms. Keep table and column names exactly as in the contract.

Try **Option A** first. If Databricks can't reach Snowflake (timeout, unknown host, connection refused), use **Option B**.

---

## Option A: Direct connection

### Snowflake → Databricks (Lakehouse Federation)

1. Databricks: **Catalog → External data → Connections → Create connection**, type **Snowflake**.
2. Enter the Snowflake account URL, warehouse and the Databricks user's login.
3. Create a **foreign catalog** from the connection, pointing at the `NFL` database.
4. Query Snowflake tables directly:

```python
df = spark.table("snowflake_nfl.public.wr_outcomes")
```

### Databricks → Snowflake (Snowflake connector)

```python
sf = {
    "sfUrl":       "<account_identifier>.snowflakecomputing.com",
    "sfUser":      dbutils.secrets.get("snowflake", "user"),
    "sfPassword":  dbutils.secrets.get("snowflake", "password"),
    "sfDatabase":  "NFL",
    "sfSchema":    "PUBLIC",
    "sfWarehouse": "COMPUTE_WH",
}

(spark.table("workspace.nfl.player_scores")
   .write.format("snowflake").options(**sf)
   .option("dbtable", "PLAYER_SCORES")
   .mode("overwrite").save())
```

The same options with `spark.read.format("snowflake")` also work for reading.

After each new Snowflake trial, update the account URL and login.

---

## Option B: Download and upload (always works)

### Snowflake → Databricks

1. Snowsight: run `SELECT * FROM <table>;` → **Download → CSV**.
2. Databricks: **Catalog → workspace → nfl → raw** → **Upload to this volume**.
3. Load it:

```python
name = "wr_outcomes"
(spark.read.csv(f"/Volumes/workspace/nfl/raw/{name}.csv",
                header=True, inferSchema=True, nullValue="NA")
   .write.mode("overwrite").option("overwriteSchema", "true")
   .saveAsTable(f"workspace.nfl.{name}"))
```

### Databricks → Snowflake

1. Databricks: write the table as one CSV file into the volume:

```python
name = "player_scores"
(spark.table(f"workspace.nfl.{name}").coalesce(1)
   .write.mode("overwrite").option("header", True)
   .csv(f"/Volumes/workspace/nfl/raw/export_{name}"))
```

2. Open the `export_<name>` folder in the volume and download the `part-….csv` file.
3. Snowsight: **Data → Add data → Load data into a table**, choose the file, and load it into the table named in the contract.

---

## Rules

- Never commit passwords. Use Databricks secrets or the connection's stored login.
- Never share or commit the raw competition data. Only contract tables move.
- If a column changes, update `docs/data_contract.md` in a pull request first.
