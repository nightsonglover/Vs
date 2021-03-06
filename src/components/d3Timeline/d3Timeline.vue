<template>
    <div class="d3-timeline" :style="{ 'width' : width, 'height' : height }"></div>
</template>

<script>
    import * as d3 from 'd3';
    import { isNull, cloneDeep } from 'lodash';
    import uuid from 'uuid/v1';
    import mixins from '../../mixins';
    import { selectPaddingInnerOuterY } from '../../utils/select';
    import { getTimelineGroups } from '../../utils/getTimelineGroups';
    import roundedRect from '../../plugins/roundedRect';
    import zoom from '../../plugins/zoom';
    import cursor from '../../plugins/cursor';
    import { brushX } from '../../plugins/brush';
    import { drawCurrentReferenceX } from '../../plugins/drawCurrentReference';
    import { drawTicksX } from '../../plugins/drawTicks';
    import { drawEntriesMultiLaneX } from '../../plugins/drawEntriesMultiLane';

    export default {
        name: 'd3-timeline',
        data() {
            return {
                scale: null,
                timer: null
            }
        },
        mixins: [mixins],
        methods: {
            drawTimeline() {
                const { dateTimeStart, dateTimeEnd, data, groups } = getTimelineGroups(cloneDeep(this.data)),
                    {
                        intervalCornerRadius = 4,

                        symbolSize = 400,

                        groupLabelFontSize = 12,
                        groupLabelFontWeight = 400,
                        groupLabelFontOpacity = 1,

                        groupLaneWidth = 200,

                        tickSize = 10,
                        tickPadding = 8,

                        axisXLaneHeight = 40,

                        axisFontSize = 12,
                        axisFontWeight = 600,
                        axisFontOpacity = 0.5,

                        axisXLabel = null,

                        axisLabelFontSize = 12,
                        axisLabelFontWeight = 600,
                        axisLabelFontOpacity = 0.5,

                        backgroundColor = 'rgba(255, 255, 255, 0.125)',

                        borderRadius = 0,
                        borderWidth = 2,
                        borderColor = 'rgba(0, 0, 0, .125)',

                        boundingLineWidth = 2,
                        boundingLineColor = 'rgba(0, 0, 0, .125)',

                        currentTimeLineWidth = 4,
                        currentTimeLineColor = 'rgba(255, 56, 96, 1)',

                        liveTimer = true,
                        liveTimerTick = 250
                    } = this.options,
                    {
                        axisXLabelLaneHeight = isNull(axisXLabel) ? 0 : 30,
                    } = this.options,
                    { left = 0, top = 0, right = 0, bottom = 0 } = this.margin,
                    [w, h] = this.getElWidthHeight(),
                    __offset__  = borderWidth,
                    g_w = w - left - right - groupLaneWidth - 2 * __offset__,
                    g_h = h - top - bottom - axisXLaneHeight - axisXLabelLaneHeight - 2 * __offset__,
                    entryClipPathId = uuid(), groupLabelClipPathId = uuid(), ctx = this;

                if (![g_w, g_h].every(el => el > 0) || !groups.length) return;

                const groupHeight = g_h / groups.length,
                    [paddingInner, paddingOuter] = selectPaddingInnerOuterY(groupHeight);

                const svg = d3.select(this.$el)
                    .append('svg')
                    .attr('width', w)
                    .attr('height', h);

                svg.append('defs')
                    .append('clipPath')
                    .attr('id', entryClipPathId)
                    .append('rect')
                    .attr('x', 0)
                    .attr('y', 0)
                    .attr('width', g_w)
                    .attr('height', g_h + axisXLaneHeight);

                svg.append('defs')
                    .append('clipPath')
                    .attr('id', groupLabelClipPathId)
                    .append('rect')
                    .attr('x', left + __offset__)
                    .attr('y', top + __offset__)
                    .attr('width', groupLaneWidth)
                    .attr('height', g_h);

                svg.append('path')
                    .attr('d', roundedRect(left + __offset__ / 2, top + __offset__ / 2, g_w + groupLaneWidth + __offset__, g_h + axisXLaneHeight + __offset__, borderRadius, true, true, true, true))
                    .attr('fill', backgroundColor)
                    .attr('stroke', borderColor)
                    .attr('stroke-width', borderWidth)
                    .attr('pointer-events', 'none');

                svg
                    .selectAll('.group--lane')
                    .data(groups)
                    .enter()
                    .append('g')
                    .attr('class', 'group--lane')
                    .attr('clip-path', `url(#${groupLabelClipPathId})`)
                    .attr('transform', (d, i) => `translate(${left + __offset__}, ${top + __offset__ + i * groupHeight})`)
                    .append('text')
                    .attr('x', groupLaneWidth / 2)
                    .attr('y', groupHeight / 2)
                    .attr('dy', '0.32em')
                    .attr('text-anchor', 'middle')
                    .attr('font-size', groupLabelFontSize)
                    .attr('font-weight', groupLabelFontWeight)
                    .attr('fill-opacity', groupLabelFontOpacity)
                    .text(d => d)
                    .attr('fill', '#000');

                svg.selectAll('.line--x')
                    .data(groups)
                    .enter()
                    .append('g')
                    .attr('transform', `translate(${left + __offset__}, ${top + __offset__})`)
                    .append('line')
                    .attr('class', 'line--x')
                    .attr('stroke', boundingLineColor)
                    .attr('stroke-width', boundingLineWidth)
                    .attr('y1', (d, i) => (i + 1) * groupHeight)
                    .attr('y2', (d, i) => (i + 1) * groupHeight)
                    .attr('x2', g_w + groupLaneWidth);

                const xScale = d3.scaleTime()
                    .domain([dateTimeStart, dateTimeEnd])
                    .range([0, g_w]);
                ctx.scale = xScale;

                const yScale = (i) => d3.scaleBand()
                    .range([groupHeight * (i + 1), groupHeight * i])
                    .paddingInner(paddingInner)
                    .paddingOuter(paddingOuter);

                const xAxis = d3
                    .axisBottom(xScale)
                    .tickSize(tickSize)
                    .tickPadding(tickPadding);

                const axisXLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + groupLaneWidth + __offset__}, ${top + g_h + __offset__})`);

                axisXLane
                    .call(xAxis)
                    .attr('class', 'axis axis--x')
                    .attr('font-size', axisFontSize)
                    .attr('font-weight', axisFontWeight)
                    .attr('fill-opacity', axisFontOpacity)
                    .selectAll('line')
                    .attr('stroke', boundingLineColor)
                    .attr('stroke-width', boundingLineWidth);

                const axisXLabelLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + __offset__},${top + g_h + axisXLaneHeight + 2 * __offset__})`);

                axisXLabelLane
                    .append('text')
                    .attr('x', (g_w + groupLaneWidth) / 2)
                    .attr('y', axisXLabelLaneHeight / 2)
                    .attr('fill', '#000')
                    .attr('dy', '0.32em')
                    .attr('text-anchor', 'middle')
                    .text(axisXLabel)
                    .attr('font-size', axisLabelFontSize)
                    .attr('font-weight', axisLabelFontWeight)
                    .attr('fill-opacity', axisLabelFontOpacity);

                const extent = [
                    [left + __offset__ + groupLaneWidth, top + __offset__],
                    [w - right - __offset__, h - axisXLaneHeight - __offset__ - axisXLabelLaneHeight]
                ];

                svg
                    .call(brushX.bind(ctx), extent, ctx.scale)
                    .call(cursor, axisXLane, [
                        [0, 0],
                        [g_w, axisXLaneHeight]
                    ])
                    .call(zoom, zooming, zoomend);

                const g = svg
                    .append('g')
                    .attr('transform', `translate(${left + __offset__ + groupLaneWidth}, ${top + __offset__})`);

                g.append('line')
                    .attr('class', 'line--y')
                    .attr('y2', g_h)
                    .attr('stroke', boundingLineColor)
                    .attr('stroke-width', boundingLineWidth);

                g.call(main, xScale);

                if (liveTimer)
                    ctx.timer = setInterval(() => {
                        g.call(drawCurrentReferenceX, ctx.scale, g_h, entryClipPathId, currentTimeLineColor, currentTimeLineWidth);
                    }, liveTimerTick);

                function zooming() {
                    const newScale = d3.event
                        .transform.rescaleX(xScale);
                    ctx.scale = newScale;

                    svg
                        .call(brushX.bind(ctx), extent, ctx.scale);

                    axisXLane
                        .call(xAxis.scale(newScale))
                        .selectAll('line')
                        .attr('stroke', boundingLineColor)
                        .attr('stroke-width', boundingLineWidth);

                    g.call(main, newScale);
                }

                function zoomend() {
                    const dateTimeStart = ctx.scale.invert(0),
                        dateTimeEnd = ctx.scale.invert(g_w);

                    ctx.$emit('range-updated', dateTimeStart, dateTimeEnd);
                }

                function main(g, scale) {
                    g
                        .call(drawCurrentReferenceX, scale, g_h, entryClipPathId, currentTimeLineColor, currentTimeLineWidth)
                        .call(drawTicksX, scale, g_h, entryClipPathId, boundingLineColor, boundingLineWidth)
                        .call(drawEntriesMultiLaneX, data, groups, scale, yScale, entryClipPathId, symbolSize, intervalCornerRadius);
                }
            },
            safeDraw() {
                this.ifExistsSvgThenRemove();
                this.drawTimeline();
            },
            onResize() {
                this.safeDraw();
            }
        },
        beforeDestroy() {
            clearInterval(this.timer);
        }
    }
</script>

<style>
    @import url(../../css/index.css);

    .d3-timeline path {
        cursor: pointer;
    }

    .d3-timeline .axis.axis--x .domain {
        display: none;
    }

    .d3-timeline text {
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
        cursor: pointer;
    }

    .d3-timeline .entry--interval--default {
        fill:#896bda;
        stroke: #896bda;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--point--default {
        fill:#896bda;
        stroke: #896bda;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--interval--error {
        fill: #ff3860;;
        stroke: #ff3860;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--point--error {
        fill: #ff3860;;
        stroke: #ff3860;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--interval--success {
        fill: #23d160;;
        stroke: #23d160;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--point--success {
        fill: #23d160;;
        stroke: #23d160;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--interval--info {
        fill: #167df0;;
        stroke: #167df0;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--point--info {
        fill: #167df0;;
        stroke: #167df0;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--interval--warn {
        fill: #ffdd57;;
        stroke: #ffdd57;
        stroke-opacity: 1;
        cursor: pointer;
    }

    .d3-timeline .entry--point--warn {
        fill: #ffdd57;;
        stroke: #ffdd57;
        stroke-opacity: 1;
        cursor: pointer;
    }
</style>
