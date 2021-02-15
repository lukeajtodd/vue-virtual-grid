<script lang="ts">
import { Prop, Component, Vue, ProvideReactive } from 'vue-property-decorator';
import {
    // getGridGapDefault,
    // getColumnCountDefault,
    // getWindowMarginDefault,
    // getItemRatioHeightDefault,
    debugLog,
    execFunc,
} from './utils';
import { Item } from './types';

interface ContainerData {
    size: ElementSize;
    scroll: ElementScroll;
    elementOffset: number;
    elementSize: ElementSize;
}

interface ElementSize {
    width: number;
    height: number;
}

interface ElementScroll {
    x: number;
    y: number;
}

interface ConfigData<P> {
    windowMargin: number;
    gridGap: number;
    columnCount: number;
    entries: Item<P>[];
}

interface Cell<P> extends Item<P> {
    columnNumber: number;
    rowNumber: number;
    offset: number;
}

interface LayoutData<P> {
    totalHeight: number;
    cells: Cell<P>[];
}

interface RenderData<P> {
    cellsToRender: Cell<P>[];
    firstRenderedRowNumber: number;
    firstRenderedRowOffset: number;
}

@Component
export default class VirtualGrid<P> extends Vue {
    @Prop({ required: true }) items: Item<P>[];
    @Prop({ default: () => () => true }) updateFunction: () => Promise<boolean>;
    @Prop({ default: 500 }) updateTriggerMargin: number;
    @Prop({ default: null }) loader: Vue.Component;
    @Prop({ default: false }) debug: boolean;
    @Prop({ default: false }) shrinkContent: boolean;
    @Prop({ default: null }) container: HTMLDivElement

    @ProvideReactive() updateLock: boolean = false;

    @ProvideReactive() bottomReached: boolean = false;

    @ProvideReactive() ref: Element = null;

    @ProvideReactive() containerData: ContainerData = {
        size: { height: 0, width: 0 },
        scroll: { x: 0, y: 0 },
        elementOffset: 0,
        elementSize: { height: 0, width: 0 },
    };

    get loadingBatch(): boolean {
        return this.loader && this.updateLock;
    }

    get configData(): ConfigData<P> {
        return this.computeConfigData(this.containerData, this.items);
    }

    get layoutData(): LayoutData<P> {
        return this.computeLayoutData(this.configData);
    }

    get renderData(): RenderData<P> {
        return this.computeRenderData(this.configData, this.containerData, this.layoutData);
    }

    get gridStyle() {
        const columns = this.shrinkContent ?
            `repeat(${this.configData.columnCount}, min-content)` :
            `repeat(${this.configData.columnCount}, 1fr)`;
        return {
            'grid-template-columns': columns,
            'gap': `${this.configData.gridGap}px`,
        };
    }

    mounted() {
        this.ref = this.$refs.virtualGrid as Element;
        this.initiliazeGrid();
        const winOrContainer = this.container ? this.container : window;
        winOrContainer.addEventListener('resize', this.resize);
        winOrContainer.addEventListener('scroll', this.scroll);
    }

    beforeDestroy() {
        const winOrContainer = this.container ? this.container : window;
        winOrContainer.removeEventListener('resize', this.resize);
        winOrContainer.removeEventListener('scroll', this.scroll);
    }

    resize(): void {
        this.loadMoreData();
    }

    scroll(): void {
        this.loadMoreData();
    }

    initiliazeGrid(): void {
        this.computeContainerData();
        this.$nextTick(async () => {
            this.loadMoreData();
        });
    }

    loadMoreData(): void {
        this.loadMoreDataAsync()
            .catch((error) => {
                if (error) {
                    console.error('Fail to load next data batch', error);
                }
            })
            .then();
    }

    getColumnCount(elementWidth: number) {
        return Math.floor(elementWidth / 250);
    }

    getGridGap(elementWidth: number, windowHeight: number) {
        if (elementWidth > 720 && windowHeight > 480) {
            return 10;
        } else {
            return 5;
        }
    }

    getWindowMargin(windowHeight: number) {
        return Math.round(windowHeight * 1.5);
    }

    getItemRatioHeight(height: number, width: number, columnWidth: number) {
        const imageRatio = height / width;
        return Math.round(columnWidth * imageRatio);
    }

