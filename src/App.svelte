<script>
  import Meta from "./Meta.svelte";
  import Title from "./Components/Title.svelte";
  import Intro from "./Components/Intro.svelte";
  import LineWrite from "./Components/LineWrite.svelte";
  import LineExp from "./Components/LineExp.svelte";
  import Map from "./Components/Map.svelte";
  import MapDayNight from "./Components/MapDayNight.svelte"
  import Sev1 from "./Components/Sev1.svelte";
  import Sev1A from "./Components/Sev1A.svelte";
  import PieChart from "./Components/PieChart.svelte";
  import Sev2 from "./Components/Sev2.svelte";
  import Sev2A from "./Components/Sev 2A.svelte";
  import BarChart from "./Components/BarChart.svelte";

  import { onMount } from 'svelte';
  import * as d3 from 'd3';


  let accidentData = [];
  let selectedYearRange = [2014, 2015];
  let pieChartData = [];
  let genderData = [];

  async function fetchAccidentData() {
    accidentData = await d3.csv("https://raw.githubusercontent.com/brybrycha/crashdata/main/Road_Collision_Vehicles_In_Camden.csv");
    cleanData();
    updatePieChartData();
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

<style>
  .container {
    width: 80%;
    margin: 0 auto;
  }
 </style> 


<Meta />
<Title />
<Intro />
<LineWrite />

<MapDayNight /> 

<LineExp />
<!-- <Intro /> -->

<div class = "container">
  <Map />
</div>

<Sev1 />
<PieChart />
<Sev1A />

<Sev2 />
<BarChart/>
<Sev2A />


<!-- <Conclusion /> -->

<!-- <Resources /> -->