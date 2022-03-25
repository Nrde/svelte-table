<script>
  import { createEventDispatcher } from "svelte";

  /** @type {Array<Object>} */
  export let columns;

  /** @type {Array<Object>} */
  export let rows;

  /** @type {Array<Object>} */
  export let c_rows;

  /** @type {Array<number>} */
  export let sortOrders = [1, -1];

  // READ AND WRITE

  /** @type {string} */
  export let sortBy = "";

  /** @type {number} */
  export let sortOrder = sortOrders?.[0] || 1;

  /** @type {Object} */
  export let filterSelections = {};

  // expand
  /** @type {Array.<string|number>} */
  export let expanded = [];

  // READ ONLY

  /** @type {string} */
  export let expandRowKey = null;

  /** @type {string} */
  export let expandSingle = false;

  /** @type {string} */
  export let iconAsc = "▲";

  /** @type {string} */
  export let iconDesc = "▼";

  /** @type {string} */
  export let iconSortable = "";

  /** @type {string} */
  export let iconExpand = "▼";

  /** @type {string} */
  export let iconExpanded = "▲";

  /** @type {boolean} */
  export let showExpandIcon = false;

  /** @type {string} */
  export let classNameTable = "";

  /** @type {string} */
  export let classNameThead = "";

  /** @type {string} */
  export let classNameTbody = "";

  /** @type {string} */
  export let classNameSelect = "";

  /** @type {string} */
  export let classNameInput = "";

  /** @type {string} */
  export let classNameRow = "";

  /** @type {string} */
  export let classNameCell = "";

  /** @type {string} class added to the expanded row*/
  export let classNameRowExpanded = "";

  /** @type {string} class added to the expanded row*/
  export let classNameExpandedContent = "";

  /** @type {string} class added to the cell that allows expanding/closing */
  export let classNameCellExpand = "";

  const dispatch = createEventDispatcher();

  export let cutLimit = null;
  
  let sortFunction = () => "";

  // Validation
  if (!Array.isArray(expanded)) throw "'expanded' needs to be an array";

  let showFilterHeader = columns.some(c => {
    // check if there are any filter or search headers
    return c.filterOptions !== undefined || c.searchValue !== undefined;
  });
  let filterValues = {};
  let columnByKey;
  $: {
    columnByKey = {};
    columns.forEach(col => {
      columnByKey[col.key] = col;
    });
  }

  $: colspan = (showExpandIcon ? 1 : 0) + columns.length;

  $: c_rows = rows
    .filter(r => {
      // get search and filter results/matches
      return Object.keys(filterSelections).every(f => {
        // check search (text input) matches
        let resSearch =
          filterSelections[f] === "" ||
          (columnByKey[f].searchValue &&
            (columnByKey[f].searchValue(r) + "")
              .toLocaleLowerCase()
              .indexOf((filterSelections[f] + "").toLocaleLowerCase()) >= 0);

        // check filter (dropdown) matches
        let resFilter =
          resSearch ||
          filterSelections[f] === undefined ||
          // default to value() if filterValue() not provided in col
          filterSelections[f] ===
            (typeof columnByKey[f].filterValue === "function"
              ? columnByKey[f].filterValue(r)
              : columnByKey[f].value(r));
        return resFilter;
      });
    })
    .map(r =>
      Object.assign({}, r, {
        // internal row property for sort order
        $sortOn: sortFunction(r),
        // internal row property for expanded rows
        $expanded:
          expandRowKey !== null && expanded.indexOf(r[expandRowKey]) >= 0,
      })
    )
    .sort((a, b) => {
      if (!sortBy) return 0;
      else if (a.$sortOn > b.$sortOn) return sortOrder;
      else if (a.$sortOn < b.$sortOn) return -sortOrder;
      return 0;
    });

  const asStringArray = v =>
    []
      .concat(v)
      .filter(v => typeof v === "string" && v !== "")
      .join(" ");

  const calculateFilterValues = () => {
    filterValues = {};
    columns.forEach(c => {
      if (typeof c.filterOptions === "function") {
        filterValues[c.key] = c.filterOptions(rows);
      } else if (Array.isArray(c.filterOptions)) {
        // if array of strings is provided, use it for name and value
        filterValues[c.key] = c.filterOptions.map(val => ({
          name: val,
          value: val,
        }));
      }
    });
  };

  $: {
    let col = columnByKey[sortBy];
    if (
      col !== undefined &&
      col.sortable === true &&
      typeof col.value === "function"
    ) {
      sortFunction = r => col.value(r);
    }
  }

  $: {
    // if filters are enabled, watch rows and columns
    if (showFilterHeader && columns && rows) {
      calculateFilterValues();
    }
  }

  const updateSortOrder = colKey => {
    return colKey === sortBy
      ? sortOrders[
          (sortOrders.findIndex(o => o === sortOrder) + 1) % sortOrders.length
        ]
      : sortOrders[0];
  };

  const handleClickCol = (event, col) => {
    if (col.sortable) {
      sortOrder = updateSortOrder(col.key);
      sortBy = sortOrder ? col.key : undefined;
    }
    dispatch("clickCol", { event, col, key: col.key });
  };

  const handleClickRow = (event, row) => {
    dispatch("clickRow", { event, row });
  };

  const handleClickExpand = (event, row) => {
    row.$expanded = !row.$expanded;
    const keyVal = row[expandRowKey];
    if (expandSingle && row.$expanded) {
      expanded = [keyVal];
    } else if (expandSingle) {
      expanded = [];
    } else if (!row.$expanded) {
      expanded = expanded.filter(r => r != keyVal);
    } else {
      expanded = [...expanded, keyVal];
    }
    dispatch("clickExpand", { event, row });
  };

  const handleClickCell = (event, row, key) => {
    dispatch("clickCell", { event, row, key });
  };
