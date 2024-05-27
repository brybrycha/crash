<script>
  import Meta from "./Meta.svelte";
  import Title from "./Components/Title.svelte";
  import Intro from "./Components/Intro.svelte";
  import TextAndMathEquations from "./Components/TextAndMathEquations.svelte";
  import PieChart from "./Components/PieChart.svelte";
  import ScrollCenter from "./Components/ScrollCenter.svelte";
  import Conclusion from "./Components/Conclusion.svelte";
  import Resources from "./Components/Resources.svelte";
  import ScrollSide from "./Components/ScrollSide.svelte";

  // Example data for the PieChart component
  const pieChartData = [
    { label: 'A', value: 30 },
    { label: 'B', value: 70 },
    { label: 'C', value: 50 },
    { label: 'D', value: 20 },
    { label: 'E', value: 90 },
  ];
</script>

<Meta />
<Title />
<Intro />
<TextAndMathEquations />
<ScrollSide />
<PieChart data = {pieChartData} />
<ScrollCenter />
<Conclusion />
<Resources />
