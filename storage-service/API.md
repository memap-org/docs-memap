# Storage Service API

## Base URL

`http://localhost:8088/storage`

## Authentication

All endpoints require a valid JWT Bearer token in the `Authorization` header:

```
Authorization: Bearer <jwt_token>
```

## Swagger

Interactive API documentation: http://localhost:8088/storage/swagger-ui.html

## Endpoints

### Upload File

**POST** `/file/upload`

Upload a file with optional custom name.

**Request:**

- Content-Type: `multipart/form-data`
- Parameters:
  - `file` (required): The file to upload
  - `customName` (optional): Custom display name for the file

**Response:** `200 OK`

```json
{
  "code": 1000,
  "result": {
    "fileId": "550e8400-e29b-41d4-a716-446655440000",
    "name": "my-document.pdf",
    "originalName": "document.pdf",
    "contentType": "application/pdf",
    "size": 1048576,
    "md5Checksum": "d41d8cd98f00b204e9800998ecf8427e",
    "downloadUrl": "http://localhost:8088/storage/file/550e8400-e29b-41d4-a716-446655440000/download",
    "createdAt": "2026-02-06T10:30:00"
  }
}
```

### Download File

**GET** `/file/{fileId}/download`

Download a file by its ID.

**Response:** Binary file with headers:

- `Content-Type`: File MIME type
- `Content-Disposition`: `attachment; filename="..."`
- `Content-Length`: File size in bytes
- `Content-MD5`: MD5 checksum

### Get File Info

**GET** `/file/{fileId}`

Get file metadata by ID.

**Response:** `200 OK`

```json
{
  "code": 1000,
  "result": {
    "fileId": "550e8400-e29b-41d4-a716-446655440000",
    "name": "my-document.pdf",
    "originalName": "document.pdf",
    "contentType": "application/pdf",
    "size": 1048576,
    "md5Checksum": "d41d8cd98f00b204e9800998ecf8427e",
    "downloadUrl": "http://localhost:8088/storage/file/550e8400-e29b-41d4-a716-446655440000/download",
    "createdAt": "2026-02-06T10:30:00"
  }
}
```

### List My Files

**GET** `/file/my-files`

Get all files owned by the authenticated user.

**Response:** `200 OK`

```json
{
  "code": 1000,
  "result": [
    {
      "fileId": "550e8400-e29b-41d4-a716-446655440000",
      "name": "document.pdf",
      "originalName": "document.pdf",
      "contentType": "application/pdf",
      "size": 1048576,
      "md5Checksum": "d41d8cd98f00b204e9800998ecf8427e",
      "downloadUrl": "http://localhost:8088/storage/file/550e8400.../download",
      "createdAt": "2026-02-06T10:30:00"
    }
  ]
}
```

### Delete File

**DELETE** `/file/{fileId}`

Delete a file by ID. Only the file owner can delete.

**Response:** `200 OK`

```json
{
  "code": 1000,
  "result": "File deleted successfully"
}
```

## Error Responses

| Code | Message                             | HTTP Status |
| ---- | ----------------------------------- | ----------- |
| 4200 | File not found                      | 404         |
| 4201 | File upload failed                  | 500         |
| 4202 | File size exceeds the maximum limit | 400         |
| 4203 | Invalid file type                   | 400         |
| 4204 | Storage quota exceeded              | 400         |
| 4205 | Storage error                       | 500         |
| 4100 | Unauthenticated                     | 401         |
| 4103 | Access denied                       | 403         |

**Error Response Format:**

```json
{
  "code": 4200,
  "message": "File not found"
}
```
