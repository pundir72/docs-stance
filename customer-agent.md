# Stance Health Customer Agent — Project KT Overview

## 1. Executive overview

The Stance Health Customer Agent is an AI-powered medical intake system. It interviews patients through text or voice, understands their answers, and stores the information in a structured clinical intake form for review by a Stance Health clinician.

The system can:

- Conduct a conversational patient interview.
- Accept typed and recorded answers.
- Convert speech to text.
- Extract medical information from natural-language answers.
- Ask follow-up questions when information is missing.
- Save and resume a patient's interview.
- Show interview and section progress.
- Accept medical-report uploads.
- Analyze uploaded medical reports.
- Generate a final summary for patient confirmation.
- Answer basic Stance Health and medical-education questions.

The complete solution consists of two separate repositories:

```text
customerAgent/
├── customer-agent-frontend   React/TypeScript patient application
└── customer-agent            Python/FastAPI AI backend
```

## 2. High-level architecture

```text
Patient browser
      │
      ▼
React + TypeScript frontend
      │
      ├── HTTP APIs
      ├── WebSocket messages
      └── Audio data
      │
      ▼
Python FastAPI backend
      │
      ├── LangGraph interview workflow
      ├── Gemini conversation and extraction
      ├── Faster-Whisper speech recognition
      ├── MongoDB patient forms and users
      ├── ChromaDB knowledge retrieval
      └── S3 uploads → Amazon Bedrock report analysis
```

The frontend is deployed separately from the backend. In production, HTTP requests are proxied through Vercel, while WebSocket connections go directly to the backend through `wss://customeragent.stance.health`.

## 3. Frontend application

### Technology

The frontend uses:

- React 18
- TypeScript
- Vite
- React Router
- Tailwind CSS
- shadcn/Radix UI components
- TanStack React Query
- Apollo Client for the separate Stance Health GraphQL API

### Main responsibilities

The frontend:

1. Opens a patient-specific interview URL.
2. Checks whether the patient has accepted consent.
3. Opens a WebSocket connection to the customer-agent backend.
4. Starts or resumes the patient's form.
5. Displays the conversation and interview progress.
6. Records microphone input and sends audio to the backend.
7. Displays and optionally auto-submits the transcription.
8. Renders interactive question controls when question metadata is available.
9. Uploads medical documents using an HTTP endpoint.
10. Displays the final interview summary.

### Main route

The normal patient route is:

```text
/{userId}/{formId}
```

The default form ID is:

```text
FRM-01
```

Therefore, a typical route is:

```text
/{userId}/FRM-01
```

If only `/{userId}` is provided, the frontend redirects to `/{userId}/FRM-01`.

### Important frontend files

| File | Responsibility |
|---|---|
| `src/App.tsx` | Application routes and providers |
| `src/pages/Index.tsx` | Reads the patient and form IDs from the URL |
| `src/components/TranscriptionInterface.tsx` | Main interview UI and interaction logic |
| `src/hooks/useWebSocket.ts` | WebSocket connection and protocol handling |
| `src/hooks/useVoiceRecorder.ts` | Browser microphone recording |
| `src/config/api.ts` | Backend HTTP and WebSocket URLs |
| `src/utils/graphql-client.ts` | Separate Stance Health GraphQL integration |

## 4. Backend application

### Technology

The backend uses:

- Python 3.12
- FastAPI
- Uvicorn
- WebSockets
- LangGraph
- LlamaIndex Google GenAI integration
- Gemini models
- Faster-Whisper
- MongoDB
- AWS S3
- Amazon Bedrock
- ChromaDB
- Docker and Docker Compose

### Main responsibilities

The backend:

