<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  export let data = [];

  let svgRef;
  let tooltip;
  let svg;

  function drawChart() {
    const width = 450;
    const height = 450;
    const margin = 40;
    const radius = Math.min(width, height) / 2 - margin;

    if (svg) {
      svg.selectAll("*").remove();

      const color = d3.scaleOrdinal()
        .domain(data.map(d => d.label))
        .range(['#1f77b4', '#ff7f0e', '#808080']); // Blue for Male, Orange for Female, gray for non-traced

      const pie = d3.pie()
        .sort(null) 
        .value(d => d.value);

      const data_ready = pie(data);

      const arc = d3.arc()
        .innerRadius(0)
        .outerRadius(radius);

      svg.selectAll('path')
        .data(data_ready)
        .enter()
        .append('path')
        .attr('d', arc)
        .attr('fill', d => color(d.data.label))
        .attr('stroke', 'black')
        .style('stroke-width', '2px')
        .style('opacity', 0.7)
        .on('mouseover', (event, d) => {
          const proportion = (d.data.value / d3.sum(data, d => d.value) * 100).toFixed(2);
          d3.select(tooltip)
            .style('opacity', 1)
            .html(`<strong>${d.data.label}</strong><br/># of Incidents: ${d.data.value}<br/>Proportion: ${proportion}%`);
        })
        .on('mousemove', (event) => {
          d3.select(tooltip)
            .style('left', `${event.pageX + 10}px`)
            .style('top', `${event.pageY - 25}px`);
        })
        .on('mouseout', () => {
          d3.select(tooltip)
            .style('opacity', 0);
        });

      svg.selectAll('text')
        .data(data_ready)
        .enter()
        .append('text')
        .text(d => d.data.label)
        .attr('transform', d => `translate(${arc.centroid(d)})`)
        .style('text-anchor', 'middle')
        .style('font-size', 15)
        .style('fill', 'black');
    }
  }

  onMount(() => {
    const width = 450;
    const height = 450;

    svg = d3.select(svgRef)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${width / 2},${height / 2})`);

    drawChart();
  });

  $: {
    if (svg) {
      drawChart();
    }
  }
</script>

<div id="tooltip" bind:this={tooltip}></div>
<svg bind:this={svgRef}></svg>

<style>
  svg {
    display: block;
    margin: auto;
  }
  #tooltip {
    position: absolute;
    text-align: center;
    width: auto;
    height: auto;
    padding: 10px;
    font: 12px sans-serif;
    background: lightsteelblue;
    border: 0px;
    border-radius: 8px;
    pointer-events: none;
    opacity: 0;
  }
</style>
