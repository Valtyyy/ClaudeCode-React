# Tables

Every list table is `@tanstack/react-table` **v8** rendered through the shared `DataTable` in
`components/ui-kit/`. The table is headless and **entirely server-driven**: the backend paginates,
sorts and filters; the URL search params are the single source of truth; the table instance only
reflects them.

## Contents

- The pieces and who owns what
- Everything is manual — never a client row model
- A column owns its filter
- Filters render in the column header, nowhere else
- Below `md` the drawer replaces the headers
- The memoization contract
- Sorting only where the API supports it
- The in-memory exception
- Where this departs from the `tanstack-table` skill
- Banned patterns

---

## The pieces and who owns what

| Piece | Lives in | Owns |
| ---- | ---- | ---- |
| `useTableSearchState` | `hooks/` | URL search ⇄ table state; the 1-based `page` ⇄ 0-based `pageIndex` conversion; `setFilters` (always resets to page 1); `getFilterValue` |
| `useDataTable` | `components/ui-kit/` | the `useReactTable` instance and its manual flags |
| `DataTable` | `components/ui-kit/` | header, rows, mobile cards, loading, empty state, header-hosted filters |
| `ColumnFilter` | `components/ui-kit/column-filter.tsx` | one column's filter popover, and the shared `FilterControl` for each descriptor variant |
| `DataTableFilterDrawer` | `components/ui-kit/` | the sub-`md` filter drawer, built from the same descriptors |
| `DataTablePagination` | `components/ui-kit/` | page size, page navigation, the `from–to of total` line |
| `columns.tsx` | `features/<resource>/` | the `ColumnDef[]` factory, each column's `meta.filter`, the mobile card |
| `<resource>-filter.tsx` | `features/<resource>/` | the concrete control a descriptor renders |
| `queryOptions()` | `api/<resource>.ts` | the request, the query key, `placeholderData: keepPreviousData` |

A route wires these together and nothing more. It never defines a filter's JSX, a `useMutation`, or
a `useReactTable` call of its own (see `.claude/rules/features.md`).

## Everything is manual — never a client row model

```ts
manualPagination: true
manualSorting: true
manualFiltering: true
getCoreRowModel: getCoreRowModel()   // and no other row model
```

`getPaginationRowModel` would paginate the already-paginated page. `getFilteredRowModel` and
`getSortedRowModel` would filter or sort one page of N and silently lie about the rest. Row counts
come from the API envelope (`total`), never from `rows.length`.

The same rule reaches the search params: **never invent a filter the API cannot serve.** A list
endpoint that accepts no text search gets no search box, and that absence is deliberate — document
it in the column file rather than leaving the next reader to think it was forgotten.

## A column owns its filter

A filter is declared as data on the column it belongs to, through `meta.filter`
(`ColumnFilterDescriptor` in `types/data-table.ts`). One column, at most one descriptor.

```tsx
columnHelper.accessor('status', {
  header: 'Status',
  enableSorting: false,
  cell: ({ row }) => <StatusBadge status={row.original.status} />,
  meta: {
    filter: {
      variant: 'control',
      param: 'status',
      render: ({ value, onChange }) => {
        const parsed = statusSchema.safeParse(value)
        return <StatusFilter value={parsed.success ? parsed.data : undefined} onChange={onChange} />
      },
    },
  },
})
```

Three variants: `text` (a debounced input), `control` (any component, via a `render` closure), and
`date-range` (two params at once). `param` is the URL search param the filter drives, and it is the
only mapping — there is no second table of param-to-column anywhere, because a second table would
drift from this one.

**The descriptor's value is `unknown` by design.** A filter's value type belongs to its search
param, not to the column's accessor: the two are frequently unrelated, as when a column accesses a
nested id for display while its filter drives a name-searching combobox. Narrow the `unknown` with a
type guard inside `render`, where the concrete type is in scope, exactly as `CLAUDE.md` requires of
any external data.

Cross-filter dependencies (one filter's option list scoped by another filter's current value) use
the `getFilterValue` passed into `render` to read a sibling param, and the route wraps `setFilters`
to enforce the invariant — clearing the dependent params whenever the parent one changes. Keep that
wrapper in the route, greppable, never hidden in `meta`.

## Filters render in the column header, nowhere else

At `md` and above, a column that declares a filter renders a ghost icon button in its `<th>`, next
to the label and next to the sort control if it has one. The button opens a `Popover` holding the
control, and carries an active mark while the filter is set. There is **no filter bar above the
table**, no page-level search box, and no route-assembled toolbar. A route renders a `PageHeader`,
then the table.

"Clear filters" belongs to the table too: a reset icon in the header cell of the last column, shown
only while some filter is active, patching every declared param to `undefined`.

