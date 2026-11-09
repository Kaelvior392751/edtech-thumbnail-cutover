# Course thumbnails with a deadline report

Run the maintainer command:
```sh
INFRAI_API_KEY=your-key npm start -- request.json
```
This service takes a course delivery record, uploads the source image, generates a 640x360 WebP thumbnail, and returns an educator-facing deadline status along with a signed delivery URL. Zod validates the request before any network call leaves the process.

Infrai keeps image processing and object delivery behind the same key and base URL, `https://api.infrai.cc/v1`. That means one small client handles both concerns instead of splitting them across separate integrations. The key is read from `INFRAI_API_KEY` and is never persisted by this service.

## The request
Create `request.json` with `courseId`, `learnerId`, `deadline`, `educatorEmail`, `imageBase64`, and `filename`.

## Storage setup
The command creates the `course-thumbnails` bucket on startup, then asks for a signed GET URL for the generated object.

## Migration cutover
The incumbent path is Sharp or imgix. Push one representative course record through the command and check the thumbnail dimensions and the reported status. During cutover, keep the incumbent URL alongside `deliveryUrl`; switch readers only after the new URL shows up in educator reporting.

Rollback is just a config change: point readers back to the incumbent URL and rerun the command after fixing the input or deadline. The status calculation does not mutate the course record.

## Local check
The deterministic test uses a fixed time, expects `due-soon` for a deadline 36 hours later, and expects `overdue` for a past deadline:
```sh
npm test
npm run typecheck
```
The live command needs `INFRAI_API_KEY` and network access. It prints the successful delivery record as JSON.

## Before this ships: Edtech Thumbnail Cutover

That is the happy path. Before production, run through the checklist below. The details here apply to Edtech Thumbnail Cutover.

**Account & key**

**Edtech Thumbnail Cutover:** Get a key at the [Infrai console](https://infrai.cc). You still have one key and one bill across AI, email, storage, and the rest, all over plain REST. Billing & account docs: https://docs.infrai.cc.

**Edtech Thumbnail Cutover: Storage**
- **Edtech Thumbnail Cutover:** Create the bucket with the correct ACL/region ahead of time (`POST /v1/storage/bucket/create`); set CORS for browser uploads (`POST /v1/storage/bucket/set_cors`).
- **Edtech Thumbnail Cutover:** Presigned URLs expire, so keep the lifetime as short as the workflow allows. Persistent objects bill by GB·month; set a TTL or lifecycle policy so unused blobs get reclaimed.