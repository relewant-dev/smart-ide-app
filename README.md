# Smart IDE App

Smart IDE App is a compact web interface for sending text prompts, asking questions about PDF documents, and downloading anonymized CVs. It provides a single, responsive composer while keeping the browser isolated from the processing service behind server-side Next.js proxy routes.

## Overview

The project gives users one place to choose between three related workflows:

- submit a text-only prompt;
- attach a PDF and ask a question about its contents; or
- attach a PDF without a question to request an anonymized CV.

The application exists as the web client and API boundary for a separate processing backend. Browser requests are sent to same-origin Next.js route handlers, which forward them to the configured backend. This avoids requiring the browser to communicate with the backend directly and, in local development, prevents cross-origin preflight issues.

> [!IMPORTANT]
> This repository contains the frontend and its proxy routes. Prompt execution, PDF analysis, and CV anonymization are performed by an external backend and are not implemented here.

## Features

- **Unified prompt composer** for text prompts and PDF-based tasks.
- **PDF question answering workflow** using multipart form uploads.
- **CV anonymization workflow** that returns a downloadable PDF.
- **Keyboard-friendly submission**: <kbd>Enter</kbd> submits and <kbd>Shift</kbd> + <kbd>Enter</kbd> inserts a new line.
- **Request state handling** for progress, backend errors, successful responses, and generated downloads.
- **Same-origin API proxy** with configurable backend routing.
- **Responsive, accessible interface** with labelled controls, live status announcements, and mobile layouts.
- **Production container build** based on a multi-stage Node.js Alpine image.

## Architecture

Smart IDE App uses the Next.js App Router. React renders the interface, TanStack Query manages the three mutations, and a small typed client translates user actions into calls to local route handlers. Those handlers relay the request to the processing backend without changing the core payload.

```text
┌──────────────────────── Browser ────────────────────────┐
│                                                        │
│  PromptComposer                                        │
│       │                                                │
│       ├── React state                                  │
│       └── TanStack Query mutations                     │
│                    │                                   │
│               Typed API client                         │
└────────────────────┬───────────────────────────────────┘
                     │ same-origin HTTP
┌────────────────────▼──── Next.js server ───────────────┐
│  /api/prompts/execute                                  │
│  /api/documents/pdf                                    │
│  /api/cv/anonymized-pdf                                │
└────────────────────┬───────────────────────────────────┘
                     │ proxied HTTP
              ┌──────▼──────────┐
              │ External backend│
              │ (default :8000) │
              └─────────────────┘
```

The proxy target is selected from `BACKEND_URL`, then `NEXT_PUBLIC_BACKEND_URL`, and finally defaults to `http://localhost:8000`. Prefer the server-only `BACKEND_URL` variable so the backend address does not need to be exposed to browser bundles.

## How It Works

1. The user enters a message, selects a PDF, or does both.
2. The composer chooses a workflow from the submitted values:
   - a message without a PDF calls the prompt endpoint;
   - a PDF with a message calls the document endpoint;
   - a PDF without a message calls the CV anonymization endpoint.
3. A TanStack Query mutation invokes the typed browser API client.
4. The client serializes text prompts as JSON and file requests as `FormData`.
5. The matching Next.js route handler parses the request and forwards it to the external backend.
6. JSON responses are displayed in the composer. An anonymized PDF response is converted to a browser object URL and downloaded as `anonymized-<original-name>`.
7. Non-success responses are converted into readable errors. Structured backend `detail` values are supported when they are strings, arrays, or validation-style objects with a `msg` field.

The application does not implement document-processing algorithms itself. For a request containing *n* bytes, proxying is **O(n)** in transfer work. The current route handlers also materialize text or PDF responses before returning them, so response memory usage is **O(n)** rather than streamed constant-memory processing.

## Technologies Used

### Frontend

