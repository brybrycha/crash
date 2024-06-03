<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";

  let accidentData = [];
  let processedData = [];
  let totalAccidentsBySeverity = {};

  function preprocessData(data) {
    const parseDate = d3.timeParse('%d/%m/%Y %I:%M:%S %p');

    data.forEach(d => {
      if (d.Date && d.Severity) {
        d.date = parseDate(d.Date);

        d.severity = d.Severity.toLowerCase().includes('fatal') ? 'FATAL' :
                      d.Severity.toLowerCase().includes('serious') ? 'SERIOUS' :
                      d.Severity.toLowerCase().includes('slight') ? 'SLIGHT' : null;

        if (d.severity) {
          processedData.push(d);
          if (!totalAccidentsBySeverity[d.severity]) {
            totalAccidentsBySeverity[d.severity] = 0;
          }
          totalAccidentsBySeverity[d.severity]++;
        }
      }
    });
  }

  function drawPieChart(data) {
    const width = 700;
    const height = 500; // Adjusted height for title and labels
    const radius = Math.min(width, height) / 2 - 40; // Adjust radius for label space

    const svg = d3.select("#pie-chart")
      .attr("width", width)
      .attr("height", height)
      .append("g")
      .attr("transform", `translate(${width / 2}, ${height / 2})`);

    const color = d3.scaleOrdinal()
      .domain(data.map(d => d.severity))
      .range(['#c9cba3', '#ffe1a8', '#e26d5c']);

    const pie = d3.pie()
      .value(d => d.count)
      .sort(null);

    const arc = d3.arc()
      .innerRadius(0)
      .outerRadius(radius);

    const labelArc = d3.arc()
      .innerRadius(radius)
      .outerRadius(radius + 20); // Adjusted for label space

    const arcs = svg.selectAll(".arc")
      .data(pie(data))
      .enter().append("g")
      .attr("class", "arc");

    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background", "#fff")
      .style("border", "1px solid #ccc")
      .style("padding", "5px")
      .style("display", "none")
      .style("pointer-events", "none");

    arcs.append("path")
      .attr("d", arc)
      .attr("fill", d => color(d.data.severity))
      .on("mouseover", function(event, d) {
        const total = d3.sum(data.map(d => d.count));
        const percent = Math.round(1000 * d.data.count / total) / 10;
        tooltip.style("display", "block")
          .html(`${d.data.severity}: ${percent}%`);
      })
      .on("mousemove", function(event) {
        tooltip.style("left", event.pageX + 10 + "px")
          .style("top", event.pageY - 20 + "px");
      })
      .on("mouseout", function() {
        tooltip.style("display", "none");
      });

    arcs.append("polyline")
      .attr("points", d => {
        const pos = labelArc.centroid(d);
        pos[0] = radius * 0.95 * (midAngle(d) < Math.PI ? 1 : -1);
        return [arc.centroid(d), labelArc.centroid(d), pos];
      })
      .style("fill", "none")
      .style("stroke", "black");

    arcs.append("text")
      .attr("transform", d => {
        const pos = labelArc.centroid(d);
        pos[0] = radius * 0.98 * (midAngle(d) < Math.PI ? 1 : -1);
        return `translate(${pos})`;
      })
      .attr("dy", "0.35em")
      .attr("text-anchor", d => midAngle(d) < Math.PI ? "start" : "end")
      .text(d => `${d.data.severity}: ${d.data.count}`);

    // Add title
    svg.append("text")
      .attr("x", 0)
      .attr("y", -(height / 2) + 15)
      .attr("text-anchor", "middle")
      .style("font-size", "16px")
      .style("text-decoration", "underline")
      .text("Distribution of Accident Severities");

    function midAngle(d) {
      return d.startAngle + (d.endAngle - d.startAngle) / 2;
    }
  }

  onMount(async () => {
    accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
    preprocessData(accidentData);
    const aggregatedData = Object.entries(totalAccidentsBySeverity).map(([severity, count]) => ({ severity, count }));
    drawPieChart(aggregatedData);
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
    width: 700px; /* Adjusted width */
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
    <svg id="pie-chart"></svg>
  </div>
</div>
