# Maestro Extractor

gRPC service for document text extraction. Converts PDF, DOCX, PPTX, XLSX, images, emails (MSG/EML), and other document formats into clean Markdown text.

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
