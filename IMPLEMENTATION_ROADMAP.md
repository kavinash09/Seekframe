
# Video Chunk Project --- 10-Day POC-to-MVP Roadmap

> **Purpose:** Validate the core video-processing workflow through
> small, independently testable proof-of-concepts (POCs), stitch the
> proven components into an end-to-end local pipeline, and then expose
> the workflow through a minimal application.
>
> **Planning style:** Jira-style epics, stories/tasks, acceptance
> criteria, dependencies, and a 10-day implementation plan.
>
> **Scope note:** Ten days is a focused engineering sprint for a
> demonstrable POC/MVP, not a production-readiness commitment. Keep the
> scope small, use a few representative test videos, and defer
> nonessential features.

---

## 1. Product outcome

A user can upload or provide a video, the system processes it
asynchronously, and the user can search its spoken content and open the
video at the relevant timestamp.

### End-to-end flow

```text
Video input
   |
   v
Inspect and validate media (FFprobe)
   |
   +------------------------------+
   |                              |
   v                              v
Create processing manifest    Store original media
   |                              |
   +---------------+--------------+
                   |
                   v
          Split/process media
          - audio/video segments
          - HLS renditions
                   |
          +--------+---------+
          |                  |
          v                  v
   Speech-to-text        HLS playback assets
          |
          v
   Timestamped transcript
          |
          v
   LLM segment summaries
          |
          v
   Index transcript + summaries
          |
          v
   Search result with video ID and timestamp
          |
          v
   React player opens at matching timestamp
```

For the first local end-to-end run, processing can be orchestrated by a
simple application service or command-line runner. After the workflow is
proven, move stages to the intended AWS event-driven architecture.

---

## 2. Technical decisions and guardrails

### Frontend

- React + TypeScript + Vite.
- React Router, TanStack Query, Zustand, MUI.
- React Hook Form + Zod for forms and validation.
- Video.js with HLS support for playback.
- WebSockets for processing-status updates when the API/application
  stage is introduced.
- Client-rendered SPA only. Do not use Next.js, SSR, or SSG.

### Backend and processing

- Java + Spring Boot for the API, job orchestration, and metadata.
- FFmpeg/FFprobe for media inspection, segment generation, and HLS
  transcoding.
- A speech-to-text API/provider for timestamped transcription.
- An LLM API/provider for segment and full-video summaries.
- Elasticsearch/OpenSearch for transcript and summary search.
- PostgreSQL for video metadata, processing jobs, and stage status.
- AWS target: S3 for media, SQS for stage queues, separate
  workers/Lambdas for stages, and ECS/Fargate or another suitable
  container compute option for long-running/heavy transcoding.

### POC-first principles

1. Prove a component in isolation before integrating it.
2. Use a small set of known test videos and retain sample outputs.
3. Keep stable video IDs, segment IDs, and timestamps across every
   stage.
4. Make processing stages idempotent where practical.
5. Track status and errors explicitly; do not silently skip failed
   stages.
6. Avoid Kubernetes, microservices, and complex orchestration during
   the first local POC.
7. Do not force large/long-running transcodes into Lambda if execution
   or resource limits make a container worker more appropriate.
8. Keep credentials in environment variables or a local secret
   mechanism; never commit API keys.

---

## 3. Jira-style issue hierarchy

Suggested Jira project key: `VCP` (Video Chunk Project). Issue keys
below are proposed planning IDs; replace them with Jira-generated keys
after importing or creating the issues.

### Epic VCP-E1 --- Media inspection and chunking

**Goal:** Reliably inspect an input video and produce a manifest of
usable media segments.

#### VCP-101 --- Build media inspection utility

**Type:** Story
**Priority:** Highest
**Estimate:** 0.5 day
**Description:** Implement a utility that validates a local media file
and extracts metadata with FFprobe.

