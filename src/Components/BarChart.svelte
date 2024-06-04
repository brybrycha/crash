<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";

  let accidentData = [];
  let processedData = [];
  let totalAccidentsBySeverity = {};

  function preprocessData(data) {
    const parseDate = d3.timeParse('%d/%m/%Y %I:%M:%S %p');
    const parseTime = d3.timeParse('%H.%M');

    data.forEach(d => {
      if (d.Date && d.Severity) {
        d.date = parseDate(d.Date);
        d.time = parseTime(d.Time);
        d.hour = d.time ? d.time.getHours() : null;
        d.daytime = d.hour >= 6 && d.hour < 18 ? 'Daytime' : 'Nighttime';

        d.severity = d.Severity.toLowerCase().includes('fatal') ? 'FATAL' :
                      d.Severity.toLowerCase().includes('serious') ? 'SERIOUS' :
                      d.Severity.toLowerCase().includes('slight') ? 'SLIGHT' : null;

        if (d.daytime && d.severity) {
          processedData.push(d);
          if (!totalAccidentsBySeverity[d.severity]) {
            totalAccidentsBySeverity[d.severity] = 0;
          }
          totalAccidentsBySeverity[d.severity]++;
        }
      }
    });
  }

  // Function to aggregate data by severity and daytime/nighttime and calculate percentages
  function aggregateData() {
    const severityCounts = d3.groups(processedData, d => d.severity, d => d.daytime).map(d => {
      const daytimeCount = d[1].find(v => v[0] === 'Daytime')?.[1].length || 0;
      const nighttimeCount = d[1].find(v => v[0] === 'Nighttime')?.[1].length || 0;
      const total = daytimeCount + nighttimeCount;
      const daytimePercentage = (daytimeCount / total) * 100;
      const nighttimePercentage = (nighttimeCount / total) * 100;
      return { severity: d[0], daytimePercentage, nighttimePercentage, daytimeCount, nighttimeCount };
    });

    return severityCounts;
  }

  // Function to draw the chart
  function drawChart(data) {
    const margin = { top: 80, right: 10, bottom: 40, left: 150 }; // Adjusted right margin
    const width = 1000 - margin.left - margin.right; // Adjusted width
    const height = 400 - margin.top - margin.bottom; // Adjusted height

    d3.select("#dual-bar-chart").selectAll("*").remove();

    const svg = d3.select("#dual-bar-chart")
      .attr("width", width + margin.left + margin.right + 100) // Added extra space for total counts
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const x = d3.scaleLinear()
      .domain([0, 100]) // Percentage scale
      .range([0, width]);

    const y = d3.scaleBand()
      .domain(data.map(d => d.severity))
      .range([0, height])
      .padding(0.2); // Adjust padding to fit bars inside the chart

    // Tooltip div
    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background", "#fff")
      .style("border", "1px solid #ccc")
      .style("padding", "5px")
      .style("display", "none")
      .style("pointer-events", "none");

    svg.append("g")
      .call(d3.axisLeft(y));

    svg.selectAll(".bar.daytime")
      .data(data)
      .enter()
      .append("rect")
      .attr("class", "bar daytime")
      .attr("x", 0) // Start from 0
      .attr("y", d => y(d.severity))
      .attr("width", d => x(d.daytimePercentage))
      .attr("height", 50)
      .attr("fill", "#ffba49")
      .on("mouseover", function(event, d) {
        tooltip.style("display", "block")
          .html(`Daytime Accidents: ${d.daytimeCount}`);
      })
      .on("mousemove", function(event) {
        tooltip.style("left", event.pageX + 10 + "px")
          .style("top", event.pageY - 20 + "px");
      })
      .on("mouseout", function() {
        tooltip.style("display", "none");
      });

    svg.selectAll(".bar.nighttime")
      .data(data)
      .enter()
      .append("rect")
      .attr("class", "bar nighttime")
      .attr("x", d => x(d.daytimePercentage)) // Start where the daytime bar ends
      .attr("y", d => y(d.severity))
      .attr("width", d => x(d.nighttimePercentage))
      .attr("height", 50)
      .attr("fill", "#20a39e")
      .on("mouseover", function(event, d) {
        tooltip.style("display", "block")
          .html(`Nighttime Accidents: ${d.nighttimeCount}`);
      })
      .on("mousemove", function(event) {
        tooltip.style("left", event.pageX + 10 + "px")
          .style("top", event.pageY - 20 + "px");
      })
      .on("mouseout", function() {
        tooltip.style("display", "none");
      });

    svg.selectAll(".label.daytime")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "label daytime")
      .attr("x", d => x(d.daytimePercentage) / 2)
      .attr("y", d => y(d.severity) + y.bandwidth() / 2)
      .attr("dy", ".35em" + 10)
      .attr("text-anchor", "middle")
      .style("fill", "black")
      .text(d => `${d.daytimePercentage.toFixed(1)}%`);

    svg.selectAll(".label.nighttime")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "label nighttime")
      .attr("x", d => x(d.daytimePercentage) + x(d.nighttimePercentage) / 2)
      .attr("y", d => y(d.severity) + y.bandwidth() / 2)
      .attr("dy", ".35em" + 10)
      .attr("text-anchor", "middle")
      .style("fill", "black")
      .text(d => `${d.nighttimePercentage.toFixed(1)}%`);

    // Add total counts on the right
    svg.selectAll(".total-label")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "total-label")
      .attr("x", width + 10)
      .attr("y", d => y(d.severity) + y.bandwidth() / 2)
      .attr("dy", ".35em" + 10)
      .attr("text-anchor", "start")
      .style("font-size", "13px")
      .style("fill", "black")
      .style("font-weight", "bold")
      .text(d => `Total: ${d.daytimeCount + d.nighttimeCount}`);

    svg.append("text")
      .attr("x", 0)
      .attr("y", -10)
      .attr("text-anchor", "end")
      .text("Daytime")
      .style("fill", "orange");

    svg.append("text")
      .attr("x", width - 50)
      .attr("y", -10)
      .attr("text-anchor", "start")
      .text("Nighttime")
      .style("fill", "blue");

    // Add title
    svg.append("text")
      .attr("x", width / 2)
      .attr("y", -margin.top / 2)
      .attr("text-anchor", "middle")
      .style("font-size", "20px")
      .style("text-decoration", "underline")
      .text("Does the Time of the Day Affect Severity of Accidents?");
  }

  onMount(async () => {
    accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
    preprocessData(accidentData);
    const aggregatedData = aggregateData();
    drawChart(aggregatedData);
  });
</script>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: 50px; /* Adjusted margin */
  }
  .chart-container {
    width: 1200px; /* Adjusted width */
  }
  .tooltip {
    position: absolute;
    background: #fff;
    border: 1px solid #ccc;
    padding: 5px;
    display: none;
    pointer-events: none;
  }
</style>

<div class="container">
  <div class="chart-container">
    <svg id="dual-bar-chart"></svg>
  </div>
</div>
