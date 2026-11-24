<script lang="ts">
import VirtualTable, {type Item} from "../src/VirtualTable.svelte";

export let count: number = 1000000
export let columnCount: number = 40

interface Column {
  name: string;
  width: number;
}

let scrollLeft: number = 0;
let scrollTop: number = 0;

const columns: Column[] = [
  {name: '#', width: 90},
  ...Array.from({length: columnCount}, (_, i) => ({
    name: `Column ${i + 1}`,
    // Vary the widths so the columns don't all line up the same way.
    width: [120, 160, 200, 140][i % 4],
  })),
]

// Cell values are derived from the row and column indices instead of stored, so a million rows with many columns
// doesn't hold tens of millions of strings in memory. All rows share one empty `values` array.
const noValues: any[] = [];
const items: Item[] = Array.from({length: count}, (_, i) => ({
  rowHeight: getRowHeight(i),
  values: noValues,
}));

function cellValue(rowIndex: number, colIndex: number): string {
  return `R${rowIndex + 1} C${colIndex}`;
}

let totalWidth: number = columns.reduce((acc, x) => acc + x.width, 0);

let startIndex: number = 0;
let endIndex: number = 0;
let bottomReachedCount: number = 0;

function getRowHeight(rowIndex: number) {
  // Simulate variable heights
  if ((rowIndex % 10) === 0) {
    return 28 * 2;
  } else {
    return 28;
  }
}
</script>

<div class="container">
  <div>
    Rows: {startIndex + 1} - {endIndex + 1}, total: {items.length}, columns: {columns.length},
    bottom reached: {bottomReachedCount} time(s)
  </div>
  <VirtualTable
    let:item
    let:index
    bind:startIndex
    bind:endIndex
    items={items}
    initialScrollLeft={scrollLeft}
    initialScrollTop={scrollTop}
    onBottomReached={() => {
      bottomReachedCount++;
      console.log("onBottomReached")
    }}
    onScrolled={(scrollLeft_, scrollTop_) => {
      scrollLeft = scrollLeft_;
      scrollTop = scrollTop_;
    }}
  >
    <div slot="header" class="header" style="min-width: {totalWidth}px;width: {totalWidth}px;max-width: {totalWidth}px;">
      {#each columns as column, index (index)}
        <div style="min-width: {column.width}px;width: {column.width}px;max-width: {column.width}px;" class:sticky-column={index === 0} >{column.name}</div>
      {/each}
    </div>
    <div class="row" style="height: {item.rowHeight}px;">
      {#each columns as column, colIndex (colIndex)}
        <div style="min-width: {column.width}px;width: {column.width}px;max-width: {column.width}px;" class:number={colIndex === 0} class:sticky-column={colIndex === 0}>
          {#if colIndex === 0}
            {index + 1}
          {:else}
            {cellValue(index, colIndex)}
          {/if}
        </div>
      {/each}
    </div>
  </VirtualTable>
</div>

<style>
  .container {
    height: 400px;
    width: 600px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    justify-content: stretch;
    border: 2px solid #333;
    position: relative;
    margin: 10px;
    /* Drag the bottom-right corner to resize the table. `resize` needs a non-visible overflow. */
    resize: both;
    overflow: hidden;
  }

  .header {
    border-left: 1px #ccc solid;
    font-size: 16px;
    display: flex;
    align-items: stretch;
    font-weight: bold;
    white-space: pre;
  }

  .header > div {
    display: block;
    box-sizing: border-box;
    padding: 4px;
    background: #eee;
    border-top: 1px #ccc solid;
    border-right: 1px #ccc solid;
    border-bottom: 1px #ccc solid;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: pre;
  }

  .row {
    font-size: 16px;
    display: flex;
    align-items: stretch;
    justify-content: stretch;
    border-left: 1px #ccc solid;
  }

  .row > div {
    box-sizing: border-box;
    padding: 4px;
    border-right: 1px #ccc solid;
    border-bottom: 1px #ccc solid;
    white-space: pre;
    overflow: hidden;
    text-overflow: ellipsis;
    background-color: #fff;
  }

  .row > div.number {
    background-color: #eee;
  }

  .sticky-column {
    position: sticky;
    left: 0;
    z-index: 10;
  }
</style>
