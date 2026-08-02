---
title: File Uploads And File Managers
description: Use this page when the task mentions file uploads, file managers, file pickers, upload progress, public storage paths, or BrowserSync-safe upload directories in Caspian.
related:
  title: Related docs
  description: Use fetch-data for the route versus RPC split, PulsePoint for client runtime behavior, database for Prisma persistence, project structure for file placement, validation for upload guards, and commands for local watcher behavior.
  links:
    - /docs/fetch-data
    - /docs/pulsepoint
    - /docs/database
    - /docs/project-structure
    - /docs/validation
    - /docs/commands
    - /docs/index
---

This page documents the recommended file upload and file manager pattern for Caspian projects.

Treat uploads as normal route behavior. Keep the owning browser UI in the route template, keep upload and delete `@rpc()` actions in the owning route `index.py`, and move reusable storage or persistence helpers into `src/lib/`.

If the file manager lives inside a grouped subtree such as a dashboard, account area, or admin area, apply the section layout pattern from [routing.md](./routing.md): keep the shared shell in the parent folder's `layout.py` and keep the file-manager route itself in a child folder.

## Source Of Truth

- File picker UI and upload progress state belong in the route's page template inside `src/app/**/index.py`.
- Upload, delete, and list refresh actions belong in the owning `src/app/**/index.py` as `@rpc()` actions.
- Reusable storage, naming, and persistence helpers belong in `src/lib/**`.
- Durable metadata belongs in Prisma when `caspian.config.json` enables Prisma.
- Browser-side progress, state replacement, and `pp-for` list rendering follow [pulsepoint-runtime-map.md](./pulsepoint-runtime-map.md).
- BrowserSync upload ignore behavior belongs in `settings/bs-config.ts`.

## Default Pattern

- Keep first-render file manager data in `page()` so the initial HTML already contains the current asset list and storage summary.
- Keep upload and delete actions in the route's `index.py`; do not move ordinary upload flows into `main.py`.
- Keep reusable file-manager helpers in `src/lib/`.
- Store uploaded blobs under a project-owned public directory such as `public/uploads/...` when the files should be browser-accessible.
- Confirm that the directory's top-level public name is present in `PublicFilesMiddleware.inline_safe_subdirectories` before storing untrusted uploads there. The conventional `public/uploads/**` path uses the `uploads` mapping.
- Create the upload directory on demand in the shared helper when it does not exist yet; do not assume the folder is committed.
- Store durable metadata in Prisma, not in JSON manifests or ad hoc metadata files.
- Use `pp.state(...)` plus `pp-for` to render and update the file list from returned server payloads.
- Use `onUploadProgress` only for progress UI; let the RPC return refreshed manager data for the authoritative post-upload state.

## Recommended Placement

| Concern | Preferred location | Notes |
| --- | --- | --- |
| File picker UI, tabs, filters, progress state | the page template in `src/app/**/index.py` | Use PulsePoint state and template rendering. |
| Upload and delete `@rpc()` actions | `src/app/**/index.py` | Keep these route-local so they stay close to the owning page. |
| Shared storage, normalization, and persistence helpers | `src/lib/**` | Reuse helpers across routes without pushing route behavior into app bootstrap. |
| Upload metadata model | `prisma/schema.prisma` | Persist owner, file name, MIME type, path, size, collection, and timestamps in Prisma. |
| Browser-accessible uploaded blobs | `public/uploads/**` or another explicitly protected public directory | Keep the public path predictable and derived from stored metadata, create it on demand, and configure its top-level name in `PublicFilesMiddleware.inline_safe_subdirectories` before accepting untrusted files. |
| BrowserSync upload ignore | `settings/bs-config.ts` | Keep the active public upload directory in `PUBLIC_IGNORE_DIRS`. |

## Route Flow

The normal Caspian file-manager flow is:

1. `page()` calls a shared helper such as `build_file_manager_payload(...)` and renders the first payload into the route template.
2. The route template hydrates that payload into `pp.state(...)`.
3. The file input calls `pp.rpc(...)` with a `File` object and optional `onUploadProgress` callback.
4. The route-local `@rpc()` action validates auth, file presence, size, and routing-specific inputs.
5. A shared helper writes the blob, creates or updates the Prisma row, and returns a serialized asset payload.
6. The RPC action returns refreshed manager data.
7. PulsePoint replaces the route state and `pp-for` rerenders the list.

Section example:

```text
src/
  app/
    dashboard/
      layout.py
      files/
        index.py
        index.py
```

In that pattern, `dashboard/layout.py` owns the shared dashboard shell, while `dashboard/files/index.py` owns the initial file-manager payload and the upload or delete `@rpc()` actions.

## Example Route-Local RPC

```python
from fastapi import File, UploadFile
from casp.rpc import rpc

from src.lib.dashboard.file_manager import (
    build_file_manager_payload,
    save_file_manager_upload,
)

@rpc(require_auth=True)
async def upload_asset(file: UploadFile = File(...), collection: str = "auto"):
    user_id = current_user_id()
    content = await file.read()

    uploaded = await save_file_manager_upload(
        user_id,
        file_name=file.filename or "file",
        content=content,
        mime_type=file.content_type,
        desired_group=collection,
    )

    return {
        "success": True,
        "uploaded": uploaded,
        "data": await build_file_manager_payload(user_id),
    }
```