- [Next.js 15](https://nextjs.org/) with the App Router
- [React 19](https://react.dev/)
- [TanStack Query 5](https://tanstack.com/query/latest) for mutation state
- [Tailwind CSS 3](https://tailwindcss.com/) for styling
- TypeScript with strict type checking

### Server and API

- Next.js route handlers
- Fetch API, JSON, and multipart form data
- Node.js 24 in the provided container image

### Tooling and Deployment

- ESLint with the Next.js configuration
- npm and `package-lock.json` for reproducible dependency installation
- Multi-stage Docker build for production deployment

No database, authentication provider, or cloud-specific service is configured in this repository.

## Project Structure

```text
.
├── app/
│   ├── api/
│   │   ├── client.tsx                 # Browser-side API client and error formatting
│   │   ├── cv/anonymized-pdf/route.ts # CV anonymization proxy
│   │   ├── documents/pdf/route.ts     # PDF question proxy
│   │   └── prompts/execute/route.ts   # Text prompt proxy
│   ├── components/                    # Composer and SVG icon components
│   ├── hooks/                         # TanStack Query mutation hooks
│   ├── models/                        # Request and response TypeScript types
│   ├── globals.css                    # Global styles and Tailwind directives
│   ├── layout.tsx                     # Root document layout and metadata
│   ├── page.tsx                       # Home page
│   └── providers.tsx                  # Application-level QueryClient provider
├── Dockerfile                         # Production container build
├── eslint.config.ts                   # Lint configuration
├── next.config.ts                     # Next.js configuration
├── tailwind.config.ts                 # Tailwind content configuration
├── tsconfig.json                      # TypeScript compiler configuration
├── package.json                       # Scripts and dependencies
└── LICENSE                            # MIT License
```

## Installation

### Prerequisites

- Node.js 24, matching the provided Docker image
- npm
- A compatible processing backend exposing the three endpoints documented below

Clone the repository and install the locked dependencies:

```bash
git clone <repository-url>
cd pdf-processor-app
npm ci
```

No frontend environment variable is required when the backend is available at `http://localhost:8000`. Otherwise, configure the backend base URL before starting the application:

```bash
export BACKEND_URL=http://127.0.0.1:8000
```

Do not include a trailing slash; the application removes one when present.

## Running Locally

Start the external processing backend on port `8000`, or at the address supplied through `BACKEND_URL`, using the instructions for that service. The backend source and its startup command are not part of this repository.

Then start the frontend development server:

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). To use a different backend for a single run:

```bash
BACKEND_URL=http://127.0.0.1:8000 npm run dev
```

For a production build:

```bash
npm run build
npm start
```

Or build and run the container:

```bash
docker build -t smart-ide-app .
docker run --rm -p 3000:3000 \
  -e BACKEND_URL=http://host.docker.internal:8000 \
  smart-ide-app
```

The backend URL in the Docker example assumes Docker Desktop. Use an address reachable from the container in other environments.

## API Documentation

All endpoints are relative to the Next.js application and accept `POST` requests. They proxy to the same path on the configured backend.

### Execute a text prompt

`POST /api/prompts/execute`

**JSON body**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `message` | string | Yes | Prompt to send to the backend. |

```bash
curl --request POST http://localhost:3000/api/prompts/execute \
  --header 'Content-Type: application/json' \
  --data '{"message":"Summarize the supplied context."}'
```

Expected successful response shape:

```json
{
  "response": "Backend-generated response"
}
```

### Ask a question about a PDF

`POST /api/documents/pdf`

**Multipart form fields**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | PDF file | Yes | Document sent to the backend. |
| `question` | string | Yes | Question about the document. |
| `max_chars` | number | No | Optional maximum-character value forwarded by the API client. |

```bash
curl --request POST http://localhost:3000/api/documents/pdf \
  --form 'file=@./document.pdf;type=application/pdf' \
  --form 'question=What are the main points?'
```

Expected successful response shape:

```json
{
  "response": "Backend-generated answer"
}
```

### Anonymize a CV

`POST /api/cv/anonymized-pdf`

**Multipart form fields**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | PDF file | Yes | CV sent to the anonymization backend. |

```bash
curl --request POST http://localhost:3000/api/cv/anonymized-pdf \
  --form 'file=@./cv.pdf;type=application/pdf' \
  --output anonymized-cv.pdf
```

On success, the endpoint returns PDF bytes with `Content-Type: application/pdf` unless the backend supplies another content type. Backend error bodies and status codes are passed through by the proxy routes.

## Implementation Details

### One composer, explicit dispatch rules

The UI derives the operation from the presence of a selected file and a non-empty message. This keeps the interface small while preserving predictable behavior for all three workflows. Only one mutation may be pending at a time, and its state controls the shared disabled, loading, error, and success UI.

### Server-side proxy boundary

The browser communicates only with the Next.js origin. Centralizing backend access in route handlers avoids local browser CORS configuration and gives the application one place to preserve upstream status codes, response bodies, and content types.

### Payload-specific handling

Text prompts use JSON. PDF operations use the platform's `FormData` implementation so the runtime supplies the multipart boundary. The anonymization route treats the response as binary data and preserves download headers; the two text workflows return the upstream text response with its content type.

### Typed mutation layer

Request and response models are separated from UI components. Small custom hooks bind those types to TanStack Query, while the shared client normalizes errors before they reach the composer.

## Performance

The frontend performs no CPU-intensive PDF processing. Network transfer and backend execution dominate request time. UI dispatch and mutation selection are **O(1)**; upload and response handling are **O(n)** in payload size.

Requests are mutation-oriented and are not cached or retried explicitly by this application. Each submission reaches the backend. Large files and generated PDFs are currently buffered through standard `FormData`, text, `Blob`, or `ArrayBuffer` APIs, so available memory limits practical payload size. Horizontal scaling remains possible because the Next.js proxy stores no application session or database state, provided each instance can reach the same backend.

## Future Improvements

Potential extensions, not currently implemented, include:

- stream large backend responses to reduce proxy memory usage;
- add automated tests for dispatch behavior, error formatting, and route handlers;
- define and validate shared request/response schemas at runtime;
- add configurable upload limits and clearer client-side file validation;
- document and automate an integrated deployment with the processing backend.

## License

This project is available under the [MIT License](./LICENSE).
