# Taquion API — Public Reference

> **Published copy:** [andrequeiroz2/taquion_api_readme](https://github.com/andrequeiroz2/taquion_api_readme) → `README.md`.

## Routes


| Method | Path                       | Description                                                                     |
| ------ | -------------------------- | ------------------------------------------------------------------------------- |
| GET    | `/health`                  | Service status and critical dependency check                                    |
| GET    | `/agents`                  | Available transcription model identifiers                                       |
| POST   | `/transcribe`              | `multipart/form-data` with `model` (text) and `audio` (file)                    |
| POST   | `/document/extract-text`   | `multipart/form-data` with `document` (PDF file)                                |
| POST   | `/document/extract-pages`  | `multipart/form-data` with `document` (PDF file) + optional `pages` query       |
| POST   | `/document/extract-fields` | `multipart/form-data` with `document` + `fields`; optional LLM/query parameters |


## Common headers

- `Content-Type` / `Accept`: route-dependent (`application/json`, `multipart/form-data`, ...).
- Body-size limits: see **Limits**.

---

## Endpoint: `GET /health`

### Purpose

Returns service health status and verifies database connectivity.

### Request format

- **Method:** `GET`
- **Path:** `/health`
- **Body:** none
- **Content-Type:** not required

### Internal validation/rules

- Executes a DB connectivity check before returning success.
- Includes `elapsed_ms` in both success and error responses.
- If DB is unavailable, returns `503 service_unavailable`.

### cURL example

```bash
curl -sS -X GET "http://localhost:18080/health"
```

### Possible responses

**200 OK**

```json
{
  "status": "ok",
  "elapsed_ms": 12
}
```

**503 Service Unavailable**

```json
{
  "error": {
    "code": "service_unavailable",
    "message": "The service is temporarily unavailable.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 8
}
```

---

## Endpoint: `GET /agents`

### Purpose

Returns the list of transcription model identifiers currently supported by the API.

### Request format

- **Method:** `GET`
- **Path:** `/agents`
- **Body:** none
- **Content-Type:** not required

### Internal validation/rules

- Includes `elapsed_ms` in success and internal-error responses.
- No request payload validation is required for this route.

### cURL example

```bash
curl -sS -X GET "http://localhost:18080/agents"
```

### Possible responses

**200 OK**

```json
{
  "agents": ["parakeet", "whisper-small", "whisper-medium"],
  "elapsed_ms": 3
}
```

**500 Internal Server Error**

```json
{
  "error": {
    "code": "internal_error",
    "message": "An internal error occurred.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

---

## Endpoint: `POST /transcribe`

### Purpose

Transcribes an uploaded audio file using a supported model and returns normalized text plus request metadata.

### Request format

- **Method:** `POST`
- **Path:** `/transcribe`
- **Content-Type:** `multipart/form-data` (required)
- **Body fields:**
  - `model` (text, required): transcription model identifier (see **Supported `model` values** below)
  - `audio` (file, required): WAV or OGG audio payload

### Supported `model` values

Use these canonical identifiers in requests (`GET /agents` returns the same values):

**Models supported:** `parakeet` or`whisper-small` or `whisper-medium`.

**Runtime note:** `parakeet` is the path supported by the default ONNX build. `whisper-small` and `whisper-medium` are accepted by validation but return `**503 whisper_unavailable`** unless the server is built with Whisper (`whisper-cpp`) and the required model files are present — see **Models (`model`)** below (TODO).

### Internal validation/rules

- Rejects non-multipart requests with `415 unsupported_media_type`.
- Requires both `model` and `audio` fields; missing/invalid multipart parts return `400`.
- Validates `model` against the supported identifiers above; unknown values return `400 invalid_model`.
- Enforces upload size from `max_file_size_bytes` (default ~50 MB), with HTTP payload limit aligned server-side.
- Validates audio format/spec before transcription:
  - WAV: 16 kHz, 16-bit PCM, mono, non-empty
  - OGG: supported decode path (converted to 16 kHz mono before transcription)

### cURL example

```bash
curl -sS -X POST "http://localhost:18080/transcribe" \
  -F "model=parakeet" \
  -F "audio=@/absolute/path/sample.wav"
```

### Possible responses

**200 OK**

```json
{
  "transcription": "hello world",
  "file_size_bytes": 123456,
  "token_count": 2,
  "model_used": "parakeet",
  "elapsed_ms": 140
}
```

**415 Unsupported Media Type**

```json
{
  "error": {
    "code": "unsupported_media_type",
    "message": "Content-Type must be multipart/form-data with model and audio fields.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 1
}
```

**400 Bad Request** (example: invalid WAV spec)

```json
{
  "error": {
    "code": "invalid_wav_spec",
    "message": "WAV must be 16 kHz, 16-bit, mono PCM.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 5
}
```

**503 Service Unavailable** (example: model not configured)

```json
{
  "error": {
    "code": "model_not_configured",
    "message": "Transcription models are not configured.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 3
}
```

---

## Endpoint: `POST /document/extract-text`

### Purpose

Extracts full text from an uploaded PDF document and returns the concatenated text with processing metadata.

### Request format

- **Method:** `POST`
- **Path:** `/document/extract-text`
- **Content-Type:** `multipart/form-data` (required)
- **Body fields:**
  - `document` (file, required): PDF payload

### Internal validation/rules

- Rejects non-multipart requests with `415 unsupported_media_type`.
- Requires `document` field; missing/empty field returns `400 missing_document`.
- Enforces upload size from `max_file_size_bytes` (default ~50 MB).
- Verifies PDF magic header (`%PDF-`) and parses page count.
- Enforces page limit from `max_pages` 15 pages (`400 too_many_pages` when exceeded).
- Attempts text extraction from all pages; extraction failure returns `422 document_ocr_failed`.
- Includes `elapsed_ms` in success and error responses.

### cURL example

```bash
curl -sS -X POST "http://localhost:18080/document/extract-text" \
  -F "document=@/absolute/path/document.pdf"
```

### Possible responses

**200 OK**

```json
{
  "text": "Hello World!",
  "pages_processed": 1,
  "page_limits": {
    "max_pages": 15
  },
  "elapsed_ms": 22
}
```

**400 Bad Request** (example: invalid PDF)

```json
{
  "error": {
    "code": "invalid_pdf",
    "message": "The uploaded file is not a PDF.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**400 Bad Request** (example: page limit exceeded)

```json
{
  "error": {
    "code": "too_many_pages",
    "message": "The document exceeds the maximum number of pages allowed.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 4
}
```

**422 Unprocessable Entity**

```json
{
  "error": {
    "code": "document_ocr_failed",
    "message": "The document text could not be extracted.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 7
}
```

---

## Endpoint: `POST /document/extract-pages`

### Purpose

Extracts text per page from an uploaded PDF and returns an array of page objects (`page`, `text`).

### Request format

- **Method:** `POST`
- **Path:** `/document/extract-pages`
- **Content-Type:** `multipart/form-data` (required)
- **Body fields:**
  - `document` (file, required): PDF payload
- **Query parameters:**
  - `pages` (optional): 1-based page selection (for example `1,3,5-7`)

### Internal validation/rules

- Rejects non-multipart requests with `415 unsupported_media_type`.
- Requires `document` field; missing/empty field returns `400 missing_document`.
- Enforces upload size from `max_file_size_bytes` (default ~50 MB).
- Verifies PDF magic header (`%PDF-`) and parses page count.
- Enforces page limit from `max_pages` 15 pages (`400 too_many_pages` when exceeded).
- Validates `pages` query syntax/range:
  - invalid token/range (`0`, `7-3`, non-numeric) -> `400 invalid_pages_parameter`
  - page out of document range -> `400 pages_out_of_range`
  - empty effective selection -> `400 empty_pages_selection`
- If `pages` is omitted or blank, all pages are selected.
- Includes `elapsed_ms` in success and error responses.

### cURL example

```bash
curl -sS -X POST "http://localhost:18080/document/extract-pages?pages=1,3,5-7" \
  -F "document=@/absolute/path/document.pdf"
```

### Possible responses

**200 OK**

```json
{
  "pages": [
    { "page": 1, "text": "First page text..." },
    { "page": 3, "text": "Third page text..." }
  ],
  "limits": {
    "max_pages": 15
  },
  "elapsed_ms": 30
}
```

**400 Bad Request** (example: invalid `pages` parameter)

```json
{
  "error": {
    "code": "invalid_pages_parameter",
    "message": "The pages query parameter contains an invalid value.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**400 Bad Request** (example: page out of range)

```json
{
  "error": {
    "code": "pages_out_of_range",
    "message": "One or more page numbers are outside the document.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**422 Unprocessable Entity**

```json
{
  "error": {
    "code": "document_ocr_failed",
    "message": "The document text could not be extracted.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 6
}
```

---

## Endpoint: `POST /document/extract-fields`

### Purpose

Extracts semantic field values from an uploaded PDF using either heuristic rules or an LLM-backed extraction mode.

### Request format

- **Method:** `POST`
- **Path:** `/document/extract-fields`
- **Content-Type:** `multipart/form-data` (required)
- **Body fields:**
  - `document` (file, required): PDF payload
  - `fields` (text, required): field definitions
    - CSV format: `name1,name2,name3`
    - JSON format: array of strings and/or objects with optional context  
    example: `[{"name":"Total","context":"decimal amount with comma"},"Issue Date"]`
- **Query parameters:**
  - `pages` (optional): 1-based page selection (for example `1,3,5-7`)
  - `internal_llm` (optional): omitted/`true`/`1` -> internal Ollama; `false`/`0` -> external LLM
  - `external_llm` (optional, used when `internal_llm=false`): `gemini`, `kimi`/`moonshot`, `openai_compatible`/`openai`

### Internal validation/rules

- Applies the same multipart/size/PDF/page-limit checks as document endpoints.
- Requires non-empty `fields`; otherwise returns `400 missing_fields`.
- `pages` parsing follows the same rules as `/document/extract-pages`.
- In LLM mode:
  - internal provider: Ollama (`internal_llm=true` or omitted)
  - external provider: selected by `external_llm` or server default (`openai_compatible`)
  - external API key and provider-specific config must be present
- Response includes `stage_ms` breakdown and optional `llm_token_usage` when provider usage is available.

### cURL example

```bash
curl -sS -X POST "http://localhost:18080/document/extract-fields?internal_llm=false&external_llm=openai_compatible&pages=1,2,7" \
  -F "document=@/absolute/path/document.pdf" \
  -F 'fields=[{"name":"Total Amount","context":"decimal amount with comma"},{"name":"Loan Amount","context":"decimal amount with comma"}]'
```

### Possible responses

**200 OK**

```json
{
  "fields": [
    { "name": "Total Amount", "value": "1.234,56" },
    { "name": "Loan Amount", "value": "800,00" }
  ],
  "field_requests": [
    { "name": "Total Amount", "context": "decimal amount with comma" },
    { "name": "Loan Amount", "context": "decimal amount with comma" }
  ],
  "extraction_mode": "llm",
  "llm_provider": "openai_compatible",
  "llm_token_usage": {
    "prompt_tokens": 1234,
    "completion_tokens": 56,
    "total_tokens": 1290
  },
  "pages_processed": 3,
  "page_limits": { "max_pages": 15 },
  "schema_version": "phase-2.0.c",
  "elapsed_ms": 240,
  "stage_ms": {
    "request_prep_ms": 20,
    "pdf_text_ms": 40,
    "llm_http_ms": 170,
    "llm_parse_ms": 5,
    "heuristic_ms": 0
  }
}
```

**400 Bad Request** (example: missing `fields`)

```json
{
  "error": {
    "code": "missing_fields",
    "message": "The fields form field is required: CSV names, or a JSON array of strings or objects with name and optional context.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**400 Bad Request** (example: invalid external provider)

```json
{
  "error": {
    "code": "invalid_external_llm_provider",
    "message": "Unknown external_llm value. Use gemini, kimi (or moonshot), or openai_compatible (or openai).",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**500 Internal Server Error** (example: Gemini model missing in server configuration)

```json
{
  "error": {
    "code": "internal_error",
    "message": "An internal error occurred.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 2
}
```

**502 Bad Gateway** (LLM backend failure)

```json
{
  "error": {
    "code": "field_extraction_llm_failed",
    "message": "Field extraction with the configured LLM service failed.",
    "documentation_url": "https://github.com/andrequeiroz2/taquion_api_readme/blob/main/README.md#errors"
  },
  "elapsed_ms": 120
}
```

---

## Models


| Value                             | Default server status                                    |
| --------------------------------- | -------------------------------------------------------- |
| `parakeet`                        | Parakeet ONNX model files must exist on disk (see below) |
| `whisper-small`, `whisper-medium` | **503** `whisper_unavailable`                            |


## Limits and audio format (`audio`)

Accepted formats: **WAV** and **OGG**. The server decodes input and **normalizes to 16 kHz mono** before transcription.

### WAV


| Property    | Value                              |
| ----------- | ---------------------------------- |
| Container   | WAV (RIFF `WAVE`)                  |
| Sample rate | **16 000 Hz**                      |
| Bits        | **16** (PCM int)                   |
| Channels    | **1** (mono)                       |
| Duration    | At least 1 sample (`duration` > 0) |


### OGG


| Property                     | Value                                                                                                |
| ---------------------------- | ---------------------------------------------------------------------------------------------------- |
| Container                    | OGG (`OggS` stream)                                                                                  |
| Codecs                       | **Opus** (decoded first) or **Vorbis** (fallback)                                                    |
| Input sample rate / channels | Any rate the decoder accepts (e.g. mono/stereo Opus); output is **16 kHz mono** before transcription |


## Errors (`error.code`)

response error: `{"error":{"code","message","documentation_url"},"elapsed_ms":int}`


| HTTP status | `error.code`                                                               | Description                                                                   |
| ----------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 404         | `not_found`                                                                | Path/method not registered                                                    |
| 400         | `invalid_model`                                                            | Unsupported model (`POST /transcribe`)                                        |
| 500         | `internal_error`                                                           | Internal failure (generic message; details in logs)                           |
| 503         | `service_unavailable`                                                      | Critical dependency unavailable (for example DB in `/health`)                 |
| 503         | `model_not_configured`                                                     | `TAQUION_MODELS_DIR` missing or incomplete Parakeet directory                 |
| 503         | `whisper_unavailable`                                                      | Whisper requested in an ONNX-only build                                       |
| 415         | `unsupported_media_type`                                                   | `POST /transcribe` without `multipart/form-data`                              |
| 413         | `payload_too_large`                                                        | Uploaded file exceeds configured maximum size                                 |
| 400         | `missing_model` / `missing_audio` / `malformed_multipart` / ...            | Invalid multipart body or missing fields                                      |
| 400         | `model_field_too_large` / `invalid_model_encoding`                         | Invalid `model` multipart field (`POST /transcribe`)                          |
| 400         | `invalid_wav`                                                              | `audio` payload is not a valid WAV                                            |
| 400         | `unsupported_audio_format`                                                 | Unsupported format (use WAV or OGG)                                           |
| 400         | `invalid_wav_spec`                                                         | WAV does not meet 16 kHz / 16-bit / mono PCM                                  |
| 400         | `empty_audio`                                                              | WAV has zero samples (`duration` = 0)                                         |
| 400         | `missing_document`                                                         | `document` multipart field is missing/empty (`/document/`*)                   |
| 400         | `invalid_pdf`                                                              | Uploaded file is not a readable/corrupted PDF (`/document/*`)                 |
| 400         | `too_many_pages`                                                           | PDF exceeds `config.doc_max_pages` (`/document/*`)                            |
| 400         | `invalid_pages_parameter` / `pages_out_of_range` / `empty_pages_selection` | Invalid `pages` query (`/document/extract-pages`, `/document/extract-fields`) |
| 400         | `missing_fields`                                                           | Missing/empty `fields` form field (`/document/extract-fields`)                |
| 400         | `invalid_external_llm_provider`                                            | Unsupported `external_llm` value (`/document/extract-fields`)                 |
| 400         | `external_llm_not_configured`                                              | Missing API key for selected external LLM provider                            |                                   |
| 422         | `document_ocr_failed`                                                      | Document text extraction failed (`/document/*`)                               |
| 502         | `field_extraction_llm_failed`                                              | Upstream/provider LLM call failed (`/document/extract-fields`)                |