Keep route-specific auth checks, input validation, and final response shape here. Keep file naming, directory choice, and Prisma persistence in the shared helper.

## Client Pattern

```html
<section class="files-page">
  <input type="file" multiple onchange="{uploadSelectedFiles(event.target.files)}" />

  <template pp-for="asset in assets">
    <article key="{asset.id}">
      <a href="{asset.url}">{asset.name}</a>
    </article>
  </template>

  <script>
    const [fileManagerData, setFileManagerData] = pp.state(initialFileManagerData);
    const assets = fileManagerData.assets ?? [];

    async function uploadSelectedFiles(fileList) {
      for (const file of Array.from(fileList ?? [])) {
        const response = await pp.rpc("upload_asset", { file }, {
          onUploadProgress: (progress) => {
            console.log(progress.percent ?? 0);
          },
        });

        if (response?.data) {
          setFileManagerData(response.data);
        }
      }
    }
  </script>
</section>
```

`onUploadProgress` receives `{ loaded, total, percent }`; `total` and `percent` may be `null` when the browser cannot compute the upload length. A multipart RPC writes companion non-file values before all `File`/`FileList` parts, so route parameters are available even when the server begins processing a streamed upload before the full request body arrives. A `FileList` is sent as repeated parts under one field name.

Prefer this state-driven shape over manual `innerHTML` list painting. In Caspian pages, manual DOM writes are easy to lose when PulsePoint rerenders the owner template.

## Persistence Rules

When Prisma is enabled, the durable record of an uploaded file should live in the database.

- Write the blob to disk or object storage.
- Write the metadata to Prisma.
- Derive public URLs from the stored path.
- On delete, remove both the stored file and its Prisma row.
- Treat old JSON manifests only as migration input, not as the primary store.

Typical metadata fields include:

- owner or user relation
- original file name
- stored file name
- asset path
- MIME type
- collection or grouping key
- kind or preview type
- size in bytes
- uploaded timestamp

## Public-Serving Security

Every existing file under `public/**` is reachable at the matching
root-relative URL without a directory-specific route. That is correct for
trusted first-party assets, but runtime uploads cross a different trust
boundary. In `main.py`, configure each top-level upload directory with the
allow-list used by `PublicFilesMiddleware`:

```python
app.add_middleware(
    PublicFilesMiddleware,
    directory="public",
    inline_safe_subdirectories={
        "uploads": INLINE_SAFE_UPLOAD_MEDIA_TYPES,
    },
)
```

The resolver rejects traversal and symlink escape. Within configured upload
directories, allow-listed raster image types render inline; HTML, SVG, unknown,
and other executable or unsafe types download as
`application/octet-stream` with `Content-Disposition: attachment` and
`X-Content-Type-Options: nosniff`. Do not place untrusted uploads in a different
top-level public directory until that directory is added to the mapping.

## BrowserSync And Uploaded Public Files

If runtime uploads write into `public/uploads/`, BrowserSync should ignore that directory during local development. Otherwise every upload can trigger a full browser reload.

The helper that stores uploaded files should create `public/uploads` before writing the first file when the directory is not present yet.

Use a public-root-relative ignore entry in `settings/bs-config.ts`:

```ts
const PUBLIC_IGNORE_DIRS = ["uploads"];
```

`settings/bs-config.ts` matches those entries against paths relative to the `public/` root, so nested uploads such as `uploads/user-1/media/example.png` are ignored too.

## What To Avoid

- Do not add ordinary upload routes or file-manager glue to `main.py`.
- Do not use JSON files under `storage/` as the active metadata store when Prisma is enabled.
- Do not treat `onUploadProgress` callbacks as the source of truth for the final asset list.
- Do not manually repaint the file list with `innerHTML` when PulsePoint state can own the list.
- Do not leave uploaded public directories out of BrowserSync ignore rules.
- Do not store untrusted runtime uploads in an unconfigured top-level public directory; generic first-party public files render inline.

## AI Retrieval Notes

- Read this page first when the task mentions uploads, file pickers, file managers, media libraries, or upload progress UI.
- Use `fetch-data.md` for the route-render versus RPC split.
- Use `database.md` when Prisma models or relations must change for upload metadata.
- Use `validation.md` for MIME, extension, and other boundary checks, then keep explicit size and auth checks in the owning RPC action.
- Use `project-structure.md` for placement rules, especially `src/app/` versus `src/lib/` and `public/uploads/`; verify public serving and the restricted-inline mapping in `main.py` plus `casp.runtime_security`.
- For grouped file-manager sections, follow the section layout pattern in [routing.md](./routing.md) and keep the upload page in a child route folder beneath the shared shell.
- Use `commands.md` and `settings/bs-config.ts` when uploads should not trigger BrowserSync reloads during `npm run dev`.
