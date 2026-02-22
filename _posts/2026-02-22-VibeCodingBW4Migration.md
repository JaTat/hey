---
layout: post
title: Vibe Coding a BW/4 Migration to a Spark Lakehouse with Delta Tables
description: >
  Using vibe coding to extract transformation logic from SAP BW/4 system tables and rebuild it on a Spark Lakehouse with Delta Tables
grouped: true
---

Hey there! If you've worked with SAP BW/4 you know the pain. The platform is powerful but proprietary, expensive to run, and increasingly hard to staff. Meanwhile the data engineering world has moved on to open formats and cloud-native compute. So what if we could use an AI coding assistant to help us migrate a BW/4 data model to a Spark Lakehouse with Delta Tables?

That's exactly what I've been experimenting with. In this post I'll walk through the approach: reading out the transformation and storage logic from BW system tables, understanding the key differences between BW objects and Lakehouse concepts, and using vibe coding to generate the Spark migration code.

## What is Vibe Coding?

Vibe coding is a term coined by Andrej Karpathy. The idea is simple: instead of writing every line yourself, you describe what you want in natural language and let an AI coding assistant (like Cursor, GitHub Copilot, or Claude) generate the code. You stay in the driver's seat, guiding the direction, reviewing outputs, and iterating. For a migration like this it's a game changer because a lot of the work is repetitive: read a BW transformation, understand it, rewrite it in PySpark. Perfect for an AI pair programmer.

The key insight: if we can extract the transformation logic from BW in a structured way, we can feed it to an AI assistant and let it generate equivalent PySpark code. The BW system tables give us exactly that.

## BW/4 Data Model Recap: Tables, ADSOs, and InfoObjects

Before we migrate anything we need to understand what we're migrating. BW/4 has its own vocabulary and abstraction layers that don't map 1:1 to a Lakehouse.

### InfoObjects: The Building Blocks

InfoObjects are BW's version of dimensions and measures. They are reusable across the entire data model which is one of BW's strengths. An InfoObject like `0CUSTOMER` or `ZMATERIAL` is defined once and used in many ADSOs. Each InfoObject carries:

- Master data (attributes, texts, hierarchies)
- A surrogate ID (SID) for performance
- Type information (characteristic vs. key figure)

This reusability is powerful but also means migrating a single InfoObject can affect many downstream objects.

### The Star Schema Abstraction

BW abstracts the classic star schema in an interesting way. Instead of explicit fact and dimension tables that you'd design yourself, BW generates them under the hood:

- **Fact tables** store the transactional data with SID keys pointing to dimensions
- **Dimension tables** are auto-generated groupings of InfoObjects (characteristics)
- **SID tables** (`/BI0/S<InfoObject>`) map business keys to surrogate IDs
- **Master data tables** (`/BI0/P<InfoObject>` for attributes, `/BI0/T<InfoObject>` for texts)

You don't design the star schema yourself, BW does it for you based on the InfoObjects you assign to an ADSO. This is convenient but it also means the physical schema is hidden from you. When migrating to a Lakehouse we need to decide: do we replicate this star schema or denormalize?

For most Lakehouse migrations I'd argue: **denormalize**. Delta Tables handle wide tables efficiently, and the query engines (Spark, Databricks SQL) don't need SID-based joins to perform well. Flatten the star schema into wide fact tables with business keys and descriptive attributes.

### ADSOs: The Main Persistence Layer

The Advanced DataStore Object (ADSO) is the primary data storage object in BW/4. It replaced the older DSO and InfoCube concepts. An ADSO can be configured in different flavors:

- **Standard (write-optimized):** Incoming data lands in an inbound table, gets activated into the active data table, and changes are tracked in the change log
- **Direct update:** Data is written directly without activation
- **Inventory / planning variants**

The critical thing for migration is understanding the **three-table architecture** of a standard ADSO:

| BW Table | Purpose | Lakehouse Equivalent |
|----------|---------|---------------------|
| Inbound table | Staging area for incoming loads | Bronze / staging layer |
| Active data table | Current truth after activation | Silver / curated layer |
| Change log | Delta records for downstream | Delta Table change data feed |

### The Change Log: Incremental Loading

This is where it gets really interesting. The ADSO change log is BW's mechanism for incremental (delta) loading. Every time data is activated in an ADSO, BW computes the difference between old and new records and writes it to the change log. Downstream objects can then pull only the changes instead of doing a full reload.

Each change log record has a `RECORDMODE` field:

| RECORDMODE | Meaning |
|------------|---------|
| '' (blank) | New image (after image) |
| 'X' | Before image (old record before change) |
| 'R' | Reverse image (deletion) |
| 'D' | Delete |
| 'N' | New (insert) |

This maps beautifully to Delta Table's **Change Data Feed (CDF)**. When you enable CDF on a Delta Table, it tracks `_change_type` with values like `insert`, `update_preimage`, `update_postimage`, and `delete`. The concepts are almost identical. So during migration we can:

1. Do an initial full load from the active data table
2. Switch to incremental loads using the change log
3. On the Lakehouse side, use Delta Table MERGE operations to apply the changes

## Reading Transformation Logic from BW System Tables

Now for the fun part: extracting what BW actually does so we can replicate it. The transformation logic lives in system tables that we already explored in my earlier post about [analyzing the BW data dictionary in Python](/2021-08-30-PythonToBW/).

The key tables for extracting transformation metadata are:

| Table | Content |
|-------|---------|
| `RSTRAN` | Transformation header: source, target, type |
| `RSTRANFIELD` | Field-level mapping between source and target |
| `RSTRANRULE` | Transformation rules (direct assignment, formula, routine) |
| `RSTRANSTEPROUT` | ABAP routines embedded in transformations |
| `RSOOBJXREF` | Cross-reference of object dependencies |
| `RSDIOBJT` | InfoObject descriptions |
| `RSOADSO` | ADSO directory |

