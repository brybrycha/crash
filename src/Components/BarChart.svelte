<script>
    export let selectedYearRange;
    export let onChange;
  
    function handleSliderChange(event) {
      const startYear = +event.target.value;
      selectedYearRange = [startYear, startYear + 1];
      onChange(selectedYearRange);
    }
  </script>
  
  <div class="year-selector">
    <input
      id="yearRangeInput"
      type="range"
      min="2014"
      max="2018"
      step="1"
      value={selectedYearRange[0]}
      on:input={handleSliderChange}
    />
    <label for="yearRangeInput">{`${selectedYearRange[0]}-${selectedYearRange[1]}`}</label>
  </div>
  
  <style>
    .year-selector {
      text-align: center;
      margin-bottom: 20px;
    }
  </style>
  