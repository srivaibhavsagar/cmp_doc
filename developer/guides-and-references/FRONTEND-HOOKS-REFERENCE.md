# Frontend Hooks Reference

Reusable React hooks located in `frontend/src/hooks/`. Use these instead of reimplementing common patterns in page components.

## `usePagination<T>(items, options?)`

Client-side pagination for slicing arrays into pages.

**Import:** `import { usePagination } from '@/hooks/usePagination'`

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `items` | `T[]` | Full array to paginate |
| `options.pageSize` | `number` | Items per page (default: `10`) |
| `options.initialPage` | `number` | Starting page, 1-indexed (default: `1`) |

**Returns:**

| Property | Type | Description |
|----------|------|-------------|
| `pageItems` | `T[]` | Items for the current page |
| `currentPage` | `number` | Current page number (1-indexed) |
| `totalPages` | `number` | Total page count |
| `totalItems` | `number` | Total item count |
| `pageSize` | `number` | Items per page |
| `goToPage(n)` | `(page: number) => void` | Navigate to a specific page |
| `nextPage()` | `() => void` | Go to next page |
| `prevPage()` | `() => void` | Go to previous page |
| `hasNext` | `boolean` | Whether a next page exists |
| `hasPrev` | `boolean` | Whether a previous page exists |

**Example:**

```tsx
const { pageItems, currentPage, totalPages, goToPage, hasNext, hasPrev } =
  usePagination(filteredResources, { pageSize: 20 })

return (
  <>
    <ResourceTable items={pageItems} />
    <Pagination
      current={currentPage}
      total={totalPages}
      onPageChange={goToPage}
      hasNext={hasNext}
      hasPrev={hasPrev}
    />
  </>
)
```

**Notes:**
- Automatically clamps the current page to a valid range when the source array changes (e.g., after filtering).
- Memoizes `pageItems` — only recalculates when `items`, page, or `pageSize` change.
- Pair with the existing `<Pagination />` component for a consistent UI.

---

## `useFetch<T>(fetcher, deps?, options?)`

Generic data-fetching hook with loading, error, and refetch support.

**Import:** `import { useFetch } from '@/hooks/useFetch'`

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `fetcher` | `() => Promise<AxiosResponse<T>>` | Function returning an Axios call |
| `deps` | `unknown[]` | Dependency array — refetches when values change |
| `options.immediate` | `boolean` | Whether to fetch on mount (default: `true`) |

**Returns:** `{ data, loading, error, refetch }`

**Example:**

```tsx
const { data: catalogs, loading, error, refetch } = useFetch(
  () => catalogApi.list(tenantId),
  [tenantId]
)
```

**Notes:**
- Safe against state updates on unmounted components (internal `mountedRef`).
- Set `immediate: false` for conditional fetching triggered by user action.

---

## `useDebounce<T>(value, delay?)`

Debounces a value by the specified delay. Ideal for search inputs.

**Import:** `import { useDebounce } from '@/hooks/useDebounce'`

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `value` | `T` | Value to debounce |
| `delay` | `number` | Debounce delay in ms (default: `300`) |

**Returns:** `T` — the debounced value

**Example:**

```tsx
const [search, setSearch] = useState('')
const debouncedSearch = useDebounce(search, 300)

useEffect(() => {
  fetchResults(debouncedSearch)
}, [debouncedSearch])
```

---

## `useBodyOverflow(isOpen)`

Prevents body scrolling when a modal or drawer is open.

**Import:** `import { useBodyOverflow } from '@/hooks/useBodyOverflow'`

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `isOpen` | `boolean` | Whether the overlay is currently open |

**Example:**

```tsx
const [modalOpen, setModalOpen] = useState(false)
useBodyOverflow(modalOpen)
```

**Notes:**
- Automatically restores `overflow: unset` on close or unmount.
- Use in any component that renders a full-screen overlay (modals, drawers, command palette).