    async loadMoreDataAsync(): Promise<void> {
        this.computeContainerData();

        const windowTop = this.containerData.scroll.y;
        const windowBottom = windowTop + this.containerData.size.height;
        const bottomTrigger = Math.max(
            0,
            this.containerData.elementOffset + this.containerData.elementSize.height - this.updateTriggerMargin
        );

        if (!this.bottomReached && windowBottom >= bottomTrigger && !this.updateLock) {
            this.updateLock = true;

            debugLog(this.debug, 'Loading next batch');
            const isLastBatch = await this.updateFunction();

            if (isLastBatch) {
                debugLog(this.debug, 'Bottom reached');
                this.bottomReached = true;
            }

            this.updateLock = false;
            await this.loadMoreDataAsync();
        }
        return;
    }

    computeContainerData(): void {
        if (this.ref === null) {
            return;
        }

        const size = this.getSize();
        const scroll = this.getScroll();
        const elementOffset = this.getElementOffset(this.ref);
        const elementSize = this.getElementSize(this.ref);

        this.containerData = { size, scroll, elementOffset, elementSize };
    }

    computeConfigData(containerData: ContainerData, items: Item<P>[]): ConfigData<P> {
        if (containerData === null || items === null) {
            return {
                windowMargin: 0,
                gridGap: 0,
                columnCount: 1,
                entries: [],
            };
        }

        const elementWidth = containerData.elementSize ? containerData.elementSize.width : null;

        const windowMargin = execFunc(this.getWindowMargin(containerData.size.height));

        const gridGap = execFunc(this.getGridGap(elementWidth, containerData.size.height));

        const columnCount = this.getColumnCount(elementWidth);

        const columnWidth = this.getColumnWidth(columnCount, gridGap, elementWidth);

        const entries = items.map((item) => {
            // if width is not set we leave the height untouched
            if (!item.width) {
                return item;
            }

            // we make sure the width takes the full column space and adjust the height based on ratio
            const imageWidth = columnWidth * item.columnSpan + gridGap * (item.columnSpan - 1);
            return {
                ...item,
                height: execFunc(this.getItemRatioHeight(item.height, item.width, imageWidth)),
                width: imageWidth,
            };
        });

        return {
            windowMargin,
            gridGap,
            columnCount,
            entries,
        };
    }

    computeLayoutData(configData: ConfigData<P>): LayoutData<P> {
        if (configData === null) {
            return { cells: [], totalHeight: 0 };
        }

        let currentRowNumber = 1;
        let prevRowsTotalHeight = 0;
        let currentRowMaxHeight = 0;

        let columnShift = 0;

        const cells: Cell<P>[] = configData.entries.map((entry, index) => {
            const { columnCount, gridGap } = configData;

            let columnSpanRecompute = entry.columnSpan;
            let heightRecompute = entry.height;

            // if the column span is 0 or negative we assume it is full width
            if (columnSpanRecompute < 1) {
                columnSpanRecompute = columnCount;
            }

            const distanceToRowStart = (index + columnShift) % columnCount;
            if (entry.newRow && distanceToRowStart !== 0) {
                columnShift += columnCount - distanceToRowStart;
            }

            const shiftedIndex = index + columnShift;
            const columnNumber = (shiftedIndex % columnCount) + 1;
            const rowNumber = Math.floor(shiftedIndex / columnCount) + 1;

            // here we check that an image is not going out of the grid, if yes we resize it
            if (columnNumber + columnSpanRecompute > columnCount + 1) {
                const overlapNumber = columnNumber + columnSpanRecompute - columnCount - 1;
                const overlapRatio = overlapNumber / columnSpanRecompute;
                heightRecompute = heightRecompute * (1 - overlapRatio);

                columnSpanRecompute -= overlapNumber;
            }

            // we need to count the shift created by multiple column objects
            if (columnSpanRecompute > 1) {
                columnShift += columnSpanRecompute - 1;
            }

            if (rowNumber !== currentRowNumber) {
                currentRowNumber = rowNumber;
                prevRowsTotalHeight += currentRowMaxHeight + gridGap;
                currentRowMaxHeight = 0;
            }

            const offset = prevRowsTotalHeight;
            const height = Math.round(heightRecompute);

            currentRowMaxHeight = Math.max(currentRowMaxHeight, height);

            return { ...entry, columnNumber, rowNumber, offset, height, columnSpan: columnSpanRecompute };
        });

        const totalHeight = prevRowsTotalHeight + currentRowMaxHeight;

        return { cells, totalHeight };
    }

