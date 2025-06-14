# azure-billing-archival
Cost-optimized archival architecture for Azure Cosmos DB billing records.


## 📌 Overview
This repository provides a **cost-effective**, **scalable**, and **seamless** data archival solution for Azure Cosmos DB used in a serverless billing system. The goal is to reduce operational costs by offloading infrequently accessed records (>3 months old) to Azure Blob Storage without impacting existing APIs or causing downtime.

---

## 🧠 Problem Statement
- **Current Storage**: Azure Cosmos DB (Read-heavy, 2M+ records, 300KB per record)
- **Issue**: Growing cost due to large volume of data
- **Constraint**: No API contract changes, no data loss, no downtime

---

                     +-----------------------+
                     |   Azure Functions     | <------- Client/API
                     |  (Read & Write API)   |
                     +----------+------------+
                                |
                                v
                +-----------------------------+
                |        Cosmos DB            | <-------- New Billing Records (<3 months)
                +-----------------------------+
                                |
                                | Archive (>3 months)
                                v
      +-----------------------------------------------+
      |         Azure Data Lake Gen2 / Blob Storage   | <--- Archived Billing Records (JSON)
      +-----------------------------------------------+
                                ^
                                |
        +---------------------------------------------+
        |     Azure Function (Archival Trigger)       |
        | Time Triggered (e.g., Daily)                |
        +---------------------------------------------+

| Feature                    | Approach                                                                  |
| -------------------------- | ------------------------------------------------------------------------- |
| **Hot Data (<3 months)**   | Stored in Cosmos DB for fast access (read-heavy).                         |
| **Cold Data (>3 months)**  | Moved to Azure Data Lake / Blob Storage in JSON format to reduce cost.    |
| **Unified Read API**       | Read logic checks Cosmos DB first → falls back to Data Lake if not found. |
| **No Downtime**            | Archival done asynchronously; existing data is untouched until archived.  |
| **No API Contract Change** | Same API; logic inside read handler changes to support archive fallback.  |

### Key Principles
| Layer          | Description                                                      |
|----------------|------------------------------------------------------------------|
| Cosmos DB      | Holds latest billing data (<= 3 months)                          |
| Blob Storage   | Holds older billing records in JSON format                       |
| Azure Function | Timer-based archival job + Read API fallback to Blob Storage     |

---

## 💡 Benefits
- **Reduced Cost**: Offloading to Blob can reduce storage cost by ~90%
- **No Data Loss**: Archival after successful upload
- **Same APIs**: Clients continue to use existing endpoints
- **No Downtime**: Transition handled asynchronously

---

## 📂 Folder Structure
```
azure_billing_archival/
├── archive_function/
│   └── archive_old_records.py
├── api_handler/
│   └── read_billing_record.py
├── assets/
│   └── architecture.png
├── README.md
```

---

## 🔁 Archival Function (Python Azure Function)
```python
# archive_old_records.py
from datetime import datetime, timedelta
import json

def archive_old_records():
    cutoff_date = datetime.utcnow() - timedelta(days=90)
    records = cosmos.query(f"SELECT * FROM c WHERE c.timestamp < '{cutoff_date.isoformat()}'")
    
    for record in records:
        record_id = record['id']
        blob_path = f"archive/{record_id}.json"
        blob_storage.upload(blob_path, json.dumps(record))
        cosmos.delete(record_id)
```

---

## 📥 Read Handler (API Function)
```python
# read_billing_record.py

def get_billing_record(record_id):
    try:
        return cosmos.read(record_id)
    except NotFoundError:
        blob_path = f"archive/{record_id}.json"
        if blob_storage.exists(blob_path):
            return blob_storage.read_json(blob_path)
        else:
            raise RecordNotFound()
```

---

## ⚠️ Edge Cases Handled
- ✅ Partial migration: Use `status` flags or separate metadata table
- ✅ Inconsistent reads: Only delete after blob upload confirmation
- ✅ On-demand restore (optional): Admin function to restore archived records

---

## 🚀 Deployment Steps
1. Deploy Azure Blob Storage with hot/cool/archive tier
2. Deploy Cosmos DB with indexing policy for recent items
3. Deploy two Azure Functions:
   - Timer Trigger: Archival Function
   - HTTP Trigger: Read API fallback logic

---

## 📦 Bonus
- Enable Blob lifecycle policy to auto-tier blobs (cool → archive)
- Use Change Feed instead of Timer Trigger for real-time archival

---
