<template>
    <div class="d3-multi-line" :style="{ 'width' : width, 'height' : height }"></div>
</template>

<script>
    import * as d3 from 'd3';
    import { isNull, isNumber, isUndefined, cloneDeep } from 'lodash';
    import uuid from 'uuid/v1';
    import { brushX } from '../../plugins/brush';
    import mixins from '../../mixins';
    import groupBy from '../../utils/groupBy';
    import { responsiveAxisX } from '../../plugins/responsiveAxis';
    import tickFormat from '../../utils/tickFormat';
    import { showTip, hideTip } from '../../plugins/tooltip';
    import { selectTicksNumY } from '../../utils/select';
    import wrap from '../../plugins/wrap';
    import { firstTickTextAnchorStart, lastTickTextAnchorEnd } from '../../plugins/textAnchor';
    import GetAllKeys from '../../utils/allKeys';
    import isAxisTime from '../../utils/isAxisTime';
    import isAxisNumber from '../../utils/isAxisNumber';
    import emit from '../../utils/emit';
    import axisShow from '../../plugins/axisShow';
    import hash from '../../utils/hash';
    import legendGen from '../../plugins/legendGen';
    import { yRuler } from '../../plugins/ruler';

    export default {
        name: 'd3-multi-line',
        mixins: [mixins],
        methods: {
            drawMultiLine() {
                const _data = cloneDeep(this.data),
                    { left = 0, top = 0, right = 0, bottom = 0 } = this.margin,
                    {
                        strokeWidth = 2,

                        circleRadius = 5,
                        circleTitle = d => `${d.value}`,

                        crossWidth = 2,
                        crossColor = '#fff',

                        axisXLabel = null,
                        axisYLabel = null,

                        tickSize = 10,
                        tickPadding = 8,

                        axisXGroupLabelLaneHeight = 20,

                        axisXGroupLabelFillColorOpacity = 1,
                        axisXGroupLabelBorderColorOpacity = 0.6,

                        axisXGroupLabelGap = 10,

                        axisXGroupLabelFontSize = 9,
                        axisXGroupLabelFontWeight = 400,
                        axisXGroupLabelFontOpacity = 0.5,


                        axisLabelFontSize = 12,
                        axisLabelFontWeight = 400,
                        axisLabelFontOpacity = 0.5,

                        axisXLaneHeight = 60,
                        axisYLaneWidth = 35,

                        axisFontSize = 12,
                        axisFontWeight = 400,
                        axisFontOpacity = 0.5,

                        axisXTimeInterval = null,

                        isAxisPathShow = true,

                        curve = 'curveLinear',

                        axisYTickFormat = '.2s',

                        groupKey = 'group',

                        negative = false,

                        nice = false,

                        yAxisRuler = true
                    } = this.options,
                    {
                        axisXLabelLaneHeight = isNull(axisXLabel) ? 0 : 30,
                        axisYLabelLaneWidth = isNull(axisYLabel) ? 0 : 30,
                    } = this.options,
                    [w, h] = this.getElWidthHeight(),
                    __offsetRight__ = 10,
                    g_w = w - left - right - axisYLabelLaneWidth - axisYLaneWidth - __offsetRight__,
                    g_h = h - top - bottom - axisXLabelLaneHeight - axisXLaneHeight - axisXGroupLabelLaneHeight,
                    clipPathId = uuid(), isAxisXTime = isAxisTime(_data), isAxisXNumber = isAxisNumber(_data),
                    data = groupBy(_data, groupKey), groups = Object.keys(data),
                    axisXTickFormat = value => isAxisXTime ? tickFormat(value, axisXTimeInterval) : value,
                    ticks = selectTicksNumY(g_h);

                if (![g_w, g_h].every(el => el > 0) || !groups.length) return;

                const schemeCategory20c = d3.scaleOrdinal()
                    .domain(groups)
                    .range(d3.schemeCategory20c);

                const yScale = d3.scaleLinear()
                    .domain(negative ? d3.extent(data, d => d.value) : [0, d3.max(_data.map(d => d.value))])
                    .range([g_h, 0]);
                if (nice) yScale.nice();

                const xScale = d3.scalePoint()
                    .domain(GetAllKeys(_data))
                    .range([0, g_w]);

                const lineGen = d3.line()
                    .x(d => xScale(d.key))
                    .y(d => yScale(d.value))
                    .defined(d => !isNull(d) && !isUndefined(d));

                if (!isNull(curve) && !isUndefined(d3[curve])) lineGen.curve(d3[curve]);

                const svg = d3.select(this.$el)
                    .append('svg')
                    .attr('width', w)
                    .attr('height', h);

                const axisXGroupLabelLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + axisYLabelLaneWidth + axisYLaneWidth}, ${top})`);

                const axisYLabelLane = svg
                    .append('g')
                    .attr('transform', `translate(${left}, ${top + axisXGroupLabelLaneHeight})`);

                const axisXLabelLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + axisYLabelLaneWidth + axisYLaneWidth}, ${top + axisXGroupLabelLaneHeight + g_h + axisXLaneHeight})`);

                const axisYLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + axisYLabelLaneWidth}, ${top + axisXGroupLabelLaneHeight})`);

                const axisXLane = svg
                    .append('g')
                    .attr('transform', `translate(${left + axisYLabelLaneWidth + axisYLaneWidth}, ${top + axisXGroupLabelLaneHeight + g_h})`);

                axisXGroupLabelLane
                    .call(
                        legendGen,
                        svg,
                        groups,
                        g_w,
                        schemeCategory20c,
                        axisXGroupLabelLaneHeight,
                        axisXGroupLabelFillColorOpacity,
                        axisXGroupLabelBorderColorOpacity,
                        crossColor,
                        crossWidth,
                        axisXGroupLabelFontSize,
                        axisXGroupLabelFontWeight,
                        axisXGroupLabelFontOpacity,
                        axisXGroupLabelGap
                    );

                axisXLabelLane
                    .append('text')
                    .attr('class', 'label label--x')
                    .attr('text-anchor', 'middle')
                    .attr('x', g_w / 2)
                    .attr('y', axisXLabelLaneHeight / 2)
                    .attr('dy', '0.32em')
                    .text(axisXLabel)
                    .attr('fill', '#000')
                    .attr('fill-opacity', axisLabelFontOpacity)
                    .attr('font-size', axisLabelFontSize)
                    .attr('font-weight', axisLabelFontWeight);

                axisYLabelLane
                    .append('text')
                    .attr('class', 'label label--y')
                    .attr('text-anchor', 'middle')
                    .attr('transform', 'rotate(-90)')
                    .attr('y', axisYLabelLaneWidth / 2)
                    .attr('x', -g_h / 2)
                    .text(axisYLabel)
                    .attr('fill', '#000')
                    .attr('fill-opacity', axisLabelFontOpacity)
                    .attr('font-size', axisLabelFontSize)
                    .attr('font-weight', axisLabelFontWeight);

                const xAxis = d3
                    .axisBottom(xScale)
                    .tickFormat(axisXTickFormat)
                    .tickSize(tickSize)
                    .tickPadding(tickPadding);

                const yAxis = d3
                    .axisLeft(yScale)
                    .ticks(ticks)
                    .tickFormat(d3.format(axisYTickFormat))
                    .tickSize(tickSize)
                    .tickPadding(tickPadding);

                axisXLane
                    .append('g')
                    .attr('class', 'axis axis--x')
                    .call(xAxis)
                    .call(firstTickTextAnchorStart)
                    .call(lastTickTextAnchorEnd)
                    .attr('opacity', axisFontOpacity)
                    .attr('font-size', axisFontSize)
                    .attr('font-weight', axisFontWeight);

                axisYLane
                    .append('g')
                    .attr('transform', `translate(${axisYLaneWidth}, 0)`)
                    .attr('class', 'axis axis--y')
                    .call(yAxis)
                    .call(axisShow, isAxisPathShow, true)
                    .attr('opacity', axisFontOpacity)
                    .attr('font-size', axisFontSize)
                    .attr('font-weight', axisFontWeight);
                if (yAxisRuler) axisYLane.call(yRuler, yScale, g_w, d3.format(axisYTickFormat), ticks, tickSize, tickPadding);

                if (isAxisXTime || isAxisXNumber) {
                    const extent = [
                        [left + axisYLaneWidth + axisYLabelLaneWidth, top + axisXGroupLabelLaneHeight],
                        [w - right - __offsetRight__, g_h + top + axisXGroupLabelLaneHeight]
                    ];

                    axisXLane
                        .call(responsiveAxisX, xAxis, xScale);

                    svg
                        .call(brushX.bind(this), extent, xScale, _data);
                }

                axisXLane
                    .call(axisShow, isAxisPathShow, true)
                    .selectAll('text')
                    .call(wrap, xScale.step());

                svg.append('defs')
                    .append('clipPath')
                    .attr('id', clipPathId)
                    .append('rect')
                    .attr('x', 0)
                    .attr('y', 0)
                    .attr('width', g_w)
                    .attr('height', g_h);

                const g = svg.append('g')
                    .attr('transform', `translate(${left + axisYLabelLaneWidth + axisYLaneWidth}, ${top + axisXGroupLabelLaneHeight})`)
                    .attr('width', g_w)
                    .attr('height', g_h);

                g.selectAll('path')
                    .data(groups)
                    .enter()
                    .append('path')
                    .attr('class', d => hash(d))
                    .attr('clip-path', `url(#${clipPathId})`)
                    .attr('d', d => lineGen(data[d]))
                    .attr('fill', 'none')
                    .attr('pointer-events', 'none')
                    .attr('stroke', d => schemeCategory20c(d))
                    .attr('stroke-width', strokeWidth);

                const circles = g.selectAll('circle')
                    .data(_data)
                    .enter()
                    .append('circle')
                    .attr('class', d => hash(d[groupKey]))
                    .attr('clip-path', `url(#${clipPathId})`)
                    .attr('cx', d => xScale(d.key))
                    .attr('cy', d => yScale(d.value))
                    .attr('r', circleRadius)
                    .attr('fill', d => schemeCategory20c(d.group))
                    .on('mouseover', showTip(circleTitle))
                    .on('mouseout', hideTip);

                if (isAxisXTime && isNumber(axisXTimeInterval)) {
                    circles.on('mousedown', d => {
                        const dateTimeStart = d.key,
                            dateTimeEnd = new Date(d.key.valueOf() + axisXTimeInterval);

                        emit(this, 'range-updated', dateTimeStart, dateTimeEnd);
                    });
                }
            },
            safeDraw() {
                this.ifExistsSvgThenRemove();
                this.drawMultiLine();
            },
            onResize() {
                this.safeDraw();
            }
        }
    }
</script>

<style>
    @import url(../../css/index.css);

    .d3-multi-line circle:hover,
    .d3-multi-line rect:not(.overlay):hover,
    .d3-multi-line text:hover {
        cursor: pointer;
    }

    .d3-multi-line text {
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }
</style>
