<script>
  import Meta from "./Meta.svelte";
  import Title from "./Components/Title.svelte";
  import Intro from "./Components/Intro.svelte";
  import TextAndMathEquations from "./Components/TextAndMathEquations.svelte";
  import PieChart from "./Components/PieChart.svelte";
  import GenderPie from "./Components/GenderPie.svelte";
  import Map from "./Components/Map.svelte";
  import Conclusion from "./Components/Conclusion.svelte";
  import Resources from "./Components/Resources.svelte";
  import ScrollSide from "./Components/ScrollSide.svelte";
  import PieSlider from "./Components/PieSlider.svelte";

  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let accidentData = [];
  let selectedYearRange = [2014, 2015];
  let pieChartData = [];
  let genderData = [];

  async function fetchAccidentData() {
    accidentData = await d3.csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
    cleanData();
    updatePieChartData();
  }

  function cleanData() {
    accidentData.forEach(d => {
      // Clean 'Date' and extract year
      if (d.Date) {
        const [day, month, year] = d.Date.split(' ')[0].split('/');
        d.accidentYear = +year;
      } else {
        d.accidentYear = null;
      }

      // Clean 'Time' column
      d.Time = d.Time ? parseFloat(d.Time.replace(':', '.')) : null;

      // Clean 'Driver Sex' column
      if (d["Driver Sex"]) {
        const sex = d["Driver Sex"].toLowerCase();
        if (sex.includes('male')) {
          d.driverSex = 'Male';
        } else if (sex.includes('female')) {
          d.driverSex = 'Female';
        } else {
          d.driverSex = 'Not Traced';
        }
      } else {
        d.driverSex = 'Not Traced';
      }
    });
  }

  function updatePieChartData() {
    const filteredData = accidentData.filter(d => {
      return d.accidentYear >= selectedYearRange[0] && d.accidentYear <= selectedYearRange[1];
    });

    const daytimeCount = filteredData.filter(d => d.Time !== null && d.Time >= 6 && d.Time < 18).length;
    const nighttimeCount = filteredData.filter(d => d.Time !== null && (d.Time < 6 || d.Time >= 18)).length;

    pieChartData = [
      { label: 'Daytime', value: daytimeCount },
      { label: 'Nighttime', value: nighttimeCount },
    ];

    const maleCount = filteredData.filter(d => d.driverSex === 'Male').length;
    const femaleCount = filteredData.filter(d => d.driverSex === 'Female').length;
    const nonCount = filteredData.filter(d => d.driverSex === 'Not Traced').length;

    genderData = [
      { label: 'Male', value: maleCount },
      { label: 'Female', value: femaleCount },
      { label: 'Non Traced', value: nonCount },
    ];
  }

  onMount(() => {
    fetchAccidentData();
  });

  $: updatePieChartData(); 
</script>

<Meta />
<Title />
<Intro />
<TextAndMathEquations />
<ScrollSide />

<h2 style="text-align: center;">Daytime vs Nighttime Accidents</h2>
<PieSlider {selectedYearRange} onChange={() => updatePieChartData()} />
<PieChart data={pieChartData} />

<h2 style="text-align: center;">Gender Distribution of Accidents</h2>
<GenderPie data = {genderData} />

<Map />
<Conclusion />
<Resources />