Here's how to extract the field mappings for all transformations:

~~~python
query = """
SELECT t.TRANID, t.SOURCETYPE, t.SOURCENAME, t.TARGETTYPE, t.TARGETNAME,
       f.FIELDNAME, f.SOURCEFIELD, f.RULETYPE
FROM RSTRAN t
JOIN RSTRANFIELD f ON t.TRANID = f.TRANID
WHERE t.OBJVERS = 'A'
ORDER BY t.TRANID
"""
df_trans = pd.read_sql_query(query, conn)
df_trans.head(10)
~~~

The `RULETYPE` column tells you how each field is mapped:

| RULETYPE | Meaning |
|----------|---------|
| `DIR` | Direct assignment (1:1 mapping) |
| `FOR` | Formula |
| `ROU` | ABAP routine |
| `CON` | Constant |
| `INI` | Initial value |

For direct mappings and formulas the migration is straightforward. The AI assistant can generate the equivalent PySpark transformation. ABAP routines are trickier since they contain custom logic, but we can extract the source code from `RSTRANSTEPROUT` and feed it to the AI to translate.

## The Vibe Coding Workflow

Here's the workflow I've been using:

### 1. Extract the metadata

Pull the transformation definitions, field mappings, and ABAP routine source code from the system tables into structured files (CSV, JSON, or just DataFrames).

### 2. Feed it to the AI assistant

For each transformation, I provide the AI with:
- Source and target object definitions (fields, data types)
- The field mapping with rule types
- Any ABAP routine source code
- Sample data if available

### 3. Generate PySpark code

The prompt looks something like this:

~~~
Here is a BW transformation that loads from ADSO ZO_SALES to ADSO ZO_SALESC:

Field mappings:
- ZCUSTOMER -> ZCUSTOMER (direct)
- ZMATERIAL -> ZMATERIAL (direct)  
- ZREVENUE -> ZREVENUE (direct)
- ZCALMONTH -> ZCALMONTH (formula: left 6 chars of ZCALDAY)
- ZMARGIN -> (routine: ZREVENUE - ZCOST)

Generate a PySpark transformation using Delta Tables.
Include schema definition, read from source Delta Table, 
apply transformations, and merge into target Delta Table.
~~~

The AI generates something like:

~~~python
from pyspark.sql import functions as F
from delta.tables import DeltaTable

# Read source
df_source = spark.read.format("delta").load("/lakehouse/silver/zo_sales")

# Apply transformations  
df_transformed = (df_source
    .withColumn("ZCALMONTH", F.substring("ZCALDAY", 1, 6))
    .withColumn("ZMARGIN", F.col("ZREVENUE") - F.col("ZCOST"))
    .select("ZCUSTOMER", "ZMATERIAL", "ZREVENUE", "ZCALMONTH", "ZMARGIN")
)

# Merge into target (upsert)
target = DeltaTable.forPath(spark, "/lakehouse/silver/zo_salesc")
target.alias("t").merge(
    df_transformed.alias("s"),
    "t.ZCUSTOMER = s.ZCUSTOMER AND t.ZMATERIAL = s.ZMATERIAL AND t.ZCALMONTH = s.ZCALMONTH"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
~~~

### 4. Review and iterate

This is where the "vibe" part is important. You don't blindly accept the output. You review it, test it against sample data, and iterate. The AI handles the boilerplate; you handle the domain knowledge.

### 5. Handle the tricky parts

Some things need human judgment:
- **ABAP routines with complex logic:** The AI can translate most ABAP to PySpark but edge cases around BW-specific function modules need careful review
- **SID resolution:** Decide whether to keep surrogate keys or resolve to business keys
- **Error handling and data quality:** BW has its own error stack; you need equivalent logic in Spark
- **Incremental loading setup:** Configure Delta Table CDF and the merge logic to match BW's change log semantics

## Mapping BW Concepts to Lakehouse

Here's a summary of how the key concepts translate:

| BW/4 Concept | Lakehouse Equivalent |
|--------------|---------------------|
| InfoObject (characteristic) | Dimension column or lookup table |
| InfoObject (key figure) | Measure column |
| ADSO inbound table | Bronze / raw layer Delta Table |
| ADSO active data table | Silver / curated Delta Table |
| ADSO change log | Delta Table Change Data Feed |
| Transformation (direct) | PySpark select / withColumn |
| Transformation (formula) | PySpark expression |
| Transformation (routine) | PySpark UDF or inline logic |
| DTP (Data Transfer Process) | Spark job / orchestration (e.g. Airflow, Fabric Pipeline) |
| Process Chain | Orchestration DAG |
| Star schema (auto-generated) | Denormalized wide table or explicit star schema |
| SID tables | Not needed (use business keys directly) |
| BEx Query | SQL view / Spark SQL / BI tool semantic model |

## Conclusion

Migrating from BW/4 to a Spark Lakehouse is not trivial, but vibe coding makes it significantly more approachable. The system tables in BW contain everything you need to understand the data model and transformation logic. By extracting this metadata and feeding it to an AI coding assistant, you can generate the bulk of the PySpark migration code and focus your energy on the parts that actually need human judgment: data quality rules, business logic validation, and incremental loading semantics.

The change log to Delta CDF mapping is particularly elegant. BW was ahead of its time with built-in change data capture, and Delta Tables offer essentially the same capability with an open format.

If you're facing a BW migration, I'd suggest starting small: pick one data flow (source → transformation → ADSO), extract the metadata, and try the vibe coding approach. You might be surprised how far you get in an afternoon.

--------

###### This post reflects my personal experience and experiments
