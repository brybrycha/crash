<script>
  import { onMount } from "svelte";

  export let currentHour = 0;
  export let onChangeHour;

  let scrollDelta = 0;
  const scrollThreshold = 18; // Adjust this value to control sensitivity

  function handleWheel(event) {
    scrollDelta += event.deltaY;

    if (scrollDelta > scrollThreshold) {
      currentHour = (currentHour + 1) % 24;
      scrollDelta = 0;
    } else if (scrollDelta < -scrollThreshold) {
      currentHour = (currentHour - 1 + 24) % 24;
      scrollDelta = 0;
    }

    if (onChangeHour) {
      onChangeHour(currentHour);
    }
  }

  onMount(() => {
    window.addEventListener('wheel', handleWheel);
    return () => {
      window.removeEventListener('wheel', handleWheel);
    };
  });
</script>

<style>
  .clock-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100px;
    background-color: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 5px;
    margin-bottom: 20px;
    transition: background-color 0.3s ease, border-color 0.3s ease;
  }

  .clock {
    font-size: 2em;
    font-family: Arial, sans-serif;
    transition: color 0.3s ease;
  }

  .dot {
    transition: transform 0.3s ease;
  }

  .line {
    transition: width 0.3s ease, height 0.3s ease;
  }
</style>

<div class="clock-container">
  <div class="clock">
    {currentHour}:00
  </div>
</div>