# Content Platform — Page Fetching Architecture

This document describes how `packages/core` fetches page content from the **VTEX Content Platform** (the newer CMS system, backed by the `@vtex/client-cp` package).

The system is active when `contentSource.type` is set to `'cp'` in `discovery.config.js`.

> **Note:** The legacy Headless CMS system (active when `contentSource.type` is `'CMS'`) is documented separately in [`headless-cms.md`](./headless-cms.md). Both systems share the same `ContentService` interface, so switching between them only requires changing the `contentSource.type` value in the config — no page or component code needs to change.

---

## Table of Contents

- [Configuration](#configuration)
- [Content Platform Client](#content-platform-client)
- [ContentService — the shared facade](#contentservice--the-shared-facade)
- [How the active system is selected](#how-the-active-system-is-selected)
- [Fetch methods and their Content Platform paths](#fetch-methods-and-their-content-platform-paths)
  - [getSingleContent](#getsinglesontent)
  - [getMultipleContent](#getmultiplecontent)
  - [getPlpContent](#getplpcontent)
  - [getPdpContent](#getpdpcontent)
- [Preview Mode](#preview-mode)
- [Data Flow](#data-flow)

---

## Configuration

Relevant fields in `discovery.config.js`:

```js
// Activates the Content Platform path in ContentService
contentSource: {
  type: 'cp',  // any value whose lowercase equals 'cp' enables Content Platform
},

api: {
  storeId: 'storeframework', // VTEX account name — passed as `tenant` to ClientCP
},
```

The `cms.data` / `CMS_DATA` environment variable and `contentSource.project` (builder project name) are specific to the legacy Headless CMS and are **not** used by the Content Platform path.

---

## Content Platform Client

**File:** `src/server/content/service.ts`

A `ClientCP` instance (from `@vtex/client-cp`) is created once per `ContentService` construction and reused across all fetch calls:

```ts
this.clientCP = new ClientCP({
  tenant: config.api.storeId,
})
```

The client exposes the following methods used by `ContentService`:

| `ClientCP` method | When used |
|---|---|
| `listEntries(params)` | List entry metadata for a content type |
| `getEntry(params)` | Fetch a single entry by `entryId` |
| `getEntryBySlug(params)` | Fetch a single entry by `slug` |
| `previewEntryById(params)` | Fetch a draft entry by `entryId` (preview mode) |
| `previewEntryBySlug(params)` | Fetch a draft entry by `slug` (preview mode) |

---

## ContentService — the shared facade

**File:** `src/server/content/service.ts`

`ContentService` is the single class through which every page fetches its CMS content, regardless of which backend is active. Pages and templates always call `contentService.*` methods and never talk to `ClientCMS` or `ClientCP` directly.

```ts
export class ContentService {
  async getSingleContent<T>(params: ContentParams): Promise<T>
  async getMultipleContent(params: ContentParams): Promise<{ data: ContentResult[] }>
  async getPlpContent(params, rewrites): Promise<PLPContentType>
  async getPdpContent(product, params): Promise<PDPContentType>
}

export const contentService = new ContentService() // singleton used by all pages
```

The `ContentParams` interface accepted by every public method:

```ts
interface ContentParams {
  contentType?: string
  previewData?: PreviewData | null
  slug?: string
  documentId?: string   // maps to entryId in Content Platform
  versionId?: string    // maps to branchId in Content Platform
  releaseId?: string    // maps to branchId in Content Platform
  filters?: Record<string, unknown>
}
```

---

## How the active system is selected

**File:** `src/server/content/utils.ts`

```ts
export function isContentPlatformSource(): boolean {
  return contentSource.type.toLocaleLowerCase() === 'cp'
}
```

Every public `ContentService` method calls `isContentPlatformSource()` as its first step. When it returns `true` the Content Platform path is executed; otherwise execution falls through to the legacy Headless CMS helpers (`getPage` / `getCMSPage` from `src/server/cms/index.ts`).

---

## Fetch methods and their Content Platform paths

### `getSingleContent`

Used by: home page, landing pages, global sections.

```
contentService.getSingleContent({ contentType, previewData, documentId?, slug?, ... })
  └── [CP path] ContentService.getFromCP(options)
        └── ContentService.getEntry(params, isPreview)
              ├── [has entryId or slug] ContentService.getEntryData(params, isPreview)
              │     ├── [preview=false, entryId] clientCP.getEntry(params)
              │     ├── [preview=false, slug]    clientCP.getEntryBySlug(params)
              │     ├── [preview=true,  entryId] clientCP.previewEntryById(params)
              │     └── [preview=true,  slug]    clientCP.previewEntryBySlug(params)
              └── [no entryId or slug] ContentService.fetchFirstEntryFromList(params, isPreview)
                    └── clientCP.listEntries(params)   ← fetches entry list
                          └── getEntryData({ entryId: entries[0].id }, isPreview)
```

`isPreview` is `true` when either:
- `previewData.contentType === contentType` (content-specific preview), or
- `isBranchPreview(previewData)` — i.e. Content Platform is active **and** `previewData` carries a `versionId` or `releaseId`.

### `getMultipleContent`

Used by the PLP and PDP paths when Content Platform is active to retrieve all templates before template-selection.

```
contentService.getMultipleContent({ contentType, ... })
  └── [CP path] clientCP.listEntries(params)   ← returns entry stubs
        └── ContentService.fillEntriesWithData(entries, params, isPreview)
              └── for each entry:
                    ContentService.getEntryData({ entryId: entry.id }, isPreview)
                      ├── [preview=false] clientCP.getEntry(params)
                      └── [preview=true]  clientCP.previewEntryById(params)
```

The result is a `{ data: (ContentEntry & PageContentType)[] }` list where each element merges the stub metadata with the full entry payload.

### `getPlpContent`

**Entry point:** called from `getStaticProps` in `src/pages/[...slug].tsx`.

```
contentService.getPlpContent({ previewData, slug }, rewrites)
  └── [CP path] contentService.getMultipleContent({ contentType: 'plp', ... })
        └── (see getMultipleContent above)
  └── findBestPLPTemplate(pages, slug, rewrites)   ← selects the most specific template
```

### `getPdpContent`

**Entry point:** called from `getStaticProps` in `src/pages/[slug]/p.tsx`.

```
contentService.getPdpContent(product, { previewData, slug })
  └── [CP path] contentService.getMultipleContent({ contentType: 'pdp', ... })
        └── (see getMultipleContent above)
  └── findBestPDPTemplate(pages, product)   ← selects the most specific template
```

---

## Parameter mapping — `ContentParams` → `EntryPathParams`

When the Content Platform path is taken, `ContentService.convertOptionsToParams()` translates the shared `ContentParams`/`Options` shape into the `EntryPathParams` expected by `ClientCP`:

| `ContentParams` / `Options` field | `EntryPathParams` field |
|---|---|
| `config.api.storeId` | `accountName` |
| `'faststore'` (hardcoded) | `storeId` |
| `contentType` | `contentType` |
| `slug` | `slug` |
| `documentId` | `entryId` |
| `versionId` | `branchId` |
| `releaseId` | `branchId` |
| `filters['settings.seo.slug']` | `slug` (leading `/` stripped) |

---

## Preview Mode

**File:** `src/pages/api/preview.ts`

The preview API route behaves identically for both backends: it calls `contentService.getSingleContent(...)` to validate the locator, then sets the Next.js preview cookie via `res.setPreviewData(locator)`.

The Content Platform-specific behaviour is in the redirect step:

```ts
if (isContentPlatformSource() && slug && ['landingPage', 'plp', 'pdp'].includes(locator.contentType)) {
  return setPreviewAndRedirect(res, locator, slug)
}
```

When Content Platform is active and a `slug` query parameter is supplied for `landingPage`, `plp`, or `pdp` content types, the handler redirects directly to that slug instead of looking it up from `previewRedirects` in the config.

`isBranchPreview` (from `src/server/content/utils.ts`) is also used inside `setPreviewAndRedirect` to decide the scope of the preview cookie: for branch previews the cookie covers the entire site (`path` omitted), whereas for document previews it is scoped to the specific page path.

---

## Data Flow

```
Next.js getStaticProps
         │
         └─ contentService.*()   [isContentPlatformSource() === true]
                │
                ├─ getSingleContent  →  getFromCP()
                │                          └─ getEntry()
                │                               ├─ getEntryData()
                │                               │    ├─ clientCP.getEntry / getEntryBySlug
                │                               │    └─ clientCP.previewEntryById / previewEntryBySlug
                │                               └─ fetchFirstEntryFromList()
                │                                    └─ clientCP.listEntries → getEntryData()
                │
                ├─ getMultipleContent →  clientCP.listEntries
                │                            └─ fillEntriesWithData → getEntryData (per entry)
                │
                ├─ getPlpContent  →  getMultipleContent → findBestPLPTemplate
                │
                └─ getPdpContent  →  getMultipleContent → findBestPDPTemplate

         Resolved content (sections + settings)
                │
                └─ Passed as props to the page component
                         │
                         └─ <RenderSections sections={...} components={COMPONENTS} />
```
