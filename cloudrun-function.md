# GCP Cloud Run Function — Cloud Storage Metadata Update Guide

## Overview

This guide walks through creating a **Cloud Run Function** triggered by **Cloud Storage** events to automatically update file metadata stored in Firestore.

---

## Step 1: Navigate to Cloud Run

1. Open [Google Cloud Console](https://console.cloud.google.com)
2. In the left sidebar, click **Cloud Run**
3. Click **"Write a Function"**
4. Select **Inline Function** as the editor type

---

## Step 2: Add a Trigger

1. Click **"Add Trigger"**
2. Select **Cloud Storage** as the trigger type
3. If prompted, click **"Enable API"** to activate required services
4. Set **Trigger Type** → `Cloud Storage`
5. Click **"Enable All"** to enable all required APIs
6. Select the appropriate **Event Type**:

| Event Type | Description |
|---|---|
| `google.cloud.storage.object.v1.finalized` | Triggered when a file upload completes |
| `google.cloud.storage.object.v1.deleted` | Triggered when a file is deleted |
| `google.cloud.storage.object.v1.archived` | Triggered when a file is archived |
| `google.cloud.storage.object.v1.metadataUpdated` | Triggered when metadata changes |

7. Click **"Save Trigger"**

---

## Step 3: Authentication

- Select the appropriate **service account** or use the default compute service account
- Ensure the account has the following IAM roles:
  - `roles/storage.objectAdmin` — to read/write object metadata
  - `roles/datastore.user` — to read from Firestore
- Click **"Create"**

> ⚠️ **Common Error:** If you see a permission error on creation, verify that the service account has been granted the above roles in **IAM & Admin**.

---

## Step 4: Function Code (Corrected)

The original code used `functions.firestore.document().onUpdate()` which is a **Firebase Functions v1 pattern** — incompatible with a Cloud Run Function triggered by Cloud Storage.

### ✅ Corrected `index.js` — Cloud Storage Trigger

```javascript
const { CloudEvent } = require("cloudevents");
const { Storage } = require("@google-cloud/storage");
const { Firestore } = require("@google-cloud/firestore");

const storage = new Storage();
const firestore = new Firestore();

/**
 * Cloud Run Function triggered by Cloud Storage finalize event.
 * Reads metadata config from Firestore and applies it to the uploaded file.
 *
 * @param {CloudEvent} cloudEvent - The Cloud Storage event payload
 */
exports.updateStorageMetadata = async (cloudEvent) => {
  const data = cloudEvent.data;

  if (!data || !data.bucket || !data.name) {
    console.error("Invalid event payload — missing bucket or file name.");
    return;
  }

  const bucketName = data.bucket;
  const fileName = data.name;

  console.log(`Processing file: gs://${bucketName}/${fileName}`);

  try {
    // Fetch metadata config from Firestore using the file name as document ID
    const docRef = firestore.collection("files_metadata").doc(fileName);
    const docSnap = await docRef.get();

    if (!docSnap.exists) {
      console.warn(`No Firestore metadata entry found for: ${fileName}`);
      return;
    }

    const { metadata } = docSnap.data();

    if (!metadata || typeof metadata !== "object") {
      console.error(`Invalid or missing 'metadata' field in Firestore for: ${fileName}`);
      return;
    }

    // Apply custom metadata to the GCS object
    const file = storage.bucket(bucketName).file(fileName);
    await file.setMetadata({ metadata });

    console.log(`✅ Successfully updated metadata for: gs://${bucketName}/${fileName}`);
  } catch (error) {
    console.error(`❌ Failed to update metadata for ${fileName}: ${error.message}`);
    throw error; // Re-throw so Cloud Run marks the invocation as failed
  }
};
```

---

## Step 5: `package.json`

```json
{
  "name": "update-storage-metadata",
  "version": "1.0.0",
  "description": "Cloud Run Function to update GCS object metadata from Firestore",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/firestore": "^7.0.0",
    "@google-cloud/storage": "^7.0.0",
    "cloudevents": "^6.0.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

---

## Key Differences: Original vs Corrected Code

| Aspect | ❌ Original (Broken) | ✅ Corrected |
|---|---|---|
| Trigger type | Firestore `onUpdate()` | Cloud Storage CloudEvent |
| SDK used | `firebase-functions` | `@google-cloud/storage` + `@google-cloud/firestore` |
| Runtime | Firebase Functions v1 | Cloud Run (Node 18+) |
| Event input | Firestore change object | `cloudEvent.data` (GCS object info) |
| Error handling | Silent `return null` | Re-throws error for proper failure signaling |
| Firestore lookup | Triggered by Firestore | Looks up Firestore using file name |

---

## Firestore Document Structure

Your `files_metadata` collection documents should follow this schema:

```json
{
  "metadata": {
    "author": "John Doe",
    "department": "Engineering",
    "project": "Alpha",
    "reviewed": "true"
  }
}
```

- **Collection:** `files_metadata`
- **Document ID:** must match the **GCS file name** (e.g., `reports/q1.pdf`)

---

## Deployment via CLI (Alternative to Console)

```bash
gcloud functions deploy updateStorageMetadata \
  --gen2 \
  --runtime=nodejs18 \
  --region=us-central1 \
  --source=. \
  --entry-point=updateStorageMetadata \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=YOUR_BUCKET_NAME" \
  --service-account=YOUR_SERVICE_ACCOUNT@YOUR_PROJECT.iam.gserviceaccount.com
```

Replace:
- `YOUR_BUCKET_NAME` — the GCS bucket to watch
- `YOUR_SERVICE_ACCOUNT` — the service account with required permissions
- `YOUR_PROJECT` — your GCP project ID

---

## Troubleshooting

| Error | Likely Cause | Fix |
|---|---|---|
| `403 Permission Denied` | Service account lacks IAM roles | Add `storage.objectAdmin` and `datastore.user` |
| `Function not triggering` | API not enabled | Enable `cloudfunctions.googleapis.com` and `eventarc.googleapis.com` |
| `No Firestore entry found` | Document ID mismatch | Ensure Firestore doc ID matches the GCS file path exactly |
| `Invalid event payload` | Wrong trigger event type | Use `google.cloud.storage.object.v1.finalized` |
| `firebase-functions not found` | Wrong SDK for Cloud Run | Replace with `@google-cloud/firestore` |

---

*Generated for GCP Cloud Run Functions (2nd Gen) — Node.js 18+*
