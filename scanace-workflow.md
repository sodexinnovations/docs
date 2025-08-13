# ScanAce Integration API Documentation

## Overview

The ScanAce Integration API provides endpoints for managing projects and snapshots for point cloud data uploads. This API follows a specific workflow for uploading snapshot data to the platform.

## Base URL

**Production:** `https://api.sodex.cloud/scanace`

## API Documentation

Interactive API documentation is available at: `https://api.sodex.cloud/scanace/docs`

## Authentication

All endpoints require authentication. Authentication documentation is provided in a separate document.

## Data Flow: Snapshot Upload Workflow

The complete workflow for uploading a snapshot follows these steps:

### 1. Get User's Organization Projects
**Endpoint:** `GET /organizations/me/projects`

Retrieve all projects that the authenticated user is a member of.

**Response:**
```json
{
  "data": [
    {
      "id": "project_id", // ObjectID
      "name": "ScanAce Integration Project"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 1
  }
}
```

### 2. Create New Snapshot
**Endpoint:** `POST /projects/{project_id}/snapshots`

Create a new snapshot in the selected project. Only cloud-only projects are supported.

**Request Body:**
```json
{
  "name": "ScanAce Test 1",
  "location": { // optional - used for locating the snapshot in the SDX-Cloud. If not provided calculated by sodex
    "latitude": 51.1656,
    "longitude": 10.4515
  },
  "utm_zone_nr": 32,
  "utm_hemisphere": "N",
  "description": "This is a scan ace upload test", // optional
  "start_capture_at": "2025-01-22T10:00:00Z", // optional provide in UTC
  "end_capture_at": "2025-01-22T12:00:00Z", // optional
}
```

**Response:**
```json
{
  "data": {
    "id": "snapshot_id", // ObjectID
    "name": "ScanAce Test 1",
    "project_id": "project_id", // ObjectID
    "status": "created",
    "snapshot_source": "scanace",
    "created_at": "2025-01-22T10:00:00Z" // always in UTC
  }
}
```

### 3. Get S3 Upload Credentials
**Endpoint:** `GET /snapshots/{snapshot_id}/s3-upload-credentials`

Request temporary S3 credentials (valid for 4 hours) to upload snapshot data.

**Response:**
```json
{
  "data": {
    "access_key_id": "AKIA...",
    "access_key_secret": "secret_key",
    "session_token": "session_token",
    "region": "eu-central-1", // eu-central-1 or us-east-1. Not fully validated/tested on us-east-1 yet
    "bucket": "sodex-integrations",
    "s3_prefix": "scanace/..." // user only has access to this prefix
  }
}
```

### 4. Upload Snapshot Data
Using the S3 credentials obtained in step 3, upload your LAZ file to the specified S3 bucket and prefix.

**Example using boto3:**
```python
import boto3

s3 = boto3.client(
    "s3",
    aws_access_key_id=credentials.access_key_id,
    aws_secret_access_key=credentials.access_key_secret,
    aws_session_token=credentials.session_token,
    region_name=credentials.region,
)

s3_key = f"{credentials.s3_prefix}your_file.laz"
s3.put_object(
    Bucket=credentials.bucket,
    Key=s3_key,
    Body=open("local_file.laz", "rb"),
)
```

### 5. Commit Snapshot
**Endpoint:** `POST /snapshots/{snapshot_id}/commit`

Commit the snapshot with the uploaded LAZ file S3 key.

**Request Body:**
```json
{
  "laz_s3_key": "{provided-prefix}/your_file.laz"
}
```

**Response:**
```json
{
  "data": {
    "id": "snapshot_id", // ObjectID
    "name": "ScanAce Test 1", // should match basename of s3 key if possible but not required
    "status": "committed",
    "laz_s3_key": "scanace/snapshot_id/your_file.laz",
    "updated_at": "2025-01-22T12:00:00Z" // always in UTC
  }
}
```

## Available Endpoints

### Projects

#### Get Organization Projects
- **Method:** `GET`
- **Path:** `/organizations/me/projects`
- **Description:** Get all projects that the authenticated user is a member of
- **Query Parameters:** Standard pagination parameters
- **Response:** List of projects with pagination

#### Get Project by ID
- **Method:** `GET`
- **Path:** `/projects/{project_id}`
- **Description:** Get a specific project by its ID
- **Response:** Single project object

### Snapshots

#### Get Snapshot by ID
- **Method:** `GET`
- **Path:** `/snapshots/{snapshot_id}`
- **Description:** Get a specific snapshot by its ID
- **Response:** Single snapshot object

#### Get Project Snapshots
- **Method:** `GET`
- **Path:** `/projects/{project_id}/snapshots`
- **Description:** Get all snapshots for a project that the authenticated user is a member of
- **Response:** List of snapshots

#### Create Snapshot
- **Method:** `POST`
- **Path:** `/projects/{project_id}/snapshots`
- **Description:** Create a new snapshot in the specified project
- **Request Body:** `InputSnapshot` schema
- **Response:** Created snapshot object

#### Get S3 Upload Credentials
- **Method:** `GET`
- **Path:** `/snapshots/{snapshot_id}/s3-upload-credentials`
- **Description:** Get temporary S3 credentials for uploading snapshot data
- **Response:** S3 upload credentials object

#### Commit Snapshot
- **Method:** `POST`
- **Path:** `/snapshots/{snapshot_id}/commit`
- **Description:** Commit a snapshot with uploaded data
- **Request Body:** `CommitSnapshot` schema
- **Response:** Updated snapshot object

## Data Models

### Project Model
```json
{
  "id": "string", // ObjectID
  "name": "string"
}
```

### Snapshot Model
```json
{
  "id": "string", // ObjectID
  "name": "string",
  "project_id": "string", // ObjectID
  "status": "string", // "created", "committed", etc.
  "snapshot_source": "scanace",
  "created_at": "datetime", // always in UTC
  "updated_at": "datetime" // always in UTC
}
```

### Input Snapshot Model
```json
{
  "name": "string",
  "location": { // optional - used for locating the snapshot in the SDX-Cloud. If not provided calculated by sodex
    "latitude": "number",
    "longitude": "number"
  },
  "utm_zone_nr": "number (1-60)",
  "utm_hemisphere": "N | S",
  "description": "string (optional)",
  "start_capture_at": "datetime (optional)", // provide in UTC
  "end_capture_at": "datetime (optional)" // provide in UTC
}
```

### Commit Snapshot Model
```json
{
  "laz_s3_key": "string"
}
```

### S3 Upload Credentials Model
```json
{
  "access_key_id": "string",
  "access_key_secret": "string",
  "session_token": "string",
  "region": "string", // eu-central-1 or us-east-1. Not fully validated/tested on us-east-1 yet
  "bucket": "string", // sodex-integrations
  "s3_prefix": "string" // scanace/... - user only has access to this prefix
}
```

## Error Handling

The API uses standard HTTP status codes:
- `200` - Success
- `400` - Bad Request (validation errors)
- `401` - Unauthorized (authentication required)
- `403` - Forbidden (insufficient permissions)
- `404` - Not Found
- `500` - Internal Server Error

## Notes
- S3 credentials are valid for 4 hours
- All timestamps are in ISO 8601 format and UTC timezone
- The API follows RESTful conventions
- All responses are wrapped in a `GenericResponse` structure with `data` and optional `pagination` fields
- S3 regions: eu-central-1 (recommended) or us-east-1 (not fully validated/tested yet)
- S3 bucket: sodex-integrations
- S3 prefix: scanace/... (user only has access to this prefix)