The corollary is a hard requirement on `DataTable`: **the header renders in every state.** Loading
rows and the empty state go inside the table body, never in place of the header. Filtering down to
zero rows must never remove the control that got you there.

## Below `md` the drawer replaces the headers

There are no headers in the card list, so `DataTableFilterDrawer` renders a single `Filters` button
carrying the active-filter count, opening a `ResponsiveModal` that lists every column's filter. It
is built by walking the same descriptors and renders the same `FilterControl`, so the two surfaces
cannot diverge. It is `md:hidden`; the header popovers are the only filter UI above that breakpoint.

## The memoization contract

`useReactTable` rebuilds the instance whenever `data` or `columns` change identity, so an inline
array or object literal is an infinite render loop, not a small waste.

- Wrap every column factory call in `useMemo` with an explicit dependency array.
- Wrap every `rowActions` object and every callback handed to a column factory in `useMemo` /
  `useCallback`.
- **Never pass a modal-state object (e.g. a `useCrudModal` result) into a column factory.** Its
  callbacks are individually stable but the returned object is new every render. Pass the discrete
  callbacks.
- Pass `data?.items ?? EMPTY_ROWS`, where `EMPTY_ROWS` is the module-level constant exported by
  `use-data-table.ts` — never an inline `[]`.
- `onFiltersChange` must be stable; `setFilters` and a route's cross-filter wrapper already are.

## Sorting only where the API supports it

Every column sets `enableSorting: false` unless the backend accepts a sort param for it. The route
holds the `SortingState` derived from its sort search param and maps `onSortingChange` back to the
URL. Client-side sorting is never enabled, for the same reason client row models are not: it would
sort one page and mislead.

A sortable header is a `<Button variant="ghost">` inside the `<th>`, with `aria-sort` on the `<th>`.
Never a bare `onClick` on the `<th>` — `components/ui-kit/**` is **not** exempt from Biome's a11y
rules (only the CLI-owned `components/ui/**` is), and the fix is the button, never a `biome-ignore`.

## The in-memory exception

A table whose rows are already fully in memory — a wizard step assigning items, an editor working on
a local collection — is the one case that inverts the configuration: `manualFiltering: false` with
`getFilteredRowModel()`, a `globalFilterFn`, no pagination, and local `useState` instead of URL
params. Such a table may keep its own local toolbar when its controls are not per-column (a search
spanning several fields, chips partitioning rows). Do not copy its configuration into a server-side
table, and do not copy a server-side table's configuration into it.

## Where this departs from the `tanstack-table` skill

The `tanstack-table` skill is upstream library documentation and everything above agrees with it —
memoized `data`/`columns`, `flexRender` everywhere, `getRowId`, `createColumnHelper`, `manualX` for
server-side, controlled state paired with its `onXChange`, module augmentation for `meta` (the
skill's own example augments `ColumnMeta` with a `filterVariant`, which is exactly what
`meta.filter` is here). Three deliberate deviations, so nobody "corrects" working code back toward
the docs:

1. **No `columnFilters` state, and no `onColumnFiltersChange`.** The skill's server-side example
   threads filter values through the table. Here they live in the URL and are read through
   `useTableSearchState`; `meta.filter.param` is the column-to-param mapping. `ColumnFiltersState`
   is one value per column id, which several filters are not: a `date-range` owns two params at
   once, and a multi-select filter's value is an array of ids unrelated to its column's accessor.
   Forcing them into `columnFilters` would mean reshaping accessors to match filters.
2. **No `autoResetPageIndex`.** The skill lists it for "filtering should reset pagination". That
   option belongs to the client-side pipeline; with `manualPagination` the reset is the URL's job
   and `setFilters` already does it, on every filter change, in one place.
3. **No `useEffect` refetch.** The skill's server-side snippet fetches from an effect. Data is
   fetched only through TanStack Query — a route loader plus `useQuery` on the same
   `queryOptions()` object — per `CLAUDE.md`. The search params change, the query key changes, the
   data follows.

## Banned patterns

These fail review:

```tsx
<DataTableToolbar … />                        // no filter bar above a table
<Input placeholder="Search…" />               // above a table — the search belongs to its column's header
getPaginationRowModel()                       // re-paginates an already-paginated page
getFilteredRowModel()                         // on a server-side table: filters one page of N
columns={createResourceColumns(actions)}      // unmemoized factory call — render loop
createResourceColumns({ modal })              // passing the whole modal object — can never memoize
<th onClick={…}>                              // a11y violation; use a ghost Button
{isLoading ? <TableSkeleton/> : <Table/>}     // drops the header, and with it every filter
```
