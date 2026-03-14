# Legacy Headless CMS — Page Fetching Architecture

This document describes how `packages/core` fetches page content from the **VTEX Headless CMS** (the legacy CMS system, backed by the `@vtex/client-cms` package).

The system is active when `contentSource.type` is set to `'CMS'` (the default) in `discovery.config.js`.

---

## Table of Contents

- [Storage Backend](#storage-backend)
- [Configuration](#configuration)
- [CMS Client](#cms-client)
- [Core Fetch Functions](#core-fetch-functions)
- [Page-Type Fetchers](#page-type-fetchers)
  - [Home Page](#home-page)
  - [Landing Pages](#landing-pages)
  - [Product Listing Pages (PLP)](#product-listing-pages-plp)
  - [Product Detail Pages (PDP)](#product-detail-pages-pdp)
  - [Global Sections](#global-sections)
- [Preview Mode](#preview-mode)
- [CMS Content Definitions](#cms-content-definitions)
- [Data Flow](#data-flow)

---

## Storage Backend

The VTEX Headless CMS stores content in **VTEX MasterData V2**, but that storage is fully managed by the **`vtex.admin-cms` VTEX IO app** (a.k.a. "Headless CMS"). The FastStore codebase never reads from or writes to MasterData directly — all content access goes through the HTTP API that the IO app exposes.

### API endpoint

All CMS content reads go through the single base URL constructed by `ClientCMS`:

```
https://{workspace}--{tenant}.myvtex.com/_v/cms/api/{builder}/
```

| URL segment | Source | Example |
|---|---|---|
| `{workspace}` | `config.api.workspace` | `master` |
| `{tenant}` | `config.api.storeId` | `storeframework` |
| `{builder}` | `contentSource.project` (default `faststore`) | `faststore` |

The `/_v/` prefix identifies this as a **VTEX IO app route** — the request is handled server-side by the `vtex.admin-cms` app, not by the VTEX Core Commerce APIs.

### How `vtex.admin-cms` stores data internally

The `vtex.admin-cms` VTEX IO app stores CMS content in **VTEX MasterData V2** under its own managed entities and schemas. These are private to the app; FastStore interacts with them only indirectly through the HTTP API above. The key concepts are:

| Concept | What it is | MasterData mapping |
|---|---|---|
| **Document** | A single CMS page instance (e.g. the Home page, or a specific PLP template) | One MasterData V2 document, identified by `documentId` |
| **Version** | A saved snapshot/draft of a document | A sub-resource of the document, identified by `versionId` |
| **Release** | A named group of scheduled document versions | A separate MasterData V2 document linking multiple versions, identified by `releaseId` |
| **Content type** | The schema that defines what fields/sections a document can have | A MasterData V2 schema registered by `vtex.admin-cms` |

The `content-types.json` and `sections.json` files in `cms/faststore/` declare **which content types and sections exist**; they are synced to the `vtex.admin-cms` app via the `faststore cms-sync` command. That command (in `packages/cli/src/commands/cms-sync.ts`) merges core and user definitions and then delegates to `vtex cms sync` (the VTEX IO CLI), which registers them as schemas inside the app's MasterData entity.

### MasterData data entity for page content

The MasterData V2 **data entity** used by `vtex.admin-cms` to persist page content is internal to the app and is not directly queryable from outside. The entity identifier is managed by the app itself (it does not follow the standard user-accessible `CL`, `AD`, etc. naming). Accessing or modifying CMS pages must be done through the `/_v/cms/api/` HTTP interface rather than the MasterData REST API (`/api/dataentities/`).

### Schema sync flow

```
cms/faststore/content-types.json    ──┐
cms/faststore/sections.json         ──┤  `faststore cms-sync` (FastStore CLI wrapper)
user/cms/faststore/content-types.json ┤  packages/cli/src/commands/cms-sync.ts
user/cms/faststore/sections.json    ──┘
        │
        ▼  merged and written to .faststore/cms/faststore/
        │
        ▼  `vtex cms sync {builder}` (VTEX IO CLI — invoked internally by faststore cms-sync)
        │
        ▼  HTTP POST → vtex.admin-cms VTEX IO app
                │
                └─ Registers content-type schemas in MasterData V2
```

### MasterData entities used by the broader FastStore API (not CMS)

For completeness, the `packages/api` layer uses the following MasterData entities directly — these are **not** related to CMS content:

| Entity | Endpoint | Purpose |
|---|---|---|
| `NL` | `POST /api/dataentities/NL/documents/` | Newsletter opt-in subscription |
| `CL` | `GET /api/dataentities/CL/documents/{id}` | Customer contract data |
| `shopper` | `GET /api/dataentities/shopper/search` | Shopper profile lookup |

---

## Configuration

Relevant fields in `discovery.config.js` (defaults in `discovery.config.default.js`):

```js
// Activates the legacy Headless CMS
contentSource: {
  type: 'CMS',          // keeps the Headless CMS path active in ContentService
  project: 'faststore', // CMS builder project name (optional, defaults to 'faststore')
},

// Optional: pre-built CMS data injected at build time via the CMS_DATA env variable.
// When present it is used to resolve documentId/versionId lookups before hitting the API.
cms: {
  data: process.env.CMS_DATA,
},

// Workspace and store account used to instantiate ClientCMS
api: {
  storeId: 'storeframework', // VTEX account name (tenant)
  workspace: 'master',
},

// Preview redirect targets, one per content type
previewRedirects: {
  home: '/',
  plp: '/office',
  search: '/s?q=headphone',
  pdp: '/apple-magic-mouse/p',
},
```

---

## CMS Client

**File:** `src/server/cms/index.ts`

A singleton `ClientCMS` instance (from `@vtex/client-cms`) is created once and reused across all fetch calls:

```ts
export const clientCMS = new ClientCMS({
  workspace: config.api.workspace,
  tenant: config.api.storeId,
  builder: (config.contentSource as Record<string, string>)?.project ?? 'faststore',
  host: sanitizeHost(config.storeUrl),
})
```

The client exposes two underlying HTTP methods that the fetch helpers wrap:

| `ClientCMS` method | When used |
|---|---|
| `getCMSPage(locator)` | Fetch a single document by its `documentId` / `versionId` / `releaseId` |
| `getCMSPagesByContentType(contentType, filters)` | Fetch all documents of a content type, with optional filters |

---

## Core Fetch Functions

**File:** `src/server/cms/index.ts`

### `getCMSPage(options, cmsClient?)`

The lowest-level fetch helper. Accepts either a **Locator** (direct document reference) or a **content-type query**:

```ts
export type Options =
  | Locator                                           // { contentType, documentId?, versionId?, releaseId? }
  | { contentType: string; filters?: Partial<ContentTypeOptions> }
```

- **Locator path:** calls `clientCMS.getCMSPage(locator)` and wraps the single result in `{ data: [page] }`.
- **Content-type path:** calls `clientCMS.getCMSPagesByContentType` with `page = 1, perPage = 10`, then fires parallel requests for any remaining pages so that _all_ documents for that content type are returned in one `{ data: pages[] }` response.

### `getPage<T>(options)`

A convenience wrapper around `getCMSPage` that asserts exactly one result is returned, throwing `MultipleContentError` if more than one document matches.

---

## Page-Type Fetchers

### Home Page

**Files:** `src/pages/index.tsx`  
**Entry point:** `getStaticProps`

```
getStaticProps
  └── contentService.getSingleContent({ contentType: 'home', previewData, ... })
        └── (legacy CMS path) getPage({ contentType: 'home', ... })
              └── getCMSPage → clientCMS.getCMSPage / getCMSPagesByContentType
```

If `config.cms.data` (the `CMS_DATA` env variable) is set, the `documentId` and `versionId` are taken from the pre-built data instead of triggering a content-type listing call.

---

### Landing Pages

**Files:** `src/components/templates/LandingPage/LandingPage.tsx`, `src/pages/[...slug].tsx`  
**Entry point:** `getLandingPageBySlug(slug, previewData)`

```
getStaticProps (pages/[...slug].tsx)
  └── getLandingPageBySlug(slug, previewData)
        └── contentService.getSingleContent({
              contentType: 'landingPage',
              filters: { 'settings.seo.slug': `/${slug}` },   // slug-based lookup
              ...
            })
              └── (legacy CMS path) getPage({ contentType: 'landingPage', filters, ... })
                    └── getCMSPage → clientCMS.getCMSPagesByContentType (filtered)
```

When `config.cms.data` is set, the pre-built data is searched by `settings.seo.slug` to extract the `documentId`/`versionId` before calling `getPage`.

`getLandingPageBySlug` returns `null` (rather than throwing) when the content is not found, so the `[...slug].tsx` route can fall through to the PLP path.

---

### Product Listing Pages (PLP)

**Files:** `src/server/cms/plp.ts`, `src/pages/[...slug].tsx`  
**Entry point:** `contentService.getPlpContent(params, rewrites)`

```
getStaticProps (pages/[...slug].tsx)
  └── contentService.getPlpContent({ previewData, slug }, rewrites)
        └── (legacy CMS path) getPLP(slug, previewData, rewrites)
              ├── [with CMS_DATA] getPLPFromCmsEnvData
              │     └── getPage({ contentType: 'plp', documentId, versionId })
              │           └── getCMSPage → clientCMS.getCMSPage
              └── [without CMS_DATA] getPLPFromCms
                    └── getCMSPage({ contentType: 'plp' })
                          └── clientCMS.getCMSPagesByContentType
```

After all PLP documents are fetched, `findBestPLPTemplate(pages, slug, rewrites)` (in `src/utils/multipleTemplates.ts`) selects the most specific template for the requested slug.

#### Key types

```ts
// src/server/cms/plp.ts
export type PLPContentType = ContentData & {
  settings: {
    template?: { value?: string }
    productGallery: {
      itemsPerPage: number
      sortBySelection: string
    }
  }
}
```

---

### Product Detail Pages (PDP)

**Files:** `src/server/cms/pdp.ts`, `src/pages/[slug]/p.tsx`  
**Entry point:** `contentService.getPdpContent(product, params)`

```
getStaticProps (pages/[slug]/p.tsx)
  └── contentService.getPdpContent(data.product, { previewData, slug })
        └── (legacy CMS path) getPDP(product, previewData)
              ├── [with CMS_DATA] getPDPFromCmsEnvData
              │     └── getPage({ contentType: 'pdp', documentId, versionId })
              │           └── getCMSPage → clientCMS.getCMSPage
              └── [without CMS_DATA] getPDPFromCms
                    └── getCMSPage({ contentType: 'pdp' })
                          └── clientCMS.getCMSPagesByContentType
```

After fetching all PDP documents, `findBestPDPTemplate(pages, product)` (in `src/utils/multipleTemplates.ts`) selects the most specific template for the requested product.

#### Key types

```ts
// src/server/cms/pdp.ts
export type PDPContentType = ContentData & {
  settings: {
    template?: { value?: string }
    seo?: { id?: string; mainEntityOfPage?: string }
  }
}
```

---

### Global Sections

**Files:** `src/components/cms/GlobalSections.tsx`, `src/server/cms/global.ts`  
**Entry points:** `getGlobalSectionsData(previewData)`, `injectGlobalSections(...)`

Every page calls `getGlobalSectionsData` in its `getStaticProps` to fetch three parallel content types that together form the shared layout (Navbar, Footer, and any cross-cutting sections):

| Content type constant | Value |
|---|---|
| `GLOBAL_SECTIONS_CONTENT_TYPE` | `'globalSections'` |
| `GLOBAL_SECTIONS_HEADER_CONTENT_TYPE` | `'globalHeaderSections'` |
| `GLOBAL_SECTIONS_FOOTER_CONTENT_TYPE` | `'globalFooterSections'` |

```
getStaticProps (any page)
  └── getGlobalSectionsData(previewData)           // returns three promises
        ├── getGlobalSectionsByType(previewData, 'globalSections')
        │     └── contentService.getSingleContent({ contentType: 'globalSections', ... })
        │           └── (legacy CMS path) getPage(...)
        ├── getGlobalSectionsByType(previewData, 'globalHeaderSections')
        │     └── contentService.getSingleContent(...)
        └── getGlobalSectionsByType(previewData, 'globalFooterSections')
              └── contentService.getSingleContent(...)
```

Once all three resolve, `injectGlobalSections({ globalSections, globalSectionsHeader, globalSectionsFooter })` (in `src/server/cms/global.ts`) merges them into a single `GlobalSectionsData` object. It locates the special `Children` section inside `globalSections` and splices header sections before it and footer sections after it.

---

## Preview Mode

**File:** `src/pages/api/preview.ts`

The preview API route (`GET /api/preview`) enables Next.js Preview Mode so that CMS editors can see unpublished content. It accepts the following query parameters:

| Parameter | Required | Description |
|---|---|---|
| `contentType` | Yes | The content type to preview (e.g. `home`, `plp`, `pdp`, `landingPage`) |
| `documentId` | Yes | The CMS document ID |
| `versionId` | One of these | A specific version of the document |
| `releaseId` | One of these | A scheduled release that contains the document |
| `slug` | No | Page slug, used for `landingPage` / `plp` / `pdp` redirects |
| `action=clear` | — | Clears preview mode and redirects |

**Flow:**

1. Extract and validate the `contentType`, `documentId`, and one of `versionId` / `releaseId` from the request.
2. Call `contentService.getSingleContent({ contentType, previewData: locator, ... })` to confirm the document exists in the CMS. Returns `400` / `404` if validation fails.
3. Call `res.setPreviewData(locator)` to write the preview cookie, then `res.redirect(...)` to the page that displays the content.  The redirect target is looked up from `previewRedirects` in `discovery.config.js`.

When preview mode is active, `previewData` is injected into every `getStaticProps` call. The fetch helpers forward `documentId` / `versionId` / `releaseId` to `getCMSPage`, which routes the call through `clientCMS.getCMSPage(locator)` instead of the content-type listing path.

---

## CMS Content Definitions

The CMS schema is declared in `cms/faststore/`:

| File | Purpose |
|---|---|
| `content-types.json` | Declares all page types available in the CMS admin |
| `sections.json` | Declares the reusable section components that can be placed inside a page |
| `cms/faststore/pages/` | Per-content-type JSON schemas (allowed sections, settings fields) |
| `cms/faststore/components/` | Per-section JSON schemas (editable fields) |

### Registered content types

| ID | Name |
|---|---|
| `home` | Home |
| `landingPage` | Landing Page |
| `plp` | Product List Page |
| `pdp` | Product Page |
| `search` | Search Page |
| `globalSections` | Global Sections |
| `globalHeaderSections` | Global Header Sections |
| `globalFooterSections` | Global Footer Sections |
| `login` | Login |
| `404` | Error 404 |
| `500` | Error 500 |

### Registered section components

Alert, BannerNewsletter, BannerText, Breadcrumb, CartSidebar, Children, CrossSellingShelf, EmptyState, Footer, Hero, Incentives, Navbar, Newsletter, ProductDetails, ProductGallery, ProductShelf, ProductTiles, RegionBar, RegionModal, RegionPopover, Search.

These definitions are synced to the VTEX CMS admin via the CLI command `vtex cms sync` (wrapped by `packages/cli/src/commands/cms-sync.ts`).

---

## Data Flow

```
Next.js getStaticProps / getServerSideProps
         │
         ├─ getGlobalSectionsData(previewData)
         │    ├─ contentService.getSingleContent({ contentType: 'globalSections' })
         │    ├─ contentService.getSingleContent({ contentType: 'globalHeaderSections' })
         │    └─ contentService.getSingleContent({ contentType: 'globalFooterSections' })
         │         └── (all three) getPage() → getCMSPage()
         │                                         ├─ clientCMS.getCMSPage(locator)     [Locator path]
         │                                         └─ clientCMS.getCMSPagesByContentType [ContentType path]
         │
         └─ Page-specific fetch
              ├─ home     → contentService.getSingleContent({ contentType: 'home' })
              ├─ landingPage → getLandingPageBySlug(slug)
              │                 └─ contentService.getSingleContent({ contentType: 'landingPage', filters })
              ├─ plp      → contentService.getPlpContent()
              │                 └─ getPLP() → getPLPFromCms() / getPLPFromCmsEnvData()
              └─ pdp      → contentService.getPdpContent()
                                └─ getPDP() → getPDPFromCms() / getPDPFromCmsEnvData()

         After all promises resolve:
         injectGlobalSections({ globalSections, globalSectionsHeader, globalSectionsFooter })
                │
                └─ Merged GlobalSectionsData + page sections
                         │
                         └─ Passed as props to the page component
                                  │
                                  └─ <RenderSections sections={...} components={COMPONENTS} />
                                           │
                                           └─ Each section.name is mapped to a React component
```
