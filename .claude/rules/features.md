# `src/features/`

`src/features/` holds the resource-specific UI for each domain entity or feature — the columns,
detail views, forms, modals, and mutation hooks that only make sense for one resource. It sits
between `components/` (generic, reusable, resource-agnostic) and `routes/` (page composition
and data loading).

## Contents

- One subfolder per resource
- What belongs in a feature folder
- What does not belong here
- Mutations live in the feature, not the route
- Routes compose, features implement
- Naming

---

## One subfolder per resource

```
src/features/
  <resource-a>/
  <resource-b>/
  users/
```

Each subfolder is named after the resource it belongs to (matching the route file and the
`api/*.ts` module for that resource). There is no shared `features/index.ts` barrel — routes
import directly from the specific file they need.

## What belongs in a feature folder

| File                          | Purpose                                                              |
| ------------------------------ | --------------------------------------------------------------------- |
| `columns.tsx`                  | `ColumnDef[]` factory + the mobile card for `DataTable`                |
| `*-detail.tsx`                 | Read-only detail view rendered inside a view modal                    |
| `*-form.tsx`                   | `TanStack Form` create/edit form, shared between create and edit modes|
| `*-modals.tsx`                 | Wires create/view/edit/delete modals to a single modal state (e.g. `useCrudModal`) |
| `use-*-mutations.ts` / `mutations.ts` | `useMutation` hooks for that resource's writes (see below)     |
| resource-specific hooks/filters | e.g. `use-<resource>-lookup.ts`, `<resource>-filter.tsx`              |

If a piece of UI or logic is only ever used by one resource's route, it goes in that resource's
feature folder — not in `components/`.

A table filter is split between exactly two of those files and no third place: the **descriptor**
goes on the column that owns it in `columns.tsx` (`meta.filter`, see `.claude/rules/tables.md`), and
the **control it renders** goes in `<resource>-filter.tsx` next to it. A route never renders filter
JSX — the shared `DataTable` reads the descriptor and paints it into the column's header. If two
resources need the same control, it stops being a feature file and moves to `components/ui-kit/`.

A record's technical `id` belongs to `*-detail.tsx` and nowhere else: first row of the detail view,
rendered `font-mono`. Never add it as a `ColumnDef` or to the mobile card — an ID is copy-paste
support data (to hit the API, file a ticket, query the DB), not something anyone scans a table for,
and it steals width from the columns that are. Don't confuse it with the `id:` key of a `ColumnDef`,
which is a column identifier for `DataTable` and unrelated.

## What does not belong here

- Generic, reusable UI (buttons, badges, skeletons, the `DataTable`/`ResponsiveModal`/`ConfirmDialog`
  primitives) — that's `components/ui/` (shadcn, CLI-owned) or `components/ui-kit/` (shared
  composites).
- `queryOptions()` factories and raw fetch functions — those live in `api/*.ts`. A feature
  imports and calls them; it does not define them.
- Shared types — `types/*.ts`, imported into the feature, not redefined there.
- Anything used by more than one resource. Promote it to `components/` or `hooks/` instead of
  duplicating it across feature folders.

## Mutations live in the feature, not the route

`useMutation` hooks belong in the feature folder (`use-<resource>-mutations.ts`, `<resource>/mutations.ts`),
one hook per write operation. Each hook owns the resource's mutation contract: invalidate that
resource's query key on success, toast on both success and error, `console.error` before the
toast on error. Call sites (usually `*-modals.tsx`) add their own `onSuccess`/`onError` for
call-site concerns (closing the modal, surfacing a field error) — React Query runs both the
hook's callbacks and the call site's, not one instead of the other.

```tsx
// features/<resource>/use-<resource>-mutations.ts
export function useDeleteResource() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (id: string | number) => deleteResource(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: resourceKeys.all })
      toast.success('Resource deleted.')
    },
    onError: (error) => {
      console.error('Failed to delete resource', error)
      toast.error(errorToastMessage(error))
    },
  })
}
```

Where a resource's create/update/delete flow is small enough to keep in one file, a single
`*-modals.tsx` component may inline its `useMutation` calls instead of a separate mutations
file. Either shape is fine — what matters is that the invalidate/toast/log contract above is
followed, and that a route component never defines a `useMutation` inline itself.

## Routes compose, features implement

A route file (`routes/.../<resource>.tsx`) owns the loader, search-param state, and modal state
(e.g. `useCrudModal`), then imports everything else from the matching feature folder:

```tsx
import { createResourceColumns, ResourceMobileCard } from '@/features/<resource>/columns'
import { ResourceDetail } from '@/features/<resource>/<resource>-detail'
import { ResourceForm } from '@/features/<resource>/<resource>-form'
import { useDeleteResource } from '@/features/<resource>/use-<resource>-mutations'
```

If a route file starts accumulating JSX or mutation logic beyond wiring these pieces together,
that logic belongs in the feature folder, not the route.

## Naming

- `create<Resource>Columns` for the column factory, `<Resource>MobileCard` for its mobile
  counterpart.
- `<Resource>Detail`, `<Resource>Form`, `<Resource>Modals` — PascalCase component, kebab-case
  filename matching it.
- `use<Verb><Resource>` for a single mutation hook (`useDeleteResource`), or a flat `mutations.ts`
  exporting several when the resource has more than a couple of writes (`<resource>/mutations.ts`).
