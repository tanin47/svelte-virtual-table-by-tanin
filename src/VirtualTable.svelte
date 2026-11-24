<script lang="ts" context="module">
export interface Item {
  values: any[]
  rowHeight: number
}
</script>
<script lang="ts">
import {onMount, tick} from 'svelte';

export let items: Item[];
export let initialScrollLeft: number = 0;
export let initialScrollTop: number = 0;
export let onBottomReached: (() => void) | null = null;
export let onScrolled: ((scrollLeft: number, scrollTop: number) => void) | null = null;
// Extra pixels of rows to render above and below the viewport, so fast scrolling doesn't reveal blank space.
export let overscan: number = 500;

let bottomElement: HTMLElement;

// read-only, but visible to consumers via bind:start
export let startIndex = 0;
export let endIndex = 0;

// local state
let viewport: HTMLDivElement;
let viewportHeight: number = 0;
let stickyHeaderHeight: number = 0;
// Indices of the rows to render. The template reads `items[index]`, so it updates on its own when `items` changes.
// These are plain numbers, not row objects: in legacy (non-runes) mode Svelte treats every object as changed, so a list
// of objects would re-render every row whenever the list is rebuilt. With numbers, only rows entering the range render.
let visible: number[] = [];

let cumulativeHeight: number[] = [];

const MAX_SCROLLABLE_AREA = 16000000; // Chrome cannot have a div, padding, or anything larger than 16,777,200px
let contentTop: number = 0;

// The rendered rows live in a block pinned to the viewport with `position: sticky`. The browser scrolls on its
// compositor thread before the scroll handler runs, so rows positioned in the scrollable content can be moved entirely
// out of view, e.g. dragging the scrollbar of a tall table moves thousands of pixels per frame and shows a blank table.
// A pinned block stays in view, so until the handler runs, the previous rows show instead of a blank table.
// pinnedTop is where the block sits in the content's coordinates, mirroring the browser's sticky positioning: it follows
// scrollTop but stops at the end of the content.
let pinnedTop: number = 0;
let pinnedHeight: number = 0;

// `visible` is updated imperatively by handleScroll() rather than via a `$:` block. A reactive block cannot see that
// startIndex/endIndex are changed inside handleScroll() (called from refresh()), so depending on block ordering it could
// compute `visible` from stale indices, e.g. showing only 5 rows after items grows from 5 to 1000 until the user scrolls.
// It is only rebuilt when the range changes, so a scroll within the same rows doesn't touch the rows.
let visibleStartIndex = -1;
let visibleEndIndex = -1;
function updateVisible(start: number, end: number) {
  if (start === visibleStartIndex && end === visibleEndIndex) { return; }
  visibleStartIndex = start;
  visibleEndIndex = end;
  visible = Array.from({length: end - start + 1}, (_, i) => start + i);
}

// Row heights depend only on `items`, so they're recomputed only when `items` changes, not when the viewport resizes.
// The array is built locally and assigned once: in legacy mode, each write to a component array is a tracked signal
// write, which is costly when repeated for every row.
function computeCumulativeHeight(items: Item[]): number[] {
  const result = new Array<number>(items.length);
  let total = 0;
  for (let i = 0; i < items.length; i++) {
    total += items[i].rowHeight;
    result[i] = total;
  }
  return result;
}

$: cumulativeHeight = computeCumulativeHeight(items);

// `cumulativeHeight` is passed explicitly so this block is ordered after the one above. Legacy `$:` blocks are ordered
// by the variables they reference directly, and refresh() reading it internally doesn't count.
$: if (viewport) {
  refresh(items, cumulativeHeight, viewportHeight);
}

// The initial scroll position is applied once, on the first refresh that has rows (so rows loaded asynchronously after
// mount still get it). Re-applying it on every refresh would move the user's scroll position whenever the table resized
// or rows were added, and would interrupt a fling in progress when "load more" adds rows.
let initialScrollApplied = false;

async function refresh(_items: Item[], _cumulativeHeight: number[], _viewportHeight: number) {
  if (initialScrollApplied || items.length === 0) {
    await handleScroll()
    return;
  }
  initialScrollApplied = true;
  // Read these before handleScroll(): it calls onScrolled() with the current position (0 on mount), and a consumer that
  // feeds onScrolled() back into initialScrollLeft/initialScrollTop would overwrite the position to restore.
  const scrollLeft = initialScrollLeft;
  const scrollTop = initialScrollTop;
  await handleScroll()
  await tick()
  viewport.scrollTo(scrollLeft, scrollTop)
}

// The binary search is needed. The flickering while scrolling would be noticeable at >1,000,000 items if we were to use
// a brute-force search.
function findItemIndexByTop(top: number): number {
  let left = 0;
  let right = items.length - 1;
  let newStartIndex: number = items.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    const midHeight = cumulativeHeight[mid];

    if (midHeight > top) {
      newStartIndex = mid;
      right = mid - 1;
    } else {
      left = mid + 1;
    }
  }

  // If we found an index, we need to ensure it's the first one that exceeds trueScrollTop
  while (newStartIndex !== null && newStartIndex > 0 && cumulativeHeight[newStartIndex - 1] > top) {
    newStartIndex--;
  }

  return newStartIndex;
}