**Acceptance criteria** - Accepts a local file path. - Fails with a
clear error for a missing, unreadable, or invalid media file. - Extracts
duration, container/format, video codec, width, height, frame rate,
audio codec when present, and file size when available. - Emits a stable
JSON result. - Has automated tests for a valid video, missing file,
invalid file, and a video without an audio stream. - Does not load the
entire video into application memory.

#### VCP-102 --- Generate media segments and manifest

**Type:** Story
**Priority:** Highest
**Estimate:** 0.75 day
**Description:** Use FFmpeg to split a sample video into manageable
segments and record the mapping to the original timeline.

**Acceptance criteria** - Segment duration is configurable. - Segment
IDs and ordering are deterministic for the same input/configuration. -
Manifest records video ID, segment ID, segment path, start time,
duration, and processing status. - Segment start times map to the
original video timeline. - The tool reports FFmpeg failures and missing
outputs. - A test verifies segment count/order and approximate duration
coverage.

### Epic VCP-E2 --- Transcription and AI enrichment

**Goal:** Convert media into timestamped text and useful summaries.

#### VCP-103 --- Integrate speech-to-text

**Type:** Story
**Priority:** Highest
**Estimate:** 1 day
**Description:** Transcribe a sample video or its extracted audio using
a configurable speech-to-text provider.

**Acceptance criteria** - Provider credentials are supplied outside
source control. - Output uses a documented schema with text and
start/end timestamps. - Transcript segments are associated with the
correct video ID. - Chunk offsets are added correctly when transcribing
audio chunks independently. - Provider errors and rate limits produce
actionable errors. - A small sample transcript is checked manually for
timestamp alignment and obvious quality issues.

#### VCP-104 --- Generate segment and full-video summaries

**Type:** Story
**Priority:** High
**Estimate:** 0.75 day
**Description:** Generate a summary for each transcript segment or
logical transcript window, then combine them into a video-level summary.

**Acceptance criteria** - LLM provider/model is configurable. - Summary
output is associated with video ID and source segment IDs. - Empty
transcripts and provider failures are handled explicitly. - Prompt and
output format are documented. - The result includes a segment summary
and a full-video summary. - At least one sample is manually reviewed for
relevance and unsupported claims.

### Epic VCP-E3 --- Search and playback assets

**Goal:** Make processed content discoverable and playable.

#### VCP-105 --- Index transcript and summaries

**Type:** Story
**Priority:** Highest
**Estimate:** 0.75 day
**Description:** Index transcript segments and summaries in
Elasticsearch/OpenSearch.

**Acceptance criteria** - Each indexed document includes video ID,
segment ID where applicable, text, start/end timestamps, and content
type. - Index mapping and index creation are reproducible. - Re-running
indexing for the same document does not create duplicate logical
results. - A query returns the matching text and timestamp. -
Search/indexing errors are observable and retryable. - A basic
integration test verifies indexing followed by a query.

#### VCP-106 --- Generate HLS renditions

**Type:** Story
**Priority:** High
**Estimate:** 1 day
**Description:** Produce HLS output for the sample video, starting with
a small set of renditions to keep the POC bounded.

**Acceptance criteria** - Generates an HLS master playlist and media
playlists. - Starts with two practical resolutions (for example, 360p
and 720p); add 480p and 1080p after the basic path works and
hardware/input constraints are understood. - Output files and playlists
are validated. - Original-video timeline behavior is documented. -
FFmpeg errors and partial output are handled. - The implementation can
later be executed by a container worker in AWS.

#### VCP-107 --- Validate timestamp-based playback

**Type:** Story
**Priority:** High
**Estimate:** 0.5 day
**Description:** Prove that a search result can open a video at the
returned timestamp.

**Acceptance criteria** - A playback URL and timestamp can be passed to
the player. - The player seeks to the intended time within a documented
tolerance. - The behavior works with the generated HLS output in a
supported browser. - A transcript/search result can be mapped to the
correct video. - Failures such as unavailable media produce a useful UI
message.

