# Maestro Extractor

gRPC service for document text extraction. Converts PDF, DOCX, PPTX, XLSX, images, emails (MSG/EML), and other document formats into clean Markdown text.

## Why gRPC?

Document extraction deals with **large binary payloads** — PDFs, images, Office files. gRPC is the right protocol here because:

- **Binary-native** — Protobuf sends file bytes directly without Base64 encoding overhead. A 10 MB PDF stays 10 MB over the wire, not 13.3 MB as it would with JSON/REST.
- **Streaming-ready** — gRPC supports bidirectional streaming out of the box. While this service currently uses simple unary calls, it can evolve to stream chunks of extracted text as they're processed without changing the transport layer.
- **Strongly typed contracts** — The `.proto` file is the single source of truth for the API. Clients in any language (Python, Go, TypeScript, Rust) can generate type-safe stubs from it — no hand-written API clients, no runtime validation.
- **Connection multiplexing** — HTTP/2 under the hood means multiple concurrent requests share one TCP connection. Maestro can fire off several extraction requests in parallel without connection pool overhead.
- **Lingua franca for microservices** — If you later want to write a faster extractor in Go or Rust, you swap the implementation behind the same `.proto` contract. No client changes needed.

For Maestro's use case — sending documents from the platform to a sidecar extraction service — gRPC is leaner, faster, and more type-safe than REST.

## Architecture

```
┌──────────────┐  gRPC :50051  ┌───────────────┐
│   Maestro    │──────────────▶│   Extractor   │
│  (Platform)  │               │               │
└──────────────┘               │  MarkItDown   │
                               │  extract-msg  │
                               │  markdownify  │
                               └───────────────┘
```

Maestro Platform calls the Extractor over gRPC when documents need to be converted to text for processing.

## Supported Formats

| Format | Extension | Method |
|--------|-----------|--------|
| PDF | `.pdf` | MarkItDown |
| Word | `.docx` | MarkItDown |
| PowerPoint | `.pptx` | MarkItDown |
| Excel | `.xlsx` | MarkItDown |
| Images | `.png`, `.jpg`, `.gif`, ... | MarkItDown (with optional LLM vision) |
| Outlook Email | `.msg` | extract-msg |
| Email | `.eml` | Python email module |
| HTML | `.html` | markdownify |
| Other | various | MarkItDown fallback |

## Quick Start

### Docker (recommended)

```bash
docker run -p 50051:50051 ghcr.io/altipard/maestro-extractor
```

### Docker Compose (with Maestro)

The extractor is included in Maestro's `compose.yaml`:

```yaml
extractor:
  image: ghcr.io/altipard/maestro-extractor
```

Configure Maestro to use it:

```yaml
extractors:
  default:
    type: grpc
    url: grpc://extractor:50051
```

### Local

```bash
pip install -r requirements.txt
python main.py
```

## Configuration

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `OPENAI_BASE_URL` | LLM API URL for vision-based extraction (e.g. image OCR) | No |
| `OPENAI_API_KEY` | LLM API key | No |
| `OPENAI_MODEL` | LLM model name for vision extraction | No |

The LLM integration is optional. Without it, images are processed using MarkItDown's built-in extraction. With an LLM configured, images can be described using vision models for richer text output.

## gRPC API

Defined in `extractor.proto`:

```protobuf
service Extractor {
  rpc Extract (ExtractRequest) returns (Document);
}

message ExtractRequest {
  File file = 1;
}

message File {
  string name = 1;
  bytes content = 2;
  string content_type = 3;
}

message Document {
  string text = 1;
}
```

### Regenerate Proto Files

```bash
task generate
```

## Available Task Commands

| Command | Description |
|---------|-------------|
| `task run` | Run via Docker |
| `task generate` | Regenerate protobuf Python files |
| `task publish` | Build and push multi-arch Docker image |

## Port

| Service | Port | Protocol |
|---------|------|----------|
| Extractor | 50051 | gRPC |
