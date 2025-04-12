<script lang="ts">
    import { onMount } from 'svelte';
    import { writable, derived } from 'svelte/store';

    interface Column {
        id: string;
        name: string;
        isDefault?: boolean;
    }

    interface TableRow {
        internalId: string;
        selected: boolean;
        values: Record<string, string>;
    }

    const STORAGE_KEY_COLUMNS = 'table_formatter_columns';
    const STORAGE_KEY_DATA = 'table_formatter_data';

    // Initialize with default columns
    const defaultColumns: Column[] = [
        { id: 'id', name: 'ID', isDefault: true },
        { id: 'name', name: 'Name', isDefault: true },
        { id: 'category', name: 'Category', isDefault: true }
    ];

    let isInitialized = false;
    let isBrowser = false;
    const columns = writable<Column[]>(defaultColumns);
    const tableData = writable<TableRow[]>([]);
    let copyButtonText = 'Copy';
    let copyTimeout: number | undefined;
    const codeBlockFormatting = writable<boolean>(false);

    // Derived store for selected rows count
    const selectedCount = derived(tableData, $data => 
        $data.filter(row => row.selected).length
    );

    // Derived store for all rows selected state
    const allSelected = derived(tableData, $data => 
        $data.length > 0 && $data.every(row => row.selected)
    );

    // Derived store for ASCII table
    const asciiTable = derived([columns, tableData], ([$columns, $data]) => {
        if ($columns.length === 0 || $data.length === 0) return '';

        // Calculate maximum width for each column
        const columnWidths = new Map<string, number>();
        $columns.forEach(col => {
            const maxWidth = Math.max(
                col.name.length,
                ...$data.map(row => (row.values[col.id] || '').length)
            );
            columnWidths.set(col.id, maxWidth);
        });

        // Build header
        let table = '|';
        $columns.forEach(col => {
            const width = columnWidths.get(col.id) || 0;
            table += ` ${col.name.padEnd(width)} |`;
        });
        table += '\n|';

        // Build separator
        $columns.forEach(col => {
            const width = columnWidths.get(col.id) || 0;
            table += '-'.repeat(width + 2) + '|';
        });
        table += '\n';

        // Build rows
        $data.forEach(row => {
            table += '|';
            $columns.forEach(col => {
                const width = columnWidths.get(col.id) || 0;
                const value = row.values[col.id] || '';
                table += ` ${value.padEnd(width)} |`;
            });
            table += '\n';
        });

        return table;
    });

    // Derived store for formatted ASCII table (with optional Discord code block formatting)
    const formattedAsciiTable = derived([asciiTable, codeBlockFormatting], ([$asciiTable, $codeBlockFormatting]) => {
        if (!$asciiTable) return '';
        
        if ($codeBlockFormatting) {
            return '```\n' + $asciiTable + '```';
        }
        
        return $asciiTable;
    });

    // Function to generate a unique internal ID
    function generateInternalId(): string {
        return crypto.randomUUID();
    }

    // Function to add a new column
    function addColumn() {
        if (!newColumnName.trim()) return;
        
        const columnId = newColumnName.toLowerCase().replace(/\s+/g, '_');
        columns.update(cols => [...cols, { id: columnId, name: newColumnName }]);
        
        // Add the new column to all existing rows with empty values
        tableData.update(rows => 
            rows.map(row => ({
                ...row,
                values: { ...row.values, [columnId]: '' }
            }))
        );
        
        newColumnName = '';
    }

    // Function to delete a column
    function deleteColumn(columnId: string) {
        columns.update(cols => cols.filter(col => col.id !== columnId));
        
        // Remove the column from all rows
        tableData.update(rows => 
            rows.map(row => {
                const newValues = { ...row.values };
                delete newValues[columnId];
                return { ...row, values: newValues };
            })
        );
    }

    // Function to add a new row
    function addRow() {
        const newRow: TableRow = {
            internalId: generateInternalId(),
            selected: false,
            values: {}
        };
        
        // Initialize all columns with empty values
        $columns.forEach(col => {
            newRow.values[col.id] = '';
        });
        
        tableData.update(data => [...data, newRow]);
    }

    // Function to delete selected rows
    function deleteSelectedRows() {
        tableData.update(data => data.filter(row => !row.selected));
    }

    // Function to delete a specific row
    function deleteRow(internalId: string) {
        tableData.update(data => data.filter(row => row.internalId !== internalId));
    }

    // Function to toggle selection of all rows
    function toggleSelectAll(event: Event) {
        const checked = (event.target as HTMLInputElement).checked;
        tableData.update(data => data.map(row => ({ ...row, selected: checked })));
    }

    async function copyToClipboard() {
        if (!isBrowser) return;
        
        try {
            await navigator.clipboard.writeText($formattedAsciiTable);
            copyButtonText = 'Copied!';
            
            // Reset button text after 2 seconds
            if (copyTimeout) clearTimeout(copyTimeout);
            copyTimeout = window.setTimeout(() => {
                copyButtonText = 'Copy';
            }, 2000);
        } catch (err) {
            console.error('Failed to copy:', err);
            copyButtonText = 'Failed to copy';
            
            if (copyTimeout) clearTimeout(copyTimeout);
            copyTimeout = window.setTimeout(() => {
                copyButtonText = 'Copy';
            }, 2000);
        }
    }

    let newColumnName = '';

    // Setup browser-specific functionality
    onMount(() => {
        isBrowser = true;

        // Load saved data
        const savedColumns = localStorage.getItem(STORAGE_KEY_COLUMNS);
        if (savedColumns) {
            try {
                const parsedColumns = JSON.parse(savedColumns);
                columns.set(parsedColumns);
            } catch (e) {
                console.error('Failed to parse saved columns:', e);
                columns.set(defaultColumns);
            }
        }

        // Load table data
        const savedData = localStorage.getItem(STORAGE_KEY_DATA);
        if (savedData) {
            try {
                const parsedData = JSON.parse(savedData);
                tableData.set(parsedData);
            } catch (e) {
                console.error('Failed to parse saved data:', e);
                tableData.set([]);
            }
        } else {
            // Initialize with an empty row if no data exists
            const initialRow: TableRow = {
                internalId: generateInternalId(),
                selected: false,
                values: {}
            };
            $columns.forEach(col => {
                initialRow.values[col.id] = '';
            });
            tableData.set([initialRow]);
        }

        // Enable saving after initial data is loaded
        isInitialized = true;

        // Setup store subscriptions for saving
        const unsubscribeColumns = columns.subscribe($columns => {
            if (isInitialized) {
                localStorage.setItem(STORAGE_KEY_COLUMNS, JSON.stringify($columns));
            }
        });

        const unsubscribeData = tableData.subscribe($data => {
            if (isInitialized) {
                const dataToSave = $data.map(row => ({
                    ...row,
                    selected: false
                }));
                localStorage.setItem(STORAGE_KEY_DATA, JSON.stringify(dataToSave));
            }
        });

        return () => {
            if (copyTimeout) clearTimeout(copyTimeout);
            unsubscribeColumns();
            unsubscribeData();
        };
    });
