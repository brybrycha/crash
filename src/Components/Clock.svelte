<script>
    import { onMount } from "svelte";
  
    export let currentHour = 0;
    export let onChangeHour;
  
    function handleWheel(event) {
      if (event.deltaY > 0) {
        currentHour = (currentHour + 1) % 24;
      } else {
        currentHour = (currentHour - 1 + 24) % 24;
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
    }
  
    .clock {
      font-size: 2em;
      font-family: Arial, sans-serif;
    }
  </style>
  
  <div class="clock-container">
    <div class="clock">
      {currentHour}:00
    </div>
  </div>
  