1. Validates the patient ID.
2. Loads or creates the patient's form.
3. Maintains interview state for the WebSocket connection.
4. Passes answers through the LangGraph workflow.
5. Uses Gemini to extract structured medical information.
6. Finds missing fields and generates the next question.
7. Detects corrections and report-upload intentions.
8. Saves form progress to MongoDB.
9. Transcribes uploaded audio through Faster-Whisper.
10. Stores medical documents in S3.
11. Uses Amazon Bedrock to summarize uploaded reports.
12. Produces a final patient-facing summary.

### Current backend structure

The production backend is in a hybrid refactoring state.

- `DataHandling/server.py` is still the main production entry point and contains much of the active HTTP, WebSocket, MongoDB, audio and upload logic.
- `DataHandling/src/llm/functionalities.py` contains the older `HealthAgent` implementation and remains an LLM wrapper and fallback.
- `DataHandling/src/graph/` contains the newer primary LangGraph interview workflow.
- `DataHandling/app/` contains modular replacements for server functionality, but several are not connected to the production entry point yet.

For current behavior, `server.py` must be treated as the production source of truth.

## 5. End-to-end patient flow

```text
1. Patient opens /{userId}/FRM-01
2. Frontend checks consent status
3. Frontend opens /ws/{clientId}
4. Frontend sends start_interview
5. Backend validates the user in MongoDB
6. Backend loads an existing form or starts a blank form
7. Backend sends the welcome message and first question
8. Patient answers through text or voice
9. Backend extracts medical details with Gemini
10. LangGraph validates missing fields
11. Backend generates the next appropriate question
12. Updated form data is saved to MongoDB
13. Steps 8–12 repeat until the form is complete
14. Backend generates a final summary
15. Patient confirms or corrects the summary
16. Interview phase becomes complete
```

## 6. Voice flow

The voice-processing sequence is:

```text
Patient starts microphone
      ↓
Frontend sends audio_start
      ↓
Frontend streams binary WebM/Opus chunks
      ↓
Patient stops microphone
      ↓
Frontend sends audio_end
      ↓
Backend decodes audio with FFmpeg/Pydub
      ↓
Faster-Whisper transcribes the recording
      ↓
Backend sends a transcription message
      ↓
Frontend displays the text
      ↓
Text is edited/submitted or automatically submitted
```

The transcription itself does not update the form. It must subsequently be sent as a `text_input` WebSocket message.

## 7. Medical intake form

The current form has six fixed sections:

| Section | Information collected |
|---|---|
| Present Complaint | Main complaint, duration, onset and cause |
| Previous Consultations | Previous diagnosis, advice, treatment and current status |
| Pain Assessment | Pain location, severity, aggravating factors and relieving factors |
| History & Diagnostics | Illnesses, surgeries, lifestyle and diagnostic reports |
| Treatment Goals | Short-term goals, long-term goals and treatment expectations |
| Referral | How the patient found Stance Health |

The form contains 17 fields in total.

The interview also identifies the visit context as one of:

- `specific_complaint`
- `general_assessment`
- `clinic_inquiry`
- `unknown`

For a general assessment, complaint-specific fields can be marked as not applicable so that the patient is not asked irrelevant pain questions.

## 8. LangGraph interview workflow

The graph uses the following phases:

- `welcome`
- `interviewing`
- `awaiting_upload`
- `summary`
- `complete`

The simplified workflow is:

```text
welcome
  └── handle first response
        └── interviewing
              ├── extract form data
              ├── detect correction/report intent
              ├── validate section
              ├── ask about missing information
              ├── advance section
              └── generate summary

summary
  ├── confirm → complete
  ├── request correction → update form
  └── reveal new complaint → reopen interview
```

## 9. AI services

### Gemini 2.5 Flash Lite

Primarily used for:

- Conversation flow
- Follow-up question generation
- Intent classification
- Corrections
- Summary generation
- General assistant responses

### Gemini 2.5 Flash

Used as the reasoning-grade model for medical form extraction. It reads conversation context and fills multiple form fields from a natural-language answer.

### Faster-Whisper

The local `base` model handles speech-to-text. It runs with CPU `int8` or GPU `float16`, depending on the host.