</script>

<div class="w-full min-h-screen bg-base-200 p-2">
    <div class="sticky top-0 z-10 bg-base-100 shadow-md p-4 mb-4 flex flex-col gap-4">
        <div class="flex justify-between items-center">
            <div class="flex gap-4">
                <button class="btn btn-primary" on:click={addRow}>Add Row</button>
                {#if $selectedCount > 0}
                    <button class="btn btn-error" on:click={deleteSelectedRows}>
                        Delete Selected ({$selectedCount})
                    </button>
                {/if}
            </div>
            <div class="flex gap-2">
                <input
                    type="text"
                    bind:value={newColumnName}
                    placeholder="New column name"
                    class="input input-bordered input-sm"
                    on:keydown={(e) => e.key === 'Enter' && addColumn()}
                >
                <button class="btn btn-sm btn-primary" on:click={addColumn}>
                    Add Column
                </button>
            </div>
        </div>
    </div>

    <div class="grid grid-cols-2 gap-4">
        <!-- Editable Table -->
        <div class="card bg-base-100 shadow-xl overflow-hidden">
            <div class="card-body p-0">
                <div class="overflow-x-auto">
                    <table class="table table-zebra w-full">
                        <thead>
                            <tr>
                                <th class="w-12">
                                    <label>
                                        <input 
                                            type="checkbox" 
                                            class="checkbox" 
                                            on:change={toggleSelectAll}
                                            checked={$allSelected}
                                        >
                                    </label>
                                </th>
                                {#each $columns as column (column.id)}
                                    <th class="relative">
                                        <span class="pr-8">{column.name}</span>
                                        {#if !column.isDefault}
                                            <button 
                                                class="btn btn-ghost btn-xs absolute right-2 top-1/2 -translate-y-1/2 hover:bg-base-300"
                                                on:click={() => deleteColumn(column.id)}
                                                title="Delete column"
                                            >
                                                ×
                                            </button>
                                        {/if}
                                    </th>
                                {/each}
                                <th class="w-12">Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            {#each $tableData as row (row.internalId)}
                                <tr class="hover:bg-base-200">
                                    <td>
                                        <label>
                                            <input type="checkbox" class="checkbox" bind:checked={row.selected}>
                                        </label>
                                    </td>
                                    {#each $columns as column (column.id)}
                                        <td>
                                            <input 
                                                type="text" 
                                                bind:value={row.values[column.id]} 
                                                placeholder="Enter {column.name}" 
                                                class="input input-bordered input-sm w-full bg-transparent hover:bg-base-100 focus:bg-base-100"
                                            >
                                        </td>
                                    {/each}
                                    <td>
                                        <button 
                                            class="btn btn-ghost btn-sm hover:text-error" 
                                            on:click={() => deleteRow(row.internalId)} 
                                            aria-label="Delete row"
                                        >
                                            <svg 
                                                xmlns="http://www.w3.org/2000/svg" 
                                                width="16" 
                                                height="16" 
                                                viewBox="0 0 24 24" 
                                                fill="none" 
                                                stroke="currentColor" 
                                                stroke-width="2" 
                                                stroke-linecap="round" 
                                                stroke-linejoin="round" 
                                                class="w-4 h-4"
                                            >
                                                <path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/>
                                            </svg>
                                        </button>
                                    </td>
                                </tr>
                            {/each}
                        </tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- ASCII Table Preview -->
        <div class="card bg-base-100 shadow-xl h-fit sticky top-[88px]">
            <div class="card-body">
                <div class="flex justify-between items-center mb-2">
                    <h2 class="card-title">ASCII/Markdown Table Preview</h2>
                    <div class="flex gap-2 items-center">
                        <div class="form-control">
                            <label class="label cursor-pointer gap-2">
                                <span class="label-text text-xs">Encode as Code Block</span>
                                <input type="checkbox" class="toggle toggle-primary toggle-sm" bind:checked={$codeBlockFormatting} />
                            </label>
                        </div>
                        <button
                            class="btn btn-sm btn-primary gap-2"
                            on:click={copyToClipboard}
                            disabled={!$asciiTable}
                    >
                        <svg 
                            xmlns="http://www.w3.org/2000/svg" 
                            width="16" 
                            height="16" 
                            viewBox="0 0 24 24" 
                            fill="none" 
                            stroke="currentColor" 
                            stroke-width="2" 
                            stroke-linecap="round" 
                            stroke-linejoin="round"
                            class="w-4 h-4"
                        >
                            <rect width="14" height="14" x="8" y="8" rx="2" ry="2"/><path d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"/>
                        </svg>
                        {copyButtonText}
                        </button>
                    </div>
                </div>
                <div class="font-mono whitespace-pre overflow-x-auto bg-base-200 p-4 rounded-lg max-h-[calc(100vh-180px)] text-sm">
                    {$formattedAsciiTable}
                </div>
            </div>
        </div>
    </div>
</div>