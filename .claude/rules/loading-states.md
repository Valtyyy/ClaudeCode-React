# Loading States

Skeletons are the only loading idiom in this codebase.

## Contents

- Skeletons replace every read-in-flight
- The shared vocabulary lives in `ui-kit/skeletons.tsx`
- Skeletons mirror the geometry of what they replace
- Pagination keeps the previous page — never skeletons
- Route loaders ship a `pendingComponent`
- Modals open immediately onto a skeleton body
- Mutations use button state, not skeletons
- Banned patterns

---

## Skeletons replace every read-in-flight

Any data being fetched renders a shadcn `<Skeleton>` placeholder shaped like the content it
is replacing. Never a spinner, never a `Loading…` string, never a blank region, never a
container collapsed to zero height.

| Situation                                    | What renders                                                  |
| -------------------------------------------- | ------------------------------------------------------------- |
| Route entered, loader resolving               | Route `pendingComponent`: real `PageHeader` + `TableSkeleton`  |
| List query loading (first load, filter change)| `DataTable isLoading` → `TableRowsSkeleton` **under the live header** / `CardListSkeleton` |
| Page change                                   | Previous page, dimmed — **no skeleton**                        |
| Detail query in a view/edit modal             | `DetailSkeleton` / `FormSkeleton` inside an already-open modal |
| Combobox / Select options loading             | Skeleton option rows in the popover                            |
| Session / profile query on first paint        | `AvatarSkeleton` + `TextSkeleton` in the header/sidebar        |
| Mutation in flight                            | Disabled button + inline spinner — **no skeleton**             |

## The shared vocabulary lives in `ui-kit/skeletons.tsx`

Compose from it. Do not write ad-hoc placeholder markup in a route or feature component.

```tsx
import { TableSkeleton, FormSkeleton, DetailSkeleton } from '@/components/ui-kit/skeletons'
```

Exports: `TableSkeleton`, `TableRowsSkeleton`, `CardListSkeleton`, `CardGridSkeleton`,
`FormSkeleton`, `DetailSkeleton`, `TextSkeleton`, `TriggerSkeleton`, `AvatarSkeleton`.

`TableSkeleton` is the whole responsive block — a table shape at `md+`, a card list below — and
belongs in a route `pendingComponent`, where no table exists yet. `TableRowsSkeleton` is its body
alone, the rows without the `<Table>` or `<TableHeader>` wrapper, and is what `DataTable` renders
while a list query is in flight: the real header stays on screen, because it carries the column
filters and losing it would strand the user (see `.claude/rules/tables.md`). `TableSkeleton`
composes `TableRowsSkeleton`, so the two geometries cannot drift.

If a new loading shape is genuinely needed, add it to `skeletons.tsx` so the next feature
reuses it — never inline a one-off.

## Skeletons mirror the geometry of what they replace

Same row count, same approximate widths, same spacing, so nothing shifts when data lands.
Vary widths across rows so it does not read as a uniform grey block. Loading UI is
mobile-first like everything else and respects the same table-vs-card breakpoint as its real
counterpart.

```tsx
// Good — final height reserved, widths vary
<TableSkeleton rows={size} cols={columns.length} />

// Bad — layout jumps when data arrives
{isLoading && <Skeleton className="h-4 w-full" />}
```

## Pagination keeps the previous page — never skeletons

`placeholderData: keepPreviousData` keeps the current page visible while the next one loads.
Dim it while `isPlaceholderData` is true. Paging must never flash skeletons.

```tsx
const { data, isPlaceholderData } = useQuery({
  ...resourceQueryOptions(search),
  placeholderData: keepPreviousData,
})

<div className={cn('transition-opacity', isPlaceholderData && 'opacity-60')}>
```

## Route loaders ship a `pendingComponent`

Every dashboard / list route defines one. It renders the real page header — title and description
are static, so they are not skeletons — above a skeleton body.

```tsx
export const Route = createFileRoute('/<resource>')({
  loader: ({ context, deps }) => context.queryClient.ensureQueryData(resourceQueryOptions(deps)),
  pendingComponent: () => (
    <>
      <PageHeader title="<Resource>" />
      <TableSkeleton rows={20} cols={6} />
    </>
  ),
  component: ResourcePage,
})
```

## Modals open immediately onto a skeleton body

Opening a view or edit modal never waits on its detail query. The `ResponsiveModal` opens at
once with its real title, and the body is a skeleton until data lands. Never delay the open,
never render an empty modal.

```tsx
<ResponsiveModal open={modal.mode === 'view'} onOpenChange={modal.close} title="Resource details">
  {isLoading ? <DetailSkeleton rows={5} /> : <ResourceDetail data={data} />}
</ResponsiveModal>
```

## Mutations use button state, not skeletons

Skeletons are for reads. Writes belong to the control that triggered them: the button goes
`disabled` with an inline spinner and the form locks. Do not skeleton-out a form or a row
because a mutation is pending.

```tsx
<Button type="submit" disabled={isPending}>
  {isPending && <Loader2 className="animate-spin" />}
  Save
</Button>
```

## Banned patterns

These fail review:

```tsx
{isLoading && <p>Loading...</p>}      // no loading text
{isLoading && <Spinner />}            // spinners never stand in for content
{isLoading && null}                   // no blank region
{isLoading ? <Skeleton /> : <Table/>} // single generic skeleton for a whole table
```
