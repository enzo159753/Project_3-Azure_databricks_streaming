# Azure Data Pipeline Project — Event Hubs & Databricks Streaming

---

### Context & Approach
This project is split into two parts: a **lift-and-shift migration** of historical bike ride data into Azure (Part 1), and a **real-time ingestion pipeline** where a simulated web application generates ride booking events, publishes them to **Azure Event Hubs**, and Databricks consumes them via a **Spark Declarative Pipeline** (Part 2). 
```
GitHub (mock API) ──► ADF Pipeline ──► ADLS Gen2          (Part 1 — Batch)
Web App (simulator) ──► Azure Event Hubs ──► Databricks DLT  (Part 2 — Streaming)
```

### Architecture

<img width="1711" height="788" alt="image" src="https://github.com/user-attachments/assets/0e7370c3-ae21-453f-b203-d5b30bf623d5" />

### Data Flow

<img width="1340" height="835" alt="image" src="https://github.com/user-attachments/assets/e16e23e8-03a7-4583-a712-b3b9fba52447" />

---

## Key Concepts

| Concept | Description |
|---|---|
| **Azure Event Hubs** | Managed event streaming service acting as the message broker for real-time ride booking events |
| **Streaming Table** | A table that continuously ingests new records from the Event Hubs stream as they arrive |
| **SCD (Slowly Changing Dimensions)** | Pattern applied to dimension tables to track historical changes over time — typically used here to maintain an up-to-date view of entities such as riders or drivers while preserving change history |
| **Medallion Architecture** | Data is progressively refined across layers: **Bronze** (raw ingestion) → **Silver** (cleaned & structured) → **Gold** (aggregated & business-ready) |

---

## Part 1 Azure Data Lake Ingestion Pipeline

### Step 1 — Create a Storage Account

Provision an **Azure Storage Account** with the hierarchical namespace option enabled,
which configures it as an **Azure Data Lake Storage Gen2 (ADLS Gen2)** account.

### Step 2 — Configure Linked Services in ADF

In **Azure Data Factory (ADF)**, create two Linked Services under the Manage tab:

| Linked Service | Target |
|---|---|
| **ADLS Gen2** | The newly provisioned Data Lake Storage account |
| **HTTP** | The GitHub repository folder hosting the historical files |

---

### Pipeline Design

A **JSON configuration file** stored in the `raw` folder of the storage account holds the
list of files to be ingested. This file acts as the dynamic driver of the pipeline.

<img width="522" height="161" alt="image" src="https://github.com/user-attachments/assets/9e0c7413-9370-494f-ad77-b892d3fbe795" />

<img width="729" height="484" alt="image" src="https://github.com/user-attachments/assets/d0caad84-2e3f-49a4-b601-47cf4c121b5b" />

The pipeline is composed of three sequential activities:

```
Lookup Activity ──► ForEach Activity ──► Copy Data Activity
(reads JSON config)  (iterates files)    (HTTP → ADLS Gen2)
```

<img width="945" height="356" alt="image" src="https://github.com/user-attachments/assets/00a3284d-3533-49b5-8b81-5588bddaeebf" />

#### Activity Breakdown

**1. Lookup Activity — Files Array**
Reads the JSON configuration file via an ADF Dataset pointing to the `raw` folder.
Returns an array of file metadata objects used to drive the rest of the pipeline.

**2. ForEach Activity**
Iterates over each element in the array returned by the Lookup activity.

**3. Copy Data Activity** *(inside ForEach)*
For each file in the array, fetches the corresponding file from GitHub via the
HTTP Linked Service and copies it into the `ingestion` folder in ADLS Gen2.

---

## Part 2 — Real-Time Ride Booking Ingestion with Azure Event Hubs & Databricks

1. Create an **Event Hubs Namespace** (`ubereventsE`) in the Azure portal.
2. Inside the namespace, create an **Event Hub topic** (`ubertopic`).
3. Generate a **SAS token** using the `Listen` policy — this token is used by
   Databricks to authenticate and read events from the hub.

| Resource | Name |
|---|---|
| Namespace | `ubereventsE` |
| Topic | `ubertopic` |
| Auth Policy | `Listen` (SAS token) |

App  data sending events to this Azure EventHub :
<img width="2521" height="1132" alt="image" src="https://github.com/user-attachments/assets/6dc854ee-654d-4a93-86e2-9d4c4e250f7e" />

Setup a listen policy for Databricks :
<img width="1625" height="900" alt="image" src="https://github.com/user-attachments/assets/066bddf9-7c4d-42e1-8a19-7835cf54930a" />
