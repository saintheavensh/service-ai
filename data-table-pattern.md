# Reusable Data Table Pattern

This guide documents the implementation of the advanced Data Table component used in this project. It is built on top of [TanStack Table](https://tanstack.com/table/v8) and [shadcn-svelte](https://www.shadcn-svelte.com/), featuring search, faceted filtering, date range filtering, and row actions.

## 1. Component Architecture

The data table system is modular, consisting of shared core components and feature-specific implementations.

### Shared Core Components (`src/lib/components/ui/data-table/`)

These files act as the foundation and should be reused across all tables:

-   `data-table.svelte`: The main orchestrator. Initializes the table instance.
-   `data-table-checkbox.svelte`: Wrapper for row selection checkboxes.
-   `data-table-pagination.svelte`: Pagination controls and page size selector.
-   `data-table-column-header.svelte`: Sortable column headers with "Hide" option.
-   `data-table-view-options.svelte`: Dropdown to toggle column visibility.
-   `data-table-toolbar.svelte`: (Base) Can be customized per feature.
-   `data-table-faceted-filter.svelte`: Filter dropdown for categorical data (e.g., Status, Category).
-   `render-helpers.ts`: Utilities (`renderComponent`, `renderSnippet`) for Svelte 5 interoperability.

### Feature-Specific Implementation

For each feature (e.g., `products`, `tasks`), create a dedicated directory (e.g., `src/routes/products/`) containing:

-   `+page.svelte`: The page rendering the `DataTable`.
-   `columns.ts`: Column definitions using `renderSnippet`.
-   `data-table-toolbar.svelte`: Customized toolbar with specific filters.
-   `components/data-table-actions.svelte`: Row-level actions (Edit, Delete).

---

## 2. Implementation Guide

### A. Main Page Integration (`+page.svelte`)

The page fetches data and passes it to the `DataTable`. It also provides a **Refresh Context** for child components (like Actions) to trigger re-fetching.

```svelte
<script lang="ts">
    import { onMount, setContext } from "svelte";
    import DataTable from "./data-table.svelte";
    import { columns } from "./columns";

    let data = $state([]);

    async function fetchData() {
        data = await Service.getAll(); // Fetch data
    }

    // Provide context for child components (Actions) to refresh the table
    setContext("refreshTable", fetchData);

    onMount(fetchData);
</script>

<DataTable {data} {columns} />
```

### B. Defining Columns (`columns.ts`)

We use `renderSnippet` to render Svelte snippets within table cells. This allows full usage of Svelte's template syntax inside standard JS column definitions.

```typescript
import { createRawSnippet } from "svelte";
import { renderComponent, renderSnippet } from "$lib/components/ui/data-table/index.js";
import DataTableActions from "./components/data-table-actions.svelte";

export const columns: ColumnDef<Product>[] = [
    // 1. Text Column
    {
        accessorKey: "name",
        header: "Name",
        cell: ({ row }) => {
            const snippet = createRawSnippet<[any]>(() => ({
                render: () => `<div class="font-bold">${row.getValue("name")}</div>`
            }));
            return renderSnippet(snippet, {});
        }
    },
    // 2. Action Component
    {
        id: "actions",
        cell: ({ row }) => renderComponent(DataTableActions, { row }),
    }
];
```

### C. Custom Toolbar (`data-table-toolbar.svelte`)

Interact with the table instance to filter data.

**Key Pattern: Faceted Filter**
Dynamically generate options from the table data to avoid hardcoded lists.

```svelte
<script lang="ts">
    // ... imports
    let { table } = $props();

    // Derive unique categories from current data
    let categories = $derived.by(() => {
        const unique = new Map();
        table.getCoreRowModel().rows.forEach(row => {
            const val = row.original.category?.name;
            if (val) unique.set(val, val);
        });
        return Array.from(unique).map(([v, l]) => ({ value: v, label: l }));
    });
    
    let categoryColumn = $derived(table.getColumn("category"));
</script>

{#if categoryColumn}
    <DataTableFacetedFilter column={categoryColumn} options={categories} />
{/if}
```

### D. Row Actions & Dialogs (`data-table-actions.svelte`)

Use a dropdown menu for actions. For "Edit", reuse the creation dialog but pass the row data.

**Context Usage:**
Consume the `refreshTable` context to reload data after an action.

```svelte
<script lang="ts">
    import { getContext } from "svelte";
    import ProductDialog from "./product-dialog.svelte";
    
    let { row } = $props();
    let showEdit = $state(false);
    
    // 1. Get Refresh Function
    const refreshTable = getContext<() => void>("refreshTable");
</script>

<!-- Action Menu -->
<DropdownMenu.Item onclick={() => showEdit = true}>Edit</DropdownMenu.Item>

<!-- Edit Dialog -->
{#if showEdit}
    <!-- 2. Bind Open State & Pass Data -->
    <ProductDialog 
        bind:open={showEdit} 
        productToEdit={row.original} 
        showTrigger={false} // Hide the redundant "New" button
        onSuccess={() => {
            refreshTable?.(); // 3. Refresh Table on Success
            showEdit = false;
        }} 
    />
{/if}
```

---

## 3. Recommended Data Structure

To ensure type safety and seamless integration, define clear TypeScript interfaces for your data.

### Example Schema (`types.ts`)

```typescript
// Core Data Entity
export type Product = {
    id: string; // Unique identifier (required for row selection/actions)
    code: string | null;
    name: string;
    stock: number;
    minStock: number | null;
    createdAt: string | null; // ISO Date string recommended

    // Relations (flattened or nested)
    categoryId: string | null;
    category?: {
        name: string; // Used for "Category" column & filters
    };
    
    // Nested Details (for Dialogs/Expanded Rows)
    batches?: Batch[];
};

export type Batch = {
    id: string;
    sku: string;
    currentStock: number;
    buyPrice: number;
    sellPrice: number;
    supplier?: { name: string };
};
```

### Tips for Data Shaping

1.  **Flatten Relations**: If possible, flatten simple relations (e.g., `categoryName`) for easier sorting/filtering.
2.  **Date Formats**: Return dates as ISO strings from the backend and format them in the column definition.
3.  **Null Safety**: Handle potential `null` or `undefined` values in your column render functions (use `|| "-"`).

## 4. Best Practices

1.  **State Management**: Use Svelte 5 runes (`$state`, `$derived`) for reactivity.
2.  **Linting**: Use `untrack` inside `$effect` if you need to update table state based on dependencies to avoid infinite loops.
3.  **Type Safety**: Always define a TS interface for your row data (e.g., `type Product = { ... }`).
4.  **Optimistic UI**: Use `toast` notifications to give immediate feedback while `refreshTable` reloads the latest data.