    computeRenderData(
        configData: ConfigData<P>,
        containerData: ContainerData,
        layoutData: LayoutData<P>
    ): RenderData<P> {
        if (layoutData === null || configData === null) {
            return { cellsToRender: [], firstRenderedRowNumber: 0, firstRenderedRowOffset: 0 };
        }
        const cellsToRender: Cell<P>[] = [];
        let firstRenderedRowNumber: null | number = null;
        let firstRenderedRowOffset: null | number = null;

        if (containerData.elementOffset !== null) {
            const elementOffset = containerData.elementOffset;

            for (const cell of layoutData.cells) {
                const cellTop = elementOffset + cell.offset;
                const cellBottom = cellTop + cell.height;

                const windowTop = containerData.scroll.y;
                const windowBottom = windowTop + containerData.size.height;

                const renderTop = windowTop - configData.windowMargin;
                const renderBottom = windowBottom + configData.windowMargin;

                if (cellTop > renderBottom) {
                    continue;
                }
                if (cellBottom < renderTop) {
                    continue;
                }

                if (firstRenderedRowNumber === null) {
                    firstRenderedRowNumber = cell.rowNumber;
                }

                if (cell.rowNumber === firstRenderedRowNumber) {
                    firstRenderedRowOffset = firstRenderedRowOffset
                        ? Math.min(firstRenderedRowOffset, cell.offset)
                        : cell.offset;
                }

                cellsToRender.push(cell);
            }
        }

        return { cellsToRender, firstRenderedRowNumber, firstRenderedRowOffset };
    }

    /** Grid utils */

    getColumnWidth(columnCount: number | null, gridGap: number | null, elementWidth: number | null) {
        if (columnCount === null || gridGap === null || elementWidth === null) {
            return null;
        }

        const totalGapSpace = (columnCount - 1) * gridGap;
        const columnWidth = Math.round((elementWidth - totalGapSpace) / columnCount);

        return columnWidth;
    }

    getGridRowStart(cell: Cell<P>, renderData: RenderData<P> | null) {
        if (renderData === null) {
            return undefined;
        }

        const offset = renderData.firstRenderedRowNumber !== null ? renderData.firstRenderedRowNumber - 1 : 0;
        const gridRowStart = cell.rowNumber - offset;

        return `${gridRowStart}`;
    }

    /** For Parent Component */

    resetGrid(): void {
        this.bottomReached = false;
        this.loadMoreData();
    }

    /** Utils */

    isSameElementSize(a: ElementSize, b: ElementSize) {
        return a.width === b.width && a.height === b.height;
    }

    getSize(): ElementSize {
        return {
            width: this.container ? this.container.clientWidth : window.innerWidth,
            height: this.container ? this.container.clientHeight : window.innerHeight,
        };
    }

    getElementSize(element: Element): ElementSize {
        const rect = element.getBoundingClientRect();
        return {
            width: rect.width,
            height: rect.height,
        };
    }

    isSameElementScroll(a: ElementScroll, b: ElementScroll) {
        return a.x === b.x && a.y === b.y;
    }

    getScroll(): ElementScroll {
        return {
            x: this.container ? this.container.scrollLeft : window.scrollX,
            y: this.container ? this.container.scrollTop : window.scrollY,
        };
    }

    getElementOffset(element: Element) {
        const scroll = this.container ? this.container.scrollTop : window.scrollY;
        return scroll + element.getBoundingClientRect().top;
    }
}
</script>


<template>
    <div
        ref="virtualGrid"
        :style="{
            boxSizing: 'border-box',
            height: `${layoutData.totalHeight}px`,
            paddingTop: `${
                renderData !== null && renderData.firstRenderedRowOffset !== null
                    ? renderData.firstRenderedRowOffset + 'px'
                    : '0px'
            }`,
        }"
    >
        <div
            class="grid"
            :style="gridStyle"
        >
            <div
                v-for="item in renderData.cellsToRender"
                :key="item.id"
                :style="{
                    'height': item.height,
                    'grid-column-start': item.columnNumber,
                    'grid-column-end': item.columnNumber + item.columnSpan,
                    'grid-row-start': getGridRowStart(item, renderData),
                }"
            >
                <component :is="item.renderComponent" :item="item" v-on="$listeners" />
            </div>
        </div>
        <component :is="loadingBatch && loader" />
    </div>
</template>

<style scoped>
.grid {
    display: grid;
    align-items: center;
}
</style>
