# ADR-0005: Store Generated Reports as PostgreSQL bytea

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

SprintPilot generates binary report files (PPTX, DOCX) during chat sessions. These files need to be persisted so users can download them after the session, and so they are accessible across future sessions. Options for storage:

1. **PostgreSQL `bytea` column** — store binary content directly in the database alongside session and message data.
2. **Container filesystem** — write files to a Docker volume. Simple but volatile; files are lost if the container is replaced without persistent volume mounts.
3. **Azure Blob Storage** — store files in Azure storage, save the blob URL in PostgreSQL. Durable and scalable, but introduces a second storage dependency, additional auth configuration, and more complex deployment.
4. **Azure Files / NFS share** — shared filesystem mounted in the container. Similar complexity to Blob Storage; more operational overhead.

Key factors:

- Generated reports are expected to be small: typical sprint review PPTX ≤ 5MB; hard limit set at 20MB.
- The FastAPI template already provides a configured PostgreSQL instance with backup and transactional guarantees.
- OIT's deployment target is Azure Container Apps, where ephemeral container filesystems are expected.
- Introducing Azure Blob Storage adds procurement, IAM configuration, and connection string management for a problem that PostgreSQL already solves at SprintPilot's scale.

## Decision

Store generated reports as **`bytea` columns in PostgreSQL**, in the `GeneratedReport` table. The `file_data` column holds the raw binary content. A download endpoint (`GET /api/v1/reports/{report_id}/download`) streams the content with appropriate `Content-Type` and `Content-Disposition` headers, gated to the report owner.

A per-file size guard of **20MB** is enforced at generation time. Reports exceeding this limit are rejected before the database write.

## Consequences

**Positive:**
- No additional storage service; reports are transactional with session data and covered by existing PostgreSQL backup procedures.
- Simplified deployment: no blob storage credentials, connection strings, or IAM roles needed.
- Download endpoint is straightforward streaming from the ORM; no pre-signed URL complexity.
- Retention and access control are handled by the same auth/ownership model as chat sessions.

**Negative / Risks:**
- Large reports inflate database size. At SprintPilot's expected scale (tens to low hundreds of reports), this is not a concern. A future cleanup job should prune reports older than a configurable threshold.
- If report volumes grow significantly (hundreds of active users, reports retained for years), migrating to Blob Storage would require a data migration. The `GeneratedReport` model's `file_data` field would be replaced with a `blob_url` field; the download endpoint would redirect rather than stream.
- PostgreSQL `bytea` is not suitable for files above a few hundred MB — but the 20MB limit prevents reaching that boundary.
