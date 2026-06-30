# Entity / List UI Patterns — Caesar → Daiton

The distilled, self-contained reference for the config-driven entity/list UI
system. Read this before building or changing any Daiton frontend surface. Start
here, then cross-check against the live Daiton code (the components are already
ported — the gap is wiring).

## The config-driven spine (the through-line)

> **One `EntityConfig<T>` per entity type drives every surface** — its table
> columns, its detail sections, its card, its graph node, its chat citation —
> from a single declaration. Add a new entity type = write one config, not touch
> six surfaces.

Caesar implements this as a **three-tier** system. The tiers, top (aspirational)
to bottom (the real load-bearing convention):

### Tier 1 — the master `EntityConfig<T>` (aspirational even in Caesar)

```ts
// ch-front-end/src/types/entity-config.ts
type EntityConfig<T> = {
  name: string;          // "Person"
  namePlural: string;    // "People"
  icon: ReactNode;       // leading type glyph
  // Factory functions return the per-surface config:
  table?:   (props) => Omit<EntityTableConfig<T>,  'data' | 'totalCount' | 'selectedItem' | 'urlState'>;
  details?: (props) => Omit<EntityDetailsConfig<T>, 'entity'>;
  // Daiton extends the master with projections for non-table surfaces:
  // graphNode, graphEdge, chatCitation, chatMention, linkerPreview
};
```

**Honest nuance:** there is **no `useEntityConfig` hook and no
`EntityConfig<Contact>` instance** in the Caesar tree. The master factory type is
the formalization, not the shipped pattern. Don't block on a perfect registry.

### Tier 2 — the per-surface config types (what Caesar actually ships)

```ts
// ch-front-end/src/types/entity-config.ts
type EntityTableConfig<T> = {
  data: T[]; columns: ColumnDef<T>[]; totalCount: number;
  selectedItem?: T; onSelectItem?: (t: T) => void; getItemId: (t: T) => string;
  searchConfig?: SearchConfig; filters?: FilterConfig<T>[]; urlState: TableUrlState;
  cardConfig?: EntityCardConfig<T>;
  loading?: boolean; fetching?: boolean; compact?: boolean;
};
type EntityDetailsConfig<T> = {
  entity?: T; emptyState?: ReactNode;
  sections: ({ type: 'info';     label: string; value: ...; format?: 'date'|'badge'|'text'|'custom' }
           | { type: 'metadata' }
           | { type: 'actions'; buttons: { label: string; onClick; condition?: (t:T)=>boolean }[] }
           | { type: 'custom';   render: (t: T) => ReactNode })[];
};
type EntityCardConfig<T> = {
  getName: (t: T) => string; getTimestamp?: (t: T) => string;
  getCategories?: (t: T) => string[]; getSubtitle?: (t: T) => string;
  getStatusDisplay?: (t: T) => { label: string; variant: BadgeVariant };
};
```

### Tier 3 — the generic components (the real, reused convention)

| Component | Caesar source | Consumes |
|---|---|---|
| `EntityTable<T>` | `components/data-table/entity-table.tsx` (~990 lines) | `EntityTableConfig<T>` |
| `EntityDetails<T>` | `components/layout/entity-details.tsx` (244) | `EntityDetailsConfig<T>` |
| `EntityCardBase` | `components/emr-core/cards/base/entity-card-base.tsx` | `EntityCardConfig<T>` (via props) |
| `MasterDetailLayout` | `components/layout/master-detail-layout.tsx` (189) | `listPanel` + `detailPanel` children |

## The `EntityTable` — the generic server-side-paginated table

`ch-front-end/src/components/data-table/entity-table.tsx`. The single component
behind every Caesar list. It bundles:

- **Toolbar** (`data-table/toolbar.tsx`) — global search + faceted filters + sort + view-options + a configurable primary action (Compose).
- **Pagination** (`data-table/pagination.tsx`) — page-based, offset + total. **Always server-side** when `totalCount` + `urlState` are passed; falls back to client-side paging only for small fully-loaded sets.
- **Column header** (`data-table/column-header.tsx`) — sortable header that drives the URL-synced sort state.
- **Faceted filter** (`data-table/faceted-filter.tsx`) — declarative `FilterConfig<T>[]` → popover checkboxes over unique values.
- **View options** (`data-table/view-options.tsx`) — column visibility / density toggles.
- **Card mode** — when `compact` is set and a `cardConfig` is provided, rows render as `EntityCardBase`s instead of `<tr>`s. This is how the same table powers both the dense list and the card gallery.