### Epic VCP-E4 --- Local end-to-end pipeline

**Goal:** Stitch the independently tested components into one
reproducible workflow.

#### VCP-108 --- Create processing manifest and state model

**Type:** Story
**Priority:** Highest
**Estimate:** 0.5 day
**Description:** Define the shared identifiers, schemas, stage states,
and artifact locations used across processing stages.

**Acceptance criteria** - A video has a stable video ID. - Each stage
has an explicit status such as `PENDING`, `RUNNING`, `SUCCEEDED`, or
`FAILED`. - Manifest schema and transcript/search document schemas are
documented. - Each stage records output locations and error details. -
Timestamps consistently use seconds or a clearly documented
alternative. - Re-running a stage can identify existing outputs and
avoid accidental duplication.

#### VCP-109 --- Build local pipeline runner

**Type:** Story
**Priority:** Highest
**Estimate:** 1 day
**Description:** Add a single command or application entry point that
runs inspection, chunking, transcription, summarization, indexing, and
HLS generation for one input video.

**Acceptance criteria** - One documented command starts the pipeline. -
Each stage consumes the prior stage's output rather than relying on
hidden local state. - Pipeline logs identify video ID, stage, duration,
and failure. - A failed stage is clearly reported and does not mark the
full job successful. - The runner produces a final processing report
with artifact paths. - A small sample video completes end to end.

### Epic VCP-E5 --- Minimal application integration

**Goal:** Expose the validated workflow through a basic user-facing
application.

#### VCP-110 --- Implement Java API and metadata persistence

**Type:** Story
**Priority:** High
**Estimate:** 1 day
**Description:** Add the minimal Spring Boot API and PostgreSQL
persistence required to submit a video job and inspect its status.

**Acceptance criteria** - API can create a video/job record and return a
stable ID. - API exposes job status and basic video metadata. - Request
validation and error responses are consistent. - Processing does not run
synchronously inside a long-lived upload/API request. - Database schema
and local setup instructions are included. - The local runner can be
invoked through an application service or a clear integration boundary.

#### VCP-111 --- Build minimal React workflow

**Type:** Story
**Priority:** High
**Estimate:** 1 day
**Description:** Build a basic SPA to submit/select a video, show
processing status, search processed content, and play a result at its
timestamp.

**Acceptance criteria** - Frontend uses React + TypeScript + Vite and
the agreed client-side libraries. - User can submit a sample job or
select a previously processed video. - UI displays processing state and
actionable errors. - Search results show matching text and timestamp. -
Selecting a result opens the player at the matching timestamp. -
Loading, empty, and error states are implemented. - Real-time WebSocket
status can be added behind a small status-update abstraction; polling is
acceptable for the first local MVP if WebSocket integration would block
the end-to-end demonstration.

### Epic VCP-E6 --- AWS migration plan and validation

**Goal:** Prepare the proven local workflow for the intended cloud
architecture without making cloud deployment a blocker for the first
end-to-end demo.

#### VCP-112 --- Define AWS stage boundaries and queue contracts

**Type:** Task
**Priority:** Medium
**Estimate:** 0.5 day
**Description:** Document how the local stages map to S3, SQS, Lambda,
and container workers.

**Acceptance criteria** - S3 artifact layout is documented. - Each
queue's message schema includes video ID, stage, input references, and
correlation/idempotency information. - Each processing stage has a
defined owner and retry/failure behavior. - Heavy transcoding is
assigned to an appropriate container-based worker when needed. -
Dead-letter queue and observability requirements are documented. - Local
interfaces avoid hard-coding cloud-specific logic into the core
processing functions.

---

## 4. Ten-day roadmap

The plan assumes one primary engineer, existing access to the required
APIs/tools, a local development environment, and a small test dataset.
Some work can overlap, but integration and validation must remain
explicit.

