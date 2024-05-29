<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";

  let selectedYear = 2015;
  let accidentData = [];
  let processedData = [];

  function preprocessData(data) {
    const parseDate = d3.timeParse('%d/%m/%Y %I:%M:%S %p');
    const parseTime = d3.timeParse('%H.%M');

    data.forEach(d => {
      if (d.Date && d.Severity) {
        d.date = parseDate(d.Date);
        d.year = d.date ? d.date.getFullYear() : null;
        d.time = parseTime(d.Time);
        d.hour = d.time ? d.time.getHours() : null;
        d.daytime = d.hour >= 6 && d.hour < 18 ? 'Daytime' : 'Nighttime';

        d.severity = d.Severity.toLowerCase().includes('fatal') ? 'Severity 1 - FATAL' :
                      d.Severity.toLowerCase().includes('serious') ? 'Severity 2 - SERIOUS' :
                      d.Severity.toLowerCase().includes('slight') ? 'Severity 3 - SLIGHT' : null;

        if (d.year && d.daytime && d.severity) {
          processedData.push(d);
        }
      }
    });
  }

  // Function to filter data by year and severity
  function filterDataByYear(year) {
    const filteredData = processedData.filter(d => d.year == year);

    const severityCounts = d3.groups(filteredData, d => d.severity, d => d.daytime).map(d => {
      const daytime = d[1].find(v => v[0] === 'Daytime')?.[1].length || 0;
      const nighttime = d[1].find(v => v[0] === 'Nighttime')?.[1].length || 0;
      return { severity: d[0], daytime, nighttime };
    });

    return severityCounts;
  }

  // Function to draw the chart
  function drawChart(data) {
    const margin = { top: 40, right: 0, bottom: 40, left: 100 };
    const width = 1500 - margin.left - margin.right;
    const height = 600 - margin.top - margin.bottom;

    d3.select("#dual-bar-chart").selectAll("*").remove();

    const svg = d3.select("#dual-bar-chart")
      .attr("width", 2300)
      .attr("height", 1000)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const maxCount = d3.max(data, d => Math.max(d.daytime, d.nighttime));

    const x = d3.scaleLinear()
      .domain([0, maxCount]) // Only positive values
      .range([0, width / 2]);

    const y = d3.scaleBand()
      .domain(data.map(d => d.severity))
      .range([0, height])
      .padding(0.6); // Adjust padding to reduce bar thickness

    svg.append("g")
      .call(d3.axisLeft(y));

    svg.append("g")
      .attr("transform", `translate(${width / 2}, 0)`)
      .call(d3.axisBottom(x).ticks(5));

    svg.selectAll(".bar.daytime")
      .data(data)
      .enter()
      .append("rect")
      .attr("class", "bar daytime")
      .attr("x", d => width / 2 - x(d.daytime))
      .attr("y", d => y(d.severity))
      .attr("width", d => x(d.daytime))
      .attr("height", y.bandwidth())
      .attr("fill", "orange");

    svg.selectAll(".bar.nighttime")
      .data(data)
      .enter()
      .append("rect")
      .attr("class", "bar nighttime")
      .attr("x", width / 2)
      .attr("y", d => y(d.severity))
      .attr("width", d => x(d.nighttime))
      .attr("height", y.bandwidth())
      .attr("fill", "blue");

    svg.selectAll(".label.daytime")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "label daytime")
      .attr("x", d => width / 2 - x(d.daytime) / 2)
      .attr("y", d => y(d.severity) + y.bandwidth() / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .style("fill", "white")
      .text(d => d.daytime);

    svg.selectAll(".label.nighttime")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "label nighttime")
      .attr("x", d => width / 2 + x(d.nighttime) / 2)
      .attr("y", d => y(d.severity) + y.bandwidth() / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .style("fill", "white")
      .text(d => d.nighttime);

    svg.append("text")
      .attr("x", width / 2 - 10)
      .attr("y", -10)
      .attr("text-anchor", "end")
      .text("Daytime")
      .style("fill", "orange");

    svg.append("text")
      .attr("x", width / 2 + 10)
      .attr("y", -10)
      .attr("text-anchor", "start")
      .text("Nighttime")
      .style("fill", "blue");
  }

  // Function to update the chart when the selected year changes
  function updateChart(year) {
    selectedYear = year;
    const filteredData = filterDataByYear(year);
    drawChart(filteredData);
  }

  onMount(async () => {
    accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
    preprocessData(accidentData);
    updateChart(selectedYear);
  });
</script>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: 150px; /* Increased margin to move the chart lower */
  }
  .chart-container {
    width: 1000px;
    margin-left: -300px;
  }
  .slider-container {
    display: flex;
    align-items: center;
    margin-bottom: 20px;
  }
  .year-label {
    margin-left: 10px;
    font-weight: bold;
  }
</style>

<div class="container">
  <div class="slider-container">
    <label for="year-slider">Select Year: </label>
    <input 
      id="year-slider" 
      type="range" 
      min="2015" 
      max="2019" 
      value={selectedYear} 
      on:input="{e => updateChart(e.target.value)}"
    >
    <span class="year-label">{selectedYear}</span>
  </div>
  <div class="chart-container">
    <svg id="dual-bar-chart"></svg>
  </div>
</div>