**Daiton status:** `EntityTable` is **ported and rendered** (inbox, llm-roles,
catalog-wrapped). The toolbar/filter/sort/view-options sub-components came with
it. Daiton wraps it in `features/catalog/virtual-data-table.tsx` for the
virtualized full-catalog view — a sanctioned exception for the "see everything"
view only; new lists should use the plain server-side path, not virtualization.

## The `EntityDetails` — the section-model detail body

`ch-front-end/src/components/layout/entity-details.tsx` (244 lines). Switches on
`section.type` and renders `Card`-wrapped sections in a `ScrollArea`:

- `info` — icon + label + value, with a `format` hint (`date` | `badge` | `text` | `custom`).
- `metadata` — key:value rows.
- `actions` — buttons, each with an optional `condition` predicate for show/hide.
- `custom` — a render prop `(t: T) => ReactNode` for anything bespoke.

**Daiton status:** `EntityDetails` is **ported but UNUSED**. Every Daiton detail
surface hand-builds its markup and re-defines a local `Field` component ad hoc
(the duplication). The highest-leverage detail move: retarget the detail panel to
render `EntityDetails` from `useEntityConfig(type).details(...)` instead of
bespoke markup — this also kills the per-file `Field` duplication.

## The `EntityCardBase` — the generic record card

`ch-front-end/src/components/emr-core/cards/base/entity-card-base.tsx`. The prop
contract (the anatomy that matters):

```text
icon            ReactNode   — leading type glyph
title           string      — primary line
subtitle?       string      — secondary line (muted)
status?         string + statusVariant  — a Badge (default/secondary/destructive/outline)
reason?         string      — a separated explanatory line (line-clamped)
metadata?       {label,value}[]  — key/value rows
children?       ReactNode   — domain-specific body
aiGenerated? + confidence?  — a subtle "AI (NN%)" footer  ← trust signal, first-class
widgets?        ('add'|'edit'|'remove')[] + onWidgetAction  — inline row actions
onClick?        — make the whole card a click target
size            'compact' | 'standard' | 'detailed'  ← progressive disclosure
```

**Variant model (`size`)** drives padding (`p-3`/`p-4`/`p-6`), text scale, and
**progressive disclosure**: `compact` = icon + title + status only; `standard`
adds reason + metadata + AI footer; `detailed` is the same at larger scale.
Separators are inserted only between the sections that actually render.

**How cards compose:**
- **Domain card → base.** Each domain card (e.g. `clinical/medication-card.tsx`) is a thin adapter: maps its typed entity into `EntityCardBase` props and may add `children`.
- **Lists.** `EntityListCard` / `EntityListCardAuto` (`cards/base/entity-list-card.tsx`) wrap a titled, empty-state-aware, loading-skeleton list of cards.
- **Renderer / gallery.** `features/scribe/components/card-renderer.tsx` is a big `switch(cardName)` mapping the encounter-engine's entity bag to the right card(s) inside `EntityListCard`s — cards are the **primary read surface** of the live encounter.

**Daiton status:** **No card model exists.** This is the biggest UI gap. The
`aiGenerated`+`confidence` footer is a perfect fit for Daiton's extraction/ER
confidence — the trust signal is first-class in the card.

## The `MasterDetailLayout` + navigation

`ch-front-end/src/components/layout/`:

- **`master-detail-layout.tsx`** (189) — two-panel responsive: desktop = side-by-side; mobile = detail slides full-screen over list. Collapsible list panel. Compact card mode.
- **`flexible-detail-layout.tsx`** (381) + `.example.tsx` — richer multi-region detail composition.
- **`split-detail-view.tsx`** (198) + **`detail-view-layout.tsx`** (235) + `detail-view-controls.tsx` — the standard detail chrome (header, close, list-toggle) that bespoke `*DetailView`s render into.

**Navigation:**
- `context/navigation-provider.tsx` — the breadcrumb trail/stack (a context provider holding the navigation history).
- `components/navigation/breadcrumb-navigation.tsx` — renders the trail.
- `components/navigation/entity-navigation.tsx` (485 lines) — the cross-entity nav graph (Caesar-only; Daiton did not port).
- `components/navigation/outline-navigator.tsx`.

**Daiton status:** `MasterDetailLayout`, `EntityPageLayout`, navigation-provider,
and breadcrumb-navigation are **ported and used**. Daiton's `MasterDetailLayout`
fork is the **weaker subset** — it lost the mobile overlay + compact-card mode;
re-sync to Caesar's if you need responsive behavior.

## The worked example — `features/contacts/index.tsx` (copy this shape)

This is the template that wires everything together. Mirror it for any new
entity surface:

