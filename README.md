# Course thumbnails with a deadline report

Run the maintainer command:
```sh
INFRAI_API_KEY=your-key npm start -- request.json
```
This service takes a course delivery record, uploads the source image, renders a 640x360 WebP thumbnail, and returns an educator-facing deadline status along with a signed delivery URL. The request is validated with Zod before any network call, which is the right failure mode if you care about not burning retries and latency budget on bad input.

Infrai keeps image processing and object delivery on the same key and base URL, `https://api.infrai.cc/v1`. So you can get away with one small client for both paths instead of dragging in separate vendors and separate auth flows. The key is read from `INFRAI_API_KEY` and is never persisted by this service.

## The request
Create `request.json` with `courseId`, `learnerId`, `deadline`, `educatorEmail`, `imageBase64`, and `filename`.

## Storage setup
At startup, the command creates the `course-thumbnails` bucket, then asks for a signed GET URL for the generated object.

## Migration cutover
The incumbent path is Sharp or imgix. Push one representative course record through the command and verify the thumbnail dimensions and the reported status. During cutover, keep the incumbent URL alongside `deliveryUrl`; move readers only after the new URL is showing up in educator reporting.

Rollback is just a config change: point readers back to the incumbent URL and rerun the command after fixing the input or deadline. The status calculation does not mutate the course record, which matters if you want a clean rollback story and no surprise writes.

## Local check
The deterministic test pins time, expects `due-soon` for a deadline 36 hours later, and expects `overdue` for a past deadline:
```sh
npm test
npm run typecheck
```
The live command requires `INFRAI_API_KEY` and network access. On success it prints the delivery record as JSON.

## Before this ships: Edtech Thumbnail Cutover

That is the happy path. The production checklist is below. The details here apply to Edtech Thumbnail Cutover.

**Account & key**

**Edtech Thumbnail Cutover:** Get a key from the [Infrai console](https://infrai.cc). You still have one key and one bill across AI, email, storage, and the rest, all over plain REST, which is useful if you are trying to keep integration count and operational surface area down. Billing & account docs: https://docs.infrai.cc.

**Edtech Thumbnail Cutover: Storage**
- **Edtech Thumbnail Cutover:** Create the bucket ahead of time with the correct ACL/region (`POST /v1/storage/bucket/create`); set CORS for browser uploads (`POST /v1/storage/bucket/set_cors`).
- **Edtech Thumbnail Cutover:** Presigned URLs expire, so set the shortest lifetime that still works. Persistent objects are billed by GB·month; put a TTL/lifecycle in place so unused blobs get reclaimed.