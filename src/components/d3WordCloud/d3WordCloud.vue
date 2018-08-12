<template>
    <div class="d3-word-cloud" :style="{ 'width' : width, 'height' : height }"></div>
</template>

<script>
    import * as d3 from 'd3';
    import { cloneDeep, isNull } from 'lodash';
    import cloud from 'd3-cloud';
    import mixins from '../../mixins';

    Object.assign(d3, { cloud });

    export default {
        name: 'd3-word-cloud',
        data() {
            return {
                wordCloud: null,
                calculating: false
            }
        },
        mixins: [mixins],
        methods: {
            drawWordCloud() {
                const data = cloneDeep(this.data),
                    { left = 0, top = 0, right = 0, bottom = 0 } = this.margin,
                    {
                        axisXLabel = null,

                        axisLabelFontSize = 12,
                        axisLabelFontWeight = 400,
                        axisLabelFontOpacity = 0.5,

                        wordPadding = 5,
                        wordFontFamily = 'Impact'
                    } = this.options,
                    {
                        axisXLabelLaneHeight = isNull(axisXLabel) ? 0 : 30
                    } = this.options,
                    [w, h] = this.getElWidthHeight(),
                    g_w = w - left - right,
                    g_h = h - top - bottom - axisXLabelLaneHeight,
                    self = this;

                if (![g_w, g_h].every(el => el > 0)) return;

                const svg = d3.select(this.$el)
                    .append('svg')
                    .attr('width', w)
                    .attr('height', h);

                const g = svg
                    .append('g');

                const axisXLabelLane = svg
                    .append('g')
                    .attr('transform', `translate(${left}, ${top + g_h})`);

                axisXLabelLane
                    .append('text')
                    .attr('text-anchor', 'middle')
                    .attr('x', g_w / 2)
                    .attr('y', axisXLabelLaneHeight / 2)
                    .attr('dy', '0.32em')
                    .text(axisXLabel)
                    .attr('fill', '#000')
                    .attr('fill-opacity', axisLabelFontOpacity)
                    .attr('font-size', axisLabelFontSize)
                    .attr('font-weight', axisLabelFontWeight);

                this.wordCloud = d3.cloud()
                    .size([g_w, g_h])
                    .words(data)
                    .padding(wordPadding)
                    .rotate(() => ~~(Math.random() * 2) * 90)
                    .font(wordFontFamily)
                    .fontSize(d => d.size)
                    .on('start', function () {
                        this.calculating = true;
                    })
                    .on('end', draw);

                function draw(words) {
                    self.calculating = false;

                    g
                        .attr('transform', `translate(${left + g_w / 2}, ${top + g_h / 2})`)
                        .selectAll('text')
                        .data(words)
                        .enter()
                        .append('text')
                        .attr('text-anchor', 'middle')
                        .style('font-size', d => `${d.size}px`)
                        .style('font-family', wordFontFamily)
                        .attr('transform', d => `translate(${[d.x, d.y]})rotate(${d.rotate})`)
                        .text(d => d.text);
                }
            },
            safeDraw() {
                if (this.calculating) this.wordCloud.stop();

                this.ifExistsSvgThenRemove();
                this.drawWordCloud();
            },
            onResize() {
                this.safeDraw();
            }
        },
        beforeDestroy() {
            this.wordCloud.stop();
        }
    }
</script>

<style>

</style>