### ChromaDB

Used for retrieval-augmented answers from local knowledge collections:

- Stance Health brand information
- Medical/anatomy reference information

### Amazon Bedrock

Used to analyze uploaded medical documents. PDF pages and supported images are sent to an Amazon Nova model, which returns findings, measurements and an impression.

## 10. MongoDB data

Default MongoDB configuration:

```text
Database: stance-dashboard
Collections:
  users
  customer-info
  consentrecords
```

Each user currently has one customer-agent form. Its identity is the combination:

```text
userId + FRM-01
```

A simplified `customer-info` document is:

```json
{
  "userId": "MongoDB user ID",
  "formId": "FRM-01",
  "title": "Derived from the primary complaint",
  "form_data": {},
  "current_section": "Pain Assessment",
  "attachments": [],
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

The backend uses per-connection graph state and deep copies form data to prevent one patient's information from leaking into another patient's session.

## 11. Backend HTTP API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Backend and dependency health |
| GET | `/api/users` | Search or list users |
| GET | `/api/users/{userId}/consent` | Check consent status |
| POST | `/api/users/{userId}/consent` | Record consent |
| GET | `/api/users/{userId}/forms` | Get a user's forms |
| GET | `/api/forms/{formId}?userId={userId}` | Get a particular form |
| GET | `/api/forms/{formId}/progress?userId={userId}` | Get section progress |
| POST | `/api/forms/{formId}/attachments` | Upload medical documents |

## 12. WebSocket protocol

### Main inbound messages

| Message type | Purpose |
|---|---|
| `start_interview` | Start or resume the patient's interview |
| `start_new_form` | Request a new form |
| `load_form` | Load an existing form |
| `text_input` | Submit a typed or transcribed answer |
| `audio_start` | Signal that audio streaming will begin |
| Binary audio | Send recorded audio chunks |
| `audio_end` | Finish audio and start transcription |
| `end_session` | Save and end/pause the session |

### Main outbound messages

| Message type | Purpose |
|---|---|
| `text_message` | Final AI response and interview state |
| `token` | Streaming AI response token |
| `thought_update` | Processing-stage information for the UI |
| `transcription` | Whisper transcription result |
| `form_loaded` | Restored form information |
| `chat_history` | Restored conversation messages |
| `audio_start` / `audio_chunk` | Server-generated speech audio |
| `error` | Structured error response |

## 13. Medical-report upload flow

```text
Patient selects PDF/image files
      ↓
Frontend sends multipart HTTP request
      ↓
Backend verifies form ownership and size
      ↓
Files are stored privately in S3
      ↓
Attachment records are saved in MongoDB
      ↓
API returns ocr_status = processing
      ↓
Background task converts documents to images
      ↓
Amazon Bedrock analyzes the documents
      ↓
Reports field is updated in MongoDB
```

The maximum file size is 15 MB per file. The analysis layer supports PDF, PNG, JPG, JPEG and WebP.

## 14. Question-management status

There is currently no complete question-management system.

The current backend does not provide:

- A questions collection in MongoDB
- An admin question-management UI
- APIs to add, edit or delete questions
- Configurable question categories
- Question priority or ordering management
- Enable/disable controls
- A conditional-question editor
- Question versioning

Instead, the current system works as follows:

```text
Fixed medical form fields
      ↓
System identifies missing fields
      ↓
Gemini generates a conversational question
      ↓
Patient answers
      ↓