</script>

<table class={asStringArray(classNameTable)}>
  <thead class={asStringArray(classNameThead)}>
    {#if showFilterHeader}
      <tr>
        {#each columns as col}
          <th class={asStringArray([col.headerFilterClass])}>
            {#if col.searchValue !== undefined}
              <input
                bind:value={filterSelections[col.key]}
                class={asStringArray(classNameInput)}
              />
            {:else if filterValues[col.key] !== undefined}
              <select
                bind:value={filterSelections[col.key]}
                class={asStringArray(classNameSelect)}
              >
                <option value={undefined} />
                {#each filterValues[col.key] as option}
                  <option value={option.value}>{option.name}</option>
                {/each}
              </select>
            {/if}
          </th>
        {/each}
        {#if showExpandIcon}
          <th />
        {/if}
      </tr>
    {/if}
    <slot name="header" {sortOrder} {sortBy}>
      <tr>
        {#each columns as col}
          <th
            on:click={e => handleClickCol(e, col)}
            class={asStringArray([
              col.sortable ? "isSortable" : "",
              col.headerClass,
            ])}
          >
            {col.title}
            {#if sortBy === col.key}
              {@html sortOrder === 1 ? iconAsc : iconDesc}
            {:else if col.sortable}
              {@html iconSortable}
            {/if}
          </th>
        {/each}
        {#if showExpandIcon}
          <th />
        {/if}
      </tr>
    </slot>
  </thead>

  <tbody class={asStringArray(classNameTbody)}>
    {#each c_rows as row, n}
      <slot name="row" {row} {n}>
        <tr
          on:click={e => {
            handleClickRow(e, row);
          }}
          class={asStringArray([
            classNameRow,
            row.$expanded && classNameRowExpanded,
          ])}
          class:cutLimit={n >= cutLimit}
          class:cutLimitBorder={n == cutLimit}
        >
          {#each columns as col}
            <td
              on:click={e => {
                handleClickCell(e, row, col.key);
              }}
              class={asStringArray([col.class, classNameCell])}
            >
              {#if col.renderComponent}
                <svelte:component
                  this={col.renderComponent.component || col.renderComponent}
                  {...col.renderComponent.props || {}}
                  {row}
                  {col}
                />
              {:else}
                {@html col.renderValue ? col.renderValue(row) : col.value(row)}
              {/if}
            </td>
          {/each}
          {#if showExpandIcon}
            <td
              on:click={e => handleClickExpand(e, row)}
              class={asStringArray(["isClickable", classNameCellExpand])}
            >
              {@html row.$expanded ? iconExpand : iconExpanded}
            </td>
          {/if}
        </tr>
        {#if row.$expanded}
          <tr class={asStringArray(classNameExpandedContent)}
            ><td {colspan}>
              <slot name="expanded" {row} {n} />
            </td></tr
          >
        {/if}
      </slot>
    {/each}
  </tbody>
</table>

<style>
  table {
    width: 100%;
  }
  .isSortable {
    cursor: pointer;
  }

  .isClickable {
    cursor: pointer;
  }

  tr th select {
    width: 100%;
  }
  
  table, th, td {
    border: 1px solid;
    border-collapse: collapse;
    margin-bottom: 10px;
  }

  table.redTable {
    border: 2px solid #A40808;
    background-color: #EEE7DB;
    width: 100%;
    text-align: center;
    border-collapse: collapse;
  }
  table.redTable td, table.redTable th {
    border: 1px solid #AAAAAA;
    padding: 3px 2px;
  }
  table.redTable tbody td {
    font-size: 13px;
  }
  table.redTable tr:nth-child(even) {
    background: #F5C8BF;
  }
  table.redTable thead {
    background: #A40808;
    background: -moz-linear-gradient(top, #bb4646 0%, #ad2020 66%, #A40808 100%);
    background: -webkit-linear-gradient(top, #bb4646 0%, #ad2020 66%, #A40808 100%);
    background: linear-gradient(to bottom, #bb4646 0%, #ad2020 66%, #A40808 100%);
  }
  table.redTable thead th {
    font-size: 19px;
    font-weight: bold;
    color: #FFFFFF;
    text-align: center;
    border-left: 2px solid #A40808;
  }
  table.redTable thead th:first-child {
    border-left: none;
  }

  table.blueTable {
    border: 1px solid #1C6EA4;
    background-color: #EEEEEE;
    width: 100%;
    text-align: left;
    border-collapse: collapse;
  }
  table.blueTable td, table.blueTable th {
    border: 1px solid #AAAAAA;
    padding: 3px 2px;
  }
  table.blueTable tbody td {
    font-size: 13px;
  }
  table.blueTable tbody tr:nth-child(even) {
    background: #D0E4F5;
  }
  table.blueTable thead {
    background: #1C6EA4;
    background: linear-gradient(to bottom, #5592bb 0%, #327cad 66%, #1C6EA4 100%);
    border-bottom: 2px solid #444444;
  }
  table.blueTable thead th {
    font-size: 15px;
    font-weight: bold;
    color: #FFFFFF;
    border-left: 2px solid #D0E4F5;
  }
  table.blueTable thead th:first-child {
    border-left: none;
  }

  .cutLimit {
    filter: brightness(85%);
  }
  table.blueTable tbody tr.cutLimitBorder {
    border-top: 3px solid #2f2f2f;
  }
</style>
