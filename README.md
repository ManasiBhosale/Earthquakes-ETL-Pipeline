<h1 style="display: flex; align-items: center; gap: 10px; margin: 0;">
  Earthquakes ETL Pipeline
  <img
    src="https://github.com/ManasiBhosale/Earthquakes-ETL-Pipeline/blob/c769ecf44f7ea3065417133133cf349e68b2cc7a/images/seismograph.gif"
    alt="Seismograph GIF"
    width="80"
    align="absmiddle"
    style="position: relative; top: -30px;"
  />
</h1>



#### *End-to-End Data Engineering | Medallion Architecture | Data Factory | Power BI*
<br>

This project demonstrates a complete **data engineering solution** on **Microsoft Fabric**, ingesting real-time earthquake event data from the **USGS Earthquake API**, processing it through **Medallion architecture (Bronze → Silver → Gold)** using **Synapse Data Engineering**, orchestrating the workflow with **Data Factory**, and visualizing global seismic activity in **Power BI**.

---

## 📡 **Overview**

* **Source:** USGS Earthquake API
  API documentation: [https://earthquake.usgs.gov/fdsnws/event/1/#parameters](https://earthquake.usgs.gov/fdsnws/event/1/#parameters)
* **Goal:** Build an automated, scalable data pipeline that ingests daily earthquake events, transforms and enriches them, and powers a dynamic Power BI report.
* **Stack:**

  * Microsoft Fabric Lakehouse
  * Synapse Data Engineering (PySpark notebooks)
  * Data Factory pipelines
  * Gold semantic model
  * Power BI analytics

---

## 🏗️ **Architecture**

```text
USGS Earthquake API
        │   (Python requests)
        ▼
[ BRONZE ] → Raw Ingestion (JSON Files)
        │
        ▼   (PySpark Transformations)
[ SILVER ] → Cleaned & Flattened Delta Table
        │
        ▼   (Reverse Geocoding + Business Logic)
[ GOLD ] → Enriched, Business-Ready Delta Table
        │
        ▼
Semantic Model
        │
        ▼
Power BI Reports (Daily Auto-Refresh)
        │
        ▼
Data Factory Pipeline (Orchestrating Everything)
```

This design follows the **Medallion architecture** pattern, ensuring scalable, modular data transformation layers.

---

# 🥉 **Bronze Layer — Raw Ingestion**

The Bronze notebook ingests raw earthquake events directly from the USGS API.

### **Key Responsibilities**

* Parameterized API ingestion
* Raw JSON extraction (`features` array)
* Storage into Lakehouse Files
* Acts as the *landing zone* for incremental loads

### **API Format Example**

```
https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&starttime=2014-01-01&endtime=2014-01-02
```

### **Core Logic**

```python
url = f"https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&starttime={start_date}&endtime={end_date}"
response = requests.get(url)

if response.status_code == 200:
    data = response.json()['features']
    file_path = f'/lakehouse/default/Files/{start_date}_earthquake_data.json'
    with open(file_path, 'w') as file:
        json.dump(data, file, indent=4)
```

### **Notes**

* During development, start/end dates were hardcoded for testing 7-day loads.
* In production, **Data Factory** passes these parameters dynamically.

---

# 🥈 **Silver Layer — Standardized & Flattened Data**

The Silver notebook shapes the raw JSON into a structured table.

### **Key Responsibilities**

* Flatten nested geometry and property fields
* Extract latitude/longitude/elevation
* Convert Unix milliseconds → timestamps
* Persist cleaned dataset to a Silver Delta table

### **Core Logic**

```python
df = spark.read.option("multiline", "true").json(f"Files/{start_date}_earthquake_data.json")

df = df.select(
    'id',
    col('geometry.coordinates').getItem(0).alias('longitude'),
    col('geometry.coordinates').getItem(1).alias('latitude'),
    col('geometry.coordinates').getItem(2).alias('elevation'),
    col('properties.title').alias('title'),
    col('properties.place').alias('place_description'),
    col('properties.sig').alias('sig'),
    col('properties.mag').alias('mag'),
    col('properties.magType').alias('magType'),
    col('properties.time').alias('time'),
    col('properties.updated').alias('updated')
).withColumn("time", (col("time")/1000).cast("timestamp"))\
 .withColumn("updated", (col("updated")/1000).cast("timestamp"))

df.write.mode('append').saveAsTable('earthquake_events_silver')
```

### **Notes**

* Parameterized `start_date` supports incremental processing.
* Silver provides a **clean, analytics-ready schema**.

---

# 🥇 **Gold Layer — Business-Ready Enrichment**

The Gold notebook enriches Silver data with **reverse geocoding** and **classification logic**.

### **Key Responsibilities**

* Lookup country codes from latitude/longitude
* Add `sig_class` categories: Low / Moderate / High
* Append enriched rows to Gold Delta table

### **Core Logic**

```python
def get_country_code(lat, lon):
    coordinates = (float(lat), float(lon))
    return rg.search(coordinates)[0]['cc']

get_country_code_udf = udf(get_country_code, StringType())

df = spark.read.table("earthquake_events_silver") \
               .filter(col('time') > start_date)

df_with_location = df.withColumn(
    "country_code",
    get_country_code_udf(col("latitude"), col("longitude"))
)

df_final = df_with_location.withColumn(
    "sig_class",
    when(col("sig") < 100, "Low")
    .when((col("sig") >= 100) & (col("sig") < 500), "Moderate")
    .otherwise("High")
)

df_final.write.mode('append').saveAsTable('earthquake_events_gold')
```

### **Notes**

* Gold is the **semantic layer**, providing business-friendly fields.
* The table powers the Power BI model.

---

# 📊 **Power BI — Earthquake Monitoring Dashboard**

A dedicated **Fabric Semantic Model** is built on top of the Gold table.

### **Visuals**

**Map View**

* **Location** → `country_code`
* **Legend** → `sig_class`
* **Bubble Size** → Max Significance (`sig`)
* **Tooltip** → Count of earthquakes

**Slicers**

1. Date range
2. Significance class (Low / Moderate / High)

### **Outcome**

The dashboard auto-refreshes daily as the pipeline loads new data, providing a live global earthquake activity monitor.

---

# 🔄 **Data Factory Pipeline — Orchestration**

A Fabric Data Factory pipeline orchestrates the 3 notebook layers:

```
Bronze Notebook
      ↓
Silver Notebook
      ↓
Gold Notebook
```

### **Dynamic Parameters**

```text
start_date = @formatDateTime(adddays(utcNow(), -1), 'yyyy-MM-dd')
end_date   = @formatDateTime(utcNow(), 'yyyy-MM-dd')
```

### **Pipeline Behavior**

* Runs daily
* Fetches previous day’s earthquake events
* Appends to Bronze → Silver → Gold
* Power BI visuals automatically update

### **Example**

On **25/03**, initial load using:

```
start=2026-03-18, end=2026-03-24
```

Pipeline run on **26/03** automatically ingests **25/03** data.

---

# 🚀 **End-to-End Outcome**

This project showcases a complete Fabric-based data engineering solution:

* Automated ingestion using **parameterized API calls**
* Transformations using **PySpark notebooks**
* Medallion architecture ensuring data quality and lineage
* Reverse geocoding and classification for enriched analytics
* Dynamic Power BI reporting
* A fully orchestrated **daily refresh pipeline**

---

# 📌 **Future Enhancements**

* Add anomaly detection for unusual seismic patterns
* Expand enrichment using tectonic plate boundaries
* Create alerting using Fabric Real-Time Hub
* Add unit tests for transformations
* Integrate monitoring through Data Factory activity logs

---

