# File Service API Documentation

## Base URL

```
http://localhost:8087
```

## Authentication

All endpoints require authentication via JWT tokens in the Authorization header:

```
Authorization: Bearer {token}
```

---

## Endpoints

### File Upload

Upload a file to MinIO storage.

```http
POST /file
Content-Type: multipart/form-data
Authorization: Bearer {token}

file: (binary)
```

**Request:**

- `file`: File to upload (multipart form data)

**Response:**

```
200 OK
```

**Example with cURL:**

```bash
curl -X POST http://localhost:8087/file \
  -H "Authorization: Bearer {token}" \
  -F "file=@/path/to/document.pdf"
```

---

### File Download

Download a file directly.

```http
GET /file/download?fileName={fileName}
Authorization: Bearer {token}
```

**Query Parameters:**

- `fileName` (required): Name of the file to download

**Response:**

- Binary file content
- Headers:
  - `Content-Disposition: attachment; filename="{fileName}"`
  - `Content-Type: application/octet-stream` (or specific MIME type)

**Example:**

```bash
curl -X GET "http://localhost:8087/file/download?fileName=document.pdf" \
  -H "Authorization: Bearer {token}" \
  -o downloaded_file.pdf
```

**Error Response (404):**

```
500 Internal Server Error
```

---

### Get Presigned Download URL

Generate a temporary presigned URL for file download.

```http
GET /file/download-link?fileName={fileName}
Authorization: Bearer {token}
```

**Query Parameters:**

- `fileName` (required): Name of the file

**Response:**

```
200 OK
Content-Type: text/plain

https://minio-server.com/bucket/path/to/file?X-Amz-Algorithm=...&X-Amz-Expires=3600
```

**Example Response:**

```
https://localhost:9000/memap-file-service/document.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=admin123%2F20260129%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20260129T100000Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=...
```

**Error Response (500):**

```
Error generating link
```

**Example:**

```bash
curl -X GET "http://localhost:8087/file/download-link?fileName=document.pdf" \
  -H "Authorization: Bearer {token}"
```

---

## Usage Examples

### 1. Upload a File

**JavaScript (Fetch API):**

```javascript
const formData = new FormData();
formData.append('file', fileInput.files[0]);

const response = await fetch('http://localhost:8087/file', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${token}`,
  },
  body: formData,
});

if (response.ok) {
  console.log('File uploaded successfully');
}
```

**Python (requests):**

```python
import requests

files = {'file': open('document.pdf', 'rb')}
headers = {'Authorization': f'Bearer {token}'}

response = requests.post(
    'http://localhost:8087/file',
    files=files,
    headers=headers
)

if response.status_code == 200:
    print('File uploaded successfully')
```

---

### 2. Download a File

**JavaScript (Fetch API):**

```javascript
const response = await fetch(
  `http://localhost:8087/file/download?fileName=${fileName}`,
  {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  },
);

const blob = await response.blob();
const url = window.URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = fileName;
a.click();
```

**Python (requests):**

```python
import requests

headers = {'Authorization': f'Bearer {token}'}
params = {'fileName': 'document.pdf'}

response = requests.get(
    'http://localhost:8087/file/download',
    headers=headers,
    params=params
)

with open('downloaded_file.pdf', 'wb') as f:
    f.write(response.content)
```

---

### 3. Get Presigned URL

**JavaScript (Fetch API):**

```javascript
const response = await fetch(
  `http://localhost:8087/file/download-link?fileName=${fileName}`,
  {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  },
);

const presignedUrl = await response.text();
console.log('Presigned URL:', presignedUrl);

// Use the presigned URL (no authentication needed)
window.open(presignedUrl, '_blank');
```

**Python (requests):**

```python
import requests

headers = {'Authorization': f'Bearer {token}'}
params = {'fileName': 'document.pdf'}

response = requests.get(
    'http://localhost:8087/file/download-link',
    headers=headers,
    params=params
)

presigned_url = response.text
print(f'Presigned URL: {presigned_url}')
```

---

## Error Codes

- `200 OK`: Successful operation
- `400 Bad Request`: Invalid request (missing file, invalid filename)
- `401 Unauthorized`: Missing or invalid authentication token
- `404 Not Found`: File not found
- `500 Internal Server Error`: Server error (MinIO error, database error)

## Error Response Format

```
Content-Type: text/plain

Error message description
```

---

## File Naming Conventions

### Original Filename Preservation

The service preserves the original filename when storing and retrieving files.

### Recommended Naming

- Use descriptive names: `project-documentation.pdf`
- Avoid special characters: Use `-` or `_` instead of spaces
- Include file extension: `.pdf`, `.jpg`, `.docx`

---

## Presigned URL Details

### URL Expiration

- Default expiration: 1 hour (3600 seconds)
- Configurable in MinIO configuration
- After expiration, URL becomes invalid

### Security

- No authentication required to access presigned URL
- URL contains signature that validates the request
- Cannot be modified without invalidating signature
- Share URLs carefully as they provide temporary public access

### Use Cases

1. **Email Attachments**: Send presigned URLs instead of files
2. **Download Links**: Provide time-limited download links
3. **Embedded Media**: Embed images/videos using presigned URLs
4. **API Responses**: Return presigned URLs instead of binary data

---

## File Metadata

While the current API doesn't expose metadata endpoints, the service stores:

- **File ID**: Unique identifier
- **Original Filename**: User-provided filename
- **Content Type**: MIME type (e.g., `application/pdf`)
- **File Size**: Size in bytes
- **Upload Date**: Timestamp of upload
- **Uploader ID**: User who uploaded the file
- **Storage Path**: Path in MinIO bucket

---

## Integration with Other Services

### Roadmap Service

```http
POST /road-map/api/roadmap/media/upload/{roadmapId}
Content-Type: multipart/form-data

file: (binary)
```

### Profile Service

```http
POST /profile/api/profile/avatar
Content-Type: multipart/form-data

file: (binary)
```

Both services internally call the File Service for storage and receive file URLs in response.

---

## Best Practices

1. **File Size Validation**: Validate file size on client before upload
2. **File Type Validation**: Check file types on client side
3. **Progress Tracking**: Implement upload progress indicators
4. **Error Handling**: Handle network errors and file upload failures
5. **Presigned URLs**: Cache presigned URLs with respect to expiration time
6. **File Naming**: Use unique, descriptive filenames to avoid conflicts
7. **Cleanup**: Implement file cleanup for temporary/unused files

---

## Limitations

- **Maximum File Size**: 10 MB (configurable via Spring Boot)
- **Concurrent Uploads**: Limited by server resources
- **Storage Quota**: Depends on MinIO server configuration
- **Bandwidth**: Limited by network and server capacity

---

## Future Enhancements

1. **Chunked Upload**: Support large file uploads with chunking
2. **File Versioning**: Track file versions and changes
3. **Thumbnail Generation**: Auto-generate thumbnails for images
4. **File Compression**: Automatic compression for large files
5. **CDN Integration**: Integrate with CDN for faster delivery
6. **Virus Scanning**: Scan uploaded files for malware
7. **Metadata API**: Expose endpoints for file metadata queries
8. **Batch Operations**: Support multiple file uploads/downloads