```tsx
// ch-front-end/src/features/contacts/index.tsx  (the canonical wiring)
import { EntityPageLayout } from '@/components/layout/entity-page-layout';
import { MasterDetailLayout } from '@/components/layout/master-detail-layout';
import { EntityTable } from '@/components/data-table/entity-table';
import { columns } from './components/contacts-columns';        // ColumnDef<T>[]
import { CONTACT_FILTERS, SEARCH_CONFIG } from './components/contacts-columns';
import { ContactDetailView } from './components/contact-detail-view';
import { useTableUrlState } from '@/hooks/use-table-url-state';

export function ContactsPage() {
  const { mode, selectedId, setSelectedId, urlState } = useTableUrlState(/*...*/);
  const { contacts, total } = useContactsQuery(urlState);       // TanStack Query, server-side page
  const selectedContact = contacts.find(c => c.id === selectedId);

  return (
    <EntityPageLayout headerConfig={{ icon: Users, title: 'Contacts', /* actions */ }}>
      <MasterDetailLayout
        mode={mode}
        listPanel={
          <EntityTable
            data={contacts} columns={columns} totalCount={total}
            selectedItem={selectedContact} onSelectItem={setSelectedId}
            getItemId={(c) => c.id}
            searchConfig={SEARCH_CONFIG} filters={CONTACT_FILTERS}
            urlState={urlState} compact={mode === 'master-detail'}
            cardConfig={{ getName, getTimestamp, getCategories, getSubtitle }}
          />
        }
        detailPanel={selectedContact && <ContactDetailView contact={selectedContact} /*…*/ />}
      />
    </EntityPageLayout>
  );
}
```

- `columns` come from `contacts-columns.tsx` — a `ColumnDef<T>[]` with a `select` checkbox, sortable `DataTableColumnHeader`s, and an actions column.
- `filters` are a declarative `FilterConfig<T>[]`.
- `urlState` is the shared `useTableUrlState` hook (page/pageSize/filter/sort all URL-synced).

**This is the pattern to copy** — and Daiton already has every one of these
components ported.

## The Daiton adaptation table

| Caesar concept | Daiton today | The move |
|---|---|---|
| Config types (`EntityConfig`/Table/Details/Card) | **Ported verbatim** | none — types exist |
| Config-driven tables (`EntityTable` + per-feature `ColumnDef[]` + filters + urlState) | **Ported & rendered** (inbox, llm-roles, catalog-wrapped) | mostly closed |
| Config-driven details (section model) | `EntityDetails` **ported but unused**; every detail hand-built | **open** — adopt the section model |
| `EntityConfig` registry/factory | Hook **ported but DORMANT** (0 registrations, 0 consumers) | **open** — author 8 configs, call `registerEntityConfig` |
| `EntityCardBase` + cards + `EntityListCard` | **NONE** | **open** — lift `EntityCardBase` verbatim, build ~4 domain cards |
| `MasterDetailLayout` | **ported (weaker fork)** | re-sync mobile overlay + compact-card mode if needed |
| Routed deep-link detail | **yes** (`catalog/$entityId`) | closed (Daiton ahead here) |
| Breadcrumb / nav-provider | provider + breadcrumb **ported & used**; no `entity-navigation` | mostly closed |
| Shared detail-field primitive | local `Field` re-defined ad hoc | **open** — retire ad-hoc `Field` for `EntityDetails` sections |
| AI-confidence affordance | ad hoc per surface | **open** — bake into `EntityCardBase` |
| Popover for record read | not used (master-detail) | **half-retired** — entity→route done; fact→`Sheet` remains |

## The detail pattern — retire the popover, make it ONE thing

Today in Daiton: entity rows route to `catalog/$entityId` (good); **fact rows
still open a `Sheet` popover** (the half-measure). Unify:

- Route the fact branch too (`catalog/$factId` or a routed detail) rendering `EntityDetails` from the fact config.
- Reserve `Sheet` for the graph canvas only.
- Re-sync `MasterDetailLayout` to Caesar's (Daiton's fork lost the mobile overlay + compact-card mode).

## Don't forget

- **Render-verify** every one of these — a beautiful config that doesn't render is not done.
- Type everything **over the generated wire types** once OpenAPI→TS codegen lands, so columns/sections can't drift from the API. (The hand-mirrored type contract is **already drifting** in Daiton HEAD: FE `type`/`object_literal`/`valid_from` vs BE `type_label`/`object_value`/`valid_at`. The durable fix: restore `response_model` on routes → OpenAPI → `openapi-typescript` → `api.gen.ts` → author EntityConfig over generated types.)