async function handleScroll() {
  if (items.length === 0) {
    visible = [];
    visibleStartIndex = -1;
    visibleEndIndex = -1;
    return;
  }
  let { scrollLeft, scrollTop } = viewport;
  let scrollableHeight = cumulativeHeight[items.length - 1]
  let contentViewHeight = viewportHeight - stickyHeaderHeight;

  let trueTop: number = scrollTop;

  // In the virtual mode, we need to scale the true top properly.
  if (cumulativeHeight[items.length - 1] >= MAX_SCROLLABLE_AREA) {
    scrollableHeight = MAX_SCROLLABLE_AREA;
    trueTop = Math.ceil(cumulativeHeight[items.length - 1] * scrollTop / scrollableHeight);
  }

  startIndex = findItemIndexByTop(trueTop);
  endIndex = findItemIndexByTop(trueTop + contentViewHeight);
  let deltaTop = trueTop - (cumulativeHeight[startIndex] - items[startIndex].rowHeight);
  contentTop = scrollTop - deltaTop;

  if (
     contentTop + cumulativeHeight[endIndex] - (cumulativeHeight[startIndex] - items[startIndex].rowHeight) >= scrollableHeight
  ) {
    // Handle over-scrolling by simply showing the last rows according to the contentViewHeight.
    // If this wasn't handled, it would flicker because it automatically adjusts back to the max scroll top that isn't overscrolled.
    endIndex = items.length - 1;
    startIndex = endIndex
    while (startIndex > 0) {
      if ((cumulativeHeight[endIndex] - cumulativeHeight[startIndex] + items[startIndex].rowHeight) < contentViewHeight) {
        startIndex--;
      } else {
        break
      }
    }

    trueTop = cumulativeHeight[startIndex] - items[startIndex].rowHeight
    let deltaTop = (cumulativeHeight[endIndex] - cumulativeHeight[startIndex] + items[startIndex].rowHeight) - contentViewHeight
    contentTop = scrollableHeight - contentViewHeight - deltaTop;
  }

  // Overscan: also render rows above and below the viewport. The browser scrolls on the compositor thread before this
  // handler runs, so during a fast scroll it can show area beyond the exactly-fitting rows. Without the buffer, that area
  // flashes blank. startIndex/endIndex keep reporting only the rows within the viewport.
  // At this point, contentTop is where the startIndex row's top sits in the viewport's scroll coordinates.
  const rowTop = (i: number) => cumulativeHeight[i] - items[i].rowHeight;
  const startRowTop = rowTop(startIndex);
  let renderStartIndex = Math.min(startIndex, findItemIndexByTop(Math.max(0, startRowTop - overscan)));
  let renderEndIndex = Math.max(endIndex, findItemIndexByTop(cumulativeHeight[endIndex] + overscan));
  let renderContentTop = contentTop - (startRowTop - rowTop(renderStartIndex));
  // Never render past the end of the scrollable area, or the browser would grow the scrollable area and flicker.
  while (
    renderEndIndex > endIndex &&
    renderContentTop + cumulativeHeight[renderEndIndex] - rowTop(renderStartIndex) > scrollableHeight
  ) {
    renderEndIndex--;
  }
  contentTop = renderContentTop;
  pinnedHeight = Math.max(0, Math.min(contentViewHeight, scrollableHeight));
  pinnedTop = Math.max(0, Math.min(scrollTop, scrollableHeight - pinnedHeight));

  updateVisible(renderStartIndex, renderEndIndex);

  if (onScrolled) {
    onScrolled(scrollLeft, scrollTop);
  }
}

onMount(() => {
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        if (onBottomReached) onBottomReached();
      }
    });
  }, {
    threshold: 0.1,
  });

  if (bottomElement) {
    observer.observe(bottomElement);
  }

  return () => observer.disconnect();
});

const hasHeaderSlot = $$slots.header;
</script>

<div
  bind:this={viewport}
  bind:offsetHeight={viewportHeight}
  class="virtual-table"
  onscroll={handleScroll}
>
  {#if hasHeaderSlot}
    <!-- we need the header here because overflow: visible auto; isn't a valid combination. Therefore, the header cannot be outside of this component. -->
    <!-- We wouldn't be able to support scrolling to the right where both rows and the header scroll together -->
    <!-- The header must be sticker with top-0 -->
    <div bind:offsetHeight={stickyHeaderHeight} class="virtual-table-header">
      <slot name="header" />
    </div>
  {/if}
  <div
    class="virtual-table-content"
    style="height: {Math.min(cumulativeHeight[items.length - 1], MAX_SCROLLABLE_AREA)}px;"
  >
    <div class="virtual-table-pinned" style="top: {stickyHeaderHeight}px; height: {pinnedHeight}px;">
      <div style="position: absolute; top: {contentTop - pinnedTop}px; left: 0;">
        {#each visible as index (index)}
          <div
            class="virtual-table-row"
            style="height: {items[index].rowHeight}px; min-height: {items[index].rowHeight}px; max-height: {items[index].rowHeight}px;"
          >
            <slot item={items[index]} {index}>Missing template</slot>
          </div>
        {/each}
      </div>
    </div>
    <div bind:this={bottomElement} class="virtual-table-bottom"></div>
  </div>
</div>

<style>
.virtual-table {
  flex-direction: column;
  -webkit-overflow-scrolling: touch;
  align-items: stretch;
  box-sizing: border-box;
  flex-grow: 1;
  font-size: 0;
  justify-content: stretch;
  overflow: auto;
  position: relative;
}

.virtual-table-header {
  position: sticky;
  top: 0;
  z-index: 20;
}

.virtual-table-content {
  position: relative;
  overflow: visible;
  -webkit-overflow-scrolling: touch;
  display: block;
  font-size: 0;
}

.virtual-table-pinned {
  position: sticky;
  /* overflow stays visible: rows wider than the viewport must still widen the scrollable area for horizontal scroll. */
  overflow: visible;
}

.virtual-table-row {
  position: relative;
  box-sizing: border-box;
  font-size: 0;
}

.virtual-table-bottom {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  height: 10px;
  min-height: 10px;
  max-height: 10px;
}
</style>
