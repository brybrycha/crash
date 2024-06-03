<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";
  import Clock from "./Clock.svelte";

  let lineSvg;
  let accidentData = [];
  let currentHour = 0;
  let updateLineGraph;

  onMount(async () => {
    accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
    updateLineGraph = drawLineGraph();
  });

  function drawLineGraph() {
    const margin = { top: 30, right: 20, bottom: 30, left: 40 };
    const width = 960 - margin.left - margin.right;
    const height = 500 - margin.top - margin.bottom;

    const svg = d3.select(lineSvg)
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const x = d3.scaleLinear()
      .domain([0, 23])
      .range([0, width]);

    const y = d3.scaleLinear()
      .range([height, 0]);

    const xAxis = d3.axisBottom(x).tickFormat(d => `${d}:00`);
    const yAxis = d3.axisLeft(y);

    svg.append("g")
      .attr("class", "x-axis")
      .attr("transform", `translate(0,${height})`);

    svg.append("g")
      .attr("class", "y-axis");

    const line = d3.line()
      .x(d => x(d.hour))
      .y(d => y(d.count));

    const dot = svg.append("circle")
      .attr("r", 5)
      .attr("fill", "red")
      .style("display", "none");

    const dotText = svg.append("text")
      .attr("dx", 10)
      .attr("dy", -10)
      .style("display", "none");

    function updateLineGraph(hour) {
      const filteredData = accidentData
        .map(d => ({
          date: new Date(d.Date),
          hour: new Date(d.Date).getHours(),
          highway: d.Highway
        }))
        .filter(d => d.hour <= hour);

      const accidentCounts = d3.rollups(filteredData, v => v.length, d => d.hour)
        .map(([hour, count]) => ({ hour, count }))
        .sort((a, b) => a.hour - b.hour);

      y.domain([0, d3.max(accidentCounts, d => d.count)]);

      svg.select(".x-axis").call(xAxis);
      svg.select(".y-axis").call(yAxis);

      const path = svg.selectAll(".line")
        .data([accidentCounts]);

      path.enter().append("path")
        .attr("class", "line")
        .merge(path)
        .attr("d", line)
        .style("fill", "none")
        .style("stroke", "green")
        .style("stroke-width", "2px");

      path.exit().remove();

      if (accidentCounts.length > 0) {
        const latestData = accidentCounts[accidentCounts.length - 1];
        dot.attr("cx", x(latestData.hour))
          .attr("cy", y(latestData.count))
          .style("display", "block");

        dotText.attr("x", x(latestData.hour))
          .attr("y", y(latestData.count))
          .text(latestData.count)
          .style("display", "block");
      } else {
        dot.style("display", "none");
        dotText.style("display", "none");
      }
    }

    updateLineGraph(currentHour);

    return updateLineGraph;
  }

  $: {
    if (accidentData.length > 0 && updateLineGraph) {
      updateLineGraph(currentHour);
    }
  }

  function handleHourChange(hour) {
    currentHour = hour;
    if (updateLineGraph) {
      updateLineGraph(currentHour);
    }
  }
</script>

<style>
  #line-graph-container {
    width: 100%;
    height: 100%;
    position: relative;
  }
  svg {
    display: block;
    margin: auto;
  }
</style>

<section>
  <div class='hero'>
    <h1>Accident with Time</h1>
  </div>
  <Clock {currentHour} onChangeHour={handleHourChange} />
  <div class="section-container">
    <div class="sticky">
      <div id="line-graph-container">
        <svg bind:this={lineSvg}></svg>
      </div>
    </div>
  </div>
</section>