---

  Day               Focus             Tickets           End-of-day deliverable

---

  1                 Inspect input     VCP-101, start    FFprobe utility, metadata JSON,
                    video             VCP-108           initial shared schema

  2                 Chunking and      VCP-102, finish   Segments, timeline mapping,
                    manifest          VCP-108           processing manifest

  3                 Speech-to-text    VCP-103           Timestamped transcript JSON

  4                 Summarization and VCP-104, VCP-105  Summaries indexed and
                    search                              searchable

  5                 HLS generation    VCP-106           Valid HLS output with a small
                                                        set of renditions

  6                 Stitch local      VCP-109, VCP-107  One command processes a video;
                    pipeline                            timestamp playback proven

  7                 Java API and      VCP-110           Spring Boot API, PostgreSQL
                    persistence                         metadata, job status

  8                 React SPA         VCP-111           Basic
                                                        submit/status/search/playback
                                                        experience

  9                 Integration and   Finish VCP-107,   End-to-end demo, regression
                    failure testing   VCP-109, VCP-110, tests, failure-path fixes
                                      VCP-111

10                Stabilize and     VCP-112 plus bug  README/runbook, demo checklist,
                    cloud plan        fixes             AWS migration contracts,
                                                        prioritized backlog
---------------------------------------------------------------------------

**Scope control:** If the schedule slips, protect the end-to-end
search-and-playback path. Defer extra resolutions, advanced
semantic/vector search, WebSocket updates, authentication,
multi-tenancy, and cloud deployment rather than leaving the core
workflow disconnected.

---

## 5. Shared data contracts

Keep these contracts small and versioned. Exact implementation types may
evolve, but the meaning of identifiers and timestamps must not change
between stages.

### Video manifest (illustrative)

```json
{
  "schemaVersion": 1,
  "videoId": "video-123",
  "source": {
    "path": "./data/input/sample.mp4",
    "durationSeconds": 600.0
  },
  "segments": [
    {
      "segmentId": "video-123-seg-0001",
      "startSeconds": 0.0,
      "durationSeconds": 30.0,
      "mediaPath": "./data/segments/video-123/0001.mp4",
      "status": "SUCCEEDED"
    }
  ],
  "stages": {
    "inspection": "SUCCEEDED",
    "chunking": "SUCCEEDED",
    "transcription": "PENDING",
    "summarization": "PENDING",
    "indexing": "PENDING",
    "transcoding": "PENDING"
  }
}
```

### Transcript segment (illustrative)

```json
{
  "videoId": "video-123",
  "segmentId": "video-123-seg-0001",
  "startSeconds": 12.4,
  "endSeconds": 17.8,
  "text": "The scheduler assigns pods to suitable nodes."
}
```

**Timestamp rule:** Prefer absolute timestamps relative to the original
video timeline for search and playback. If a transcription provider
returns timestamps relative to an audio chunk, add the chunk's original
start offset before persisting or indexing the transcript.

### Search document (illustrative)

```json
{
  "documentId": "video-123-transcript-00042",
  "videoId": "video-123",
  "segmentId": "video-123-seg-0002",
  "contentType": "TRANSCRIPT",
  "startSeconds": 72.1,
  "endSeconds": 78.6,
  "text": "Kubernetes schedules pods onto nodes."
}
```

---

## 6. Suggested repository structure

Start with a simple monorepo. Avoid splitting into multiple repositories
before the component boundaries are proven.

```text
video-chunk-project/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── data-contracts.md
│   └── aws-migration.md
├── samples/
│   └── README.md
├── scripts/
│   ├── inspect-video.sh
│   └── run-local-pipeline.sh
├── processing/
│   ├── README.md
│   ├── inspection/
│   ├── chunking/
│   ├── transcription/
│   ├── summarization/
│   ├── transcoding/
│   └── indexing/
├── backend/
│   └── README.md
├── frontend/
│   └── README.md
├── schemas/
│   ├── video-manifest.schema.json
│   ├── transcript-segment.schema.json
│   └── search-document.schema.json
├── tests/
│   ├── fixtures/
│   ├── integration/
│   └── e2e/
└── .env.example
```

