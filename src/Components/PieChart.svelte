<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { csv } from "d3-fetch";

  export let data = [];

  let svgRef;
  let tooltip;

  function drawChart() {
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
      .range(['#FFD700', '#00008B']); // Yellow for Daytime, Dark Blue for Nighttime

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
      .style('opacity', 0.7);
  }

  onMount(() => {
    drawChart();
  });

  $: if (data.length) drawChart();
</script>

<svg bind:this={svgRef}></svg>