Gemini extracts the answer into the fixed form
```

Three broad initial question groups are hardcoded in the backend. Follow-up questions are dynamically generated from the conversation and missing form fields.

### Frontend question UI

The frontend has recently added the ability to display many dynamic question types, including:

- Single choice
- Multiple choice
- Checkbox
- Scale and slider
- Rating
- Text and paragraph
- Number
- Yes/no
- Dropdown
- Date and time
- Likert scale
- Choice and checkbox grids
- File upload
- Multi-answer/PROM questions

However, the current backend does not send the required `question_meta`, `promSteps`, `question_id`, `question_options` or `multi_answer` fields. The frontend capability therefore exists, but it is not integrated with the backend version in this workspace.

## 15. Configuration

### Backend environment

Important backend variables include:

```dotenv
GEMINI_API_KEY=
MONGO_URI=
MONGO_DB_NAME=stance-dashboard
MONGO_USERS_COLLECTION=users

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_REGION=
S3_BUCKET_NAME=

AWS_REGION=
BEDROCK_MODEL_ID=
BEDROCK_INFERENCE_PROFILE=

LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_BASE_URL=
```

### Frontend environment

Important frontend variables include:

```dotenv
VITE_API_URL=
VITE_WS_URL=
VITE_API_KEY=
VITE_GRAPHQL_URL=
```

The GraphQL API is a separate Stance Health service and is not served by the customer-agent backend.

## 16. Local development

### Backend

From `customer-agent/`:

```bash
docker compose up -d --build
docker compose logs -f
```

Health check:

```bash
curl http://localhost:8000/health
```

Smoke test:

```bash
cd DataHandling
python3 tests/smoke.py
```

### Frontend

From `customer-agent-frontend/`:

```bash
npm install
npm run dev
```

The Vite development server runs on port `8080` by default.

## 17. Known gaps and risks

1. **No question-management module:** Questions cannot be managed through an admin interface or database-backed API.
2. **Frontend/backend PROM mismatch:** The frontend supports dynamic/PROM question metadata that the current backend does not produce.
3. **Incomplete backend refactor:** Active logic is duplicated between the monolithic server and newer modules.
4. **Missing frontend source file:** `src/App.tsx` imports `DirectFormPage`, but that file is not present in the inspected frontend checkout.
5. **Limited automated testing:** The backend mainly has a smoke test; there is no comprehensive unit or end-to-end suite.
6. **Runtime verification prerequisites:** The backend requires a valid `.env`, database access, AI credentials and Docker access.
7. **Authorization boundary:** The application passes user IDs through URLs and requests. Authentication and authorization must be enforced by the surrounding platform or implemented explicitly.
8. **In-memory graph history:** Form data is stored in MongoDB, but complete graph execution state is not durably checkpointed.
9. **Background save ordering:** Rapid turns could create ordering risks because some MongoDB saves run in background tasks.
10. **Stale documentation:** The older README describes a deterministic MongoDB question pool that is not implemented in the current code.

## 18. Client-ready explanation

> The project is an AI-powered medical intake system for Stance Health. Patients access the assessment through a secure, patient-specific frontend link and communicate with the Sage assistant using voice or text. The backend uses speech recognition and Gemini AI to understand each response, convert it into structured medical information, identify missing details, and ask appropriate follow-up questions. Interview progress is stored in MongoDB so the patient can continue later. Medical reports can be uploaded to AWS S3 and analyzed through Amazon Bedrock. Once the required information is collected, the system generates a final summary for confirmation and clinician review.

### Question-management explanation

> The application currently does not have an admin-managed question bank. The medical form has six fixed categories, and the AI generates conversational questions based on missing fields and the patient's previous answers. The frontend can display several structured question types, including multi-answer and PROM questions, but the present backend does not yet provide the metadata required to drive those components. A full question-management module would need database collections, CRUD APIs, conditions, ordering, categories and an administrative interface.

## 19. Recommended next steps

1. Fix the missing frontend `DirectFormPage` import/file.
2. Confirm whether PROM/question metadata exists in another backend branch or service.
3. Decide whether questions remain AI-generated or become database-managed.
4. Define authentication and authorization responsibilities across the frontend, backend and main Stance platform.
5. Complete the backend modular refactor.
6. Add unit, protocol, cross-user isolation and full end-to-end tests.
7. Create a shared versioned frontend/backend WebSocket contract.