The processing implementation can use Java and FFmpeg bindings or invoke
FFmpeg/FFprobe as subprocesses from Java. Keep the first approach simple
and record the choice in `processing/README.md`. Do not commit real
media files, credentials, generated output, or provider responses
containing sensitive data.

---

## 7. Definition of done

A ticket is complete only when: - Its acceptance criteria are met. - A
repeatable test or manual verification procedure is documented. -
Expected outputs and failure cases are recorded. - Errors are visible
and actionable. - Relevant documentation is updated. - No secrets are
committed.

The 10-day roadmap is complete when a developer can follow the README to
run the application locally, process a sample video, search its
transcript, and open the video at the matching timestamp.

### Final demo checklist

- [ ] Inspect a valid sample video and reject an invalid file.
- [ ] Produce segments and a manifest with correct timeline offsets.
- [ ] Generate a timestamped transcript.
- [ ] Generate segment and full-video summaries.
- [ ] Index transcript and summaries.
- [ ] Search for a phrase and receive a timestamped result.
- [ ] Generate valid HLS output and play it in the browser.
- [ ] Select a search result and seek to its timestamp.
- [ ] Display job status and errors in the application.
- [ ] Re-run processing without creating duplicate logical search
  documents.
- [ ] Document the next steps for moving each stage to S3/SQS and the
  appropriate worker.

---

## 8. Deferred backlog (after the 10-day MVP)

1. AWS infrastructure-as-code and S3 upload flow.
2. Separate SQS queues and workers for each processing stage.
3. Retry policies, dead-letter queues, idempotency, and recovery from
   partial failure.
4. API Gateway WebSocket status updates.
5. 480p and 1080p renditions, encoding profiles, and thumbnail
   generation.
6. Semantic/vector search and hybrid ranking.
7. Authentication, authorization, quotas, and multi-tenant isolation.
8. Observability dashboards, cost metrics, alarms, and operational
   runbooks.
9. Large-file, long-duration, malformed-media, and load testing.
10. CI/CD, deployment environments, backups, and production security
    review.

---

## 9. Risks and mitigation

---

  Risk                                Mitigation

---

  Transcription or LLM API access is  Use a provider adapter and a small
  unavailable                         saved fixture for development
                                      tests; clearly distinguish mocked
                                      tests from live-provider
                                      validation.

  FFmpeg output differs across input  Begin with a documented supported
  formats                             input set and test representative
                                      codecs.

  Chunk-relative timestamps cause     Normalize all timestamps to the
  wrong search jumps                  original video timeline at the
                                      boundary.

  Transcoding takes too long for      Use a container worker for
  Lambda                              heavy/long-running jobs.

  Reprocessing creates duplicate      Use deterministic document IDs and
  search results                      idempotent indexing.

  One failed stage leaves the job     Persist per-stage state and error
  stuck                               details; define retry and
                                      terminal-failure behavior.

Ten days becomes too ambitious      Keep the local end-to-end pipeline
                                      as the priority; defer
                                      production/cloud hardening and
                                      advanced features.
--------------------------------------------------------

---

## 10. First action

Start with **VCP-101: Build media inspection utility**.

1. Install and verify FFmpeg/FFprobe locally.
2. Select two or three small, legally usable sample videos with
   different durations and audio characteristics.
3. Implement metadata extraction and JSON output.
4. Add tests for valid, missing, invalid, and audio-less inputs.
5. Save a sample output and record the exact run command.
6. Move to VCP-102 only after inspection is repeatable.

This keeps the project incremental: prove each stage, define its
contract, integrate it, and then replace local execution with cloud
workers without redesigning the whole application.
