# File Upload Security Assessment

Date: 2026-01-25

## Scope
- API: `apps/api/src/routers/upload.ts`, `apps/api/src/lib/s3.ts`, `apps/api/src/lib/trpc.ts`, `apps/api/prisma/schema.prisma`
- Mobile app: `apps/mobile/app/file-uploads/s3.tsx`, `apps/mobile/lib/upload/multipart.ts`, `apps/mobile/lib/trpc/client.ts`

## Data Flow Summary
1. Mobile requests a presigned S3 URL via `trpc.upload.requestUploadUrl` or multipart init.
2. Mobile uploads directly to S3 via the presigned URL.
3. Mobile confirms upload via `trpc.upload.confirmUpload` or `completeMultipart`.
4. API stores metadata in `File` table scoped to `userId`.

## What Prevents Cross-User Access
- All upload endpoints are behind `protectedProcedure` and require an authenticated session.
- File operations verify ownership (`file.userId === ctx.user.id`) before allowing actions.
- S3 object keys are generated as `uploads/{userId}/{uuid}.{ext}` and stored per-user in DB.
- There are no download/read endpoints or presigned download URLs in the API, so the app cannot request someone else's file data via the API.

## Strengths Observed
- **Auth + ownership checks** on every file operation in API.
- **Per-user file isolation** in DB queries and S3 key format.
- **Presigned URL expiry** (10 minutes) for upload URLs.
- **MIME type allowlist** and **size limits** on upload requests.
- **Multipart support with tracked parts** and server-side completion step.

## Gaps / Risks
- **No server-side verification that upload exists or matches metadata** when `confirmUpload` is called. The server trusts the client that S3 received the object.
- **MIME type is client-provided only.** No server-side file content inspection or magic-byte validation.
- **No malware/virus scanning** on uploaded content.
- **Storage quota check is non-atomic.** Parallel uploads can exceed the quota.
- **Soft delete continues when S3 deletion fails.** This can leave orphaned objects and unexpected storage exposure.
- **Bucket policy/ACLs not enforced in code.** The code assumes S3 is private, but policies are not validated here.
- **Presigned URL leakage risk.** Any leaked upload URL can be used to upload to that object key until expiry.

## Recommendations
1. Add a **download endpoint** that returns presigned **GET** URLs and verify ownership at request time.
2. Add **server-side integrity checks** on confirm (e.g., `HeadObject` to validate size and content type).
3. Add **file content validation** (magic-byte checks) and/or **AV scanning** for uploaded files.
4. Enforce **atomic quota checks** (transaction with `SELECT ... FOR UPDATE` or pre-reserved quota).
5. Add **cleanup jobs** for failed/abandoned uploads and S3 delete failures.
6. Document and enforce **bucket policy** (deny public access, restrict to server credentials, block public ACLs).
7. Consider **upload URL scope tightening** (shorter expiry or signed conditions) for sensitive flows.

## Conclusion
The current API and mobile upload flows are reasonably secure for **upload isolation**: only authenticated users can create upload URLs, and all API actions verify file ownership. There is no API path to download another user's file. The main risks are around **post-upload verification, malware scanning, and quota enforcement**, along with reliance on **external S3 bucket policy** for absolute privacy. Addressing the gaps above would strengthen the project’s mission of teaching secure file storage.
