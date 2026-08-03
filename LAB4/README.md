# CST8917 - Lab 4: PhotoPipe Event-Driven Image Processing with Event Grid & Functions

Event-driven image processing pipeline using Azure Event Grid, Functions, and Blob Storage.

## Video Demo
https://drive.google.com/file/d/1YBuMtXfZ54nVoN6NkQZN77DWlzvIgzhs/view?usp=sharing

## Part 1: Storage Account

1. Create resource group `rg-serverless-lab4` in Canada Central
2. Create a storage account (e.g. `yournamephotopipe`): Standard, LRS
3. Under **Configuration**, enable **Allow Blob anonymous access**
4. Create two containers:

   | Container       | Access                |
   | --------------- | --------------------- |
   | `image-uploads` | Blob (anonymous read) |
   | `image-results` | Private               |

5. Under **Resource sharing (CORS)**; Blob service, add:

   | Field                   | Value                     |
   | ----------------------- | ------------------------- |
   | Allowed origins         | `*`                       |
   | Allowed methods         | `GET, PUT, OPTIONS, HEAD` |
   | Allowed/Exposed headers | `*`                       |
   | Max age                 | `3600`                    |

---

## Part 2: Azure Functions

### Local Setup

1. Create a new Functions project (Python, skip template)
2. Copy `function_app.py`, `test-function.http`, and `client.html` from lab materials
3. Ensure `requirements.txt` contains:
   ```
   azure-functions
   azure-storage-blob
   azure-data-tables
   ```
4. Create `local.settings.json`:
   ```json
   {
     "IsEncrypted": false,
     "Values": {
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "FUNCTIONS_WORKER_RUNTIME": "python",
       "STORAGE_CONNECTION_STRING": "<your-connection-string>"
     },
     "Host": { "CORS": "*" }
   }
   ```
5. Install dependencies and test HTTP functions locally (F5 + Azurite)

> Event Grid-triggered functions (`process-image`, `audit-log`) can only be tested end-to-end after deployment.

### Deploy to Azure

1. Create Function App: **Python 3.12**, Linux, Consumption plan, `rg-serverless-lab4`
2. Deploy via **Azure Functions: Deploy to Function App**
3. In the portal; Function App; **Environment variables**, add:

   | Name                        | Value                                  |
   | --------------------------- | -------------------------------------- |
   | `STORAGE_CONNECTION_STRING` | Your storage account connection string |

4. Under **CORS**, add `*`
5. Verify: `https://yourname-photopipe-func.azurewebsites.net/api/health`; should return `{"status": "healthy"}`

---

## Part 3: Event Grid

### Create System Topic

In the portal, search **Event Grid System Topics**; Create:

| Field      | Value                   |
| ---------- | ----------------------- |
| Topic Type | Storage Accounts        |
| Resource   | Your storage account    |
| Name       | `photopipe-blob-events` |

### Subscription 1: Image Processing

- **Name:** `process-image-sub`
- **Event type:** Blob Created
- **Endpoint:** `process-image` function
- **Filters tab:**
  - Subject begins with: `/blobServices/default/containers/image-uploads`
  - Advanced filter; Key: `subject`, Operator: `String ends with`, Values: `.jpg`, `.png`
  - Clear the "Subject Ends With" field (replaced by advanced filter)

### Subscription 2: Audit Log

- **Name:** `audit-log-sub`
- **Event type:** Blob Created
- **Endpoint:** `audit-log` function
- **Filters tab:**
  - Subject begins with: `/blobServices/default/containers/image-uploads`
  - No suffix filter

---

## Part 4: Web Client

1. Generate a SAS token: Storage account; **Shared access signature**
   - Services: Blob | Resource types: Container + Object | Permissions: Read, Write, Create, List
   - Expiry: 24 hours | Protocol: HTTPS only
   - Copy the token (starts with `?sv=`)

2. Open `client.html` and fill in:
   - Storage Account Name
   - SAS Token
   - Function App URL

---

## Part 5: Testing

| Test                         | Expected Result                                                        |
| ---------------------------- | ---------------------------------------------------------------------- |
| Upload `.jpg` or `.png`      | Both functions fire: result in **Results** tab, entry in **Audit Log** |
| Upload any other file format | Only audit-log fires: entry in **Audit Log**, nothing in **Results**   |
