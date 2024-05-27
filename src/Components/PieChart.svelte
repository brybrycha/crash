<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  export let data = [];

  let svgRef;

  onMount(() => {
    const width = 450;
    const height = 450;
    const margin = 40;
    const radius = Math.min(width, height) / 2 - margin;

    const svg = d3.select(svgRef)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${width / 2},${height / 2})`);

    const color = d3.scaleOrdinal()
      .domain(data.map(d => d.label))
      .range(d3.schemeSet2);

    const pie = d3.pie()
      .value(d => d.value);

    const data_ready = pie(data);

    svg.selectAll('whatever')
      .data(data_ready)
      .enter()
      .append('path')
      .attr('d', d3.arc()
        .innerRadius(0)
        .outerRadius(radius))
      .attr('fill', d => color(d.data.label))
      .attr('stroke', 'black')
      .style('stroke-width', '2px')
      .style('opacity', 0.7);
  });
</script>

<svg bind:this={svgRef}></svg>

<style>
  svg {
    display: block;
    margin: auto;
  }
</style>
