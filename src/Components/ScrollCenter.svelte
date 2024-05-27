<script>
  import { onMount } from "svelte";

  onMount(() => {
      mapboxgl.accessToken = "pk.eyJ1IjoiYnJjaGEiLCJhIjoiY2x3bGFxcDRlMW5wODJzb2N0eHFmM2FjeCJ9.Q47OaoGh6RIGs5d54fPiqw";
      const map = new mapboxgl.Map({
          container: "map",
          style: getStyleBasedOnTime(),
          center: [-0.1426, 51.5413], // Longitude and latitude for Camden
          zoom: 15, // Starting zoom level
          minZoom: 12,
          maxZoom: 25,
      });

      map.on("load", () => {
          fetch('/export.geojson')
              .then(response => response.json())
              .then(data => {
                  map.addSource("camden_roads", {
                      type: "geojson",
                      data: data,
                  });

                  map.addLayer({
                      id: "camden_roads",
                      type: "line",
                      source: "camden_roads",
                      paint: {
                          "line-color": "#BEE5B0",
                          "line-width": 3,
                      },
                  });
              })
              .catch(error => console.error('Error loading GeoJSON data:', error));
      });

      function getStyleBasedOnTime() {
          const hour = new Date().getHours();
          return hour >= 6 && hour < 18 
              ? "mapbox://styles/mapbox/light-v11" 
              : "mapbox://styles/mapbox/dark-v11";
      }
  });
</script>

<style>
  #map-container {
      width: 100%; /* Adjust as needed */
      height: 400px; /* Adjust as needed */
      position: relative;
  }
  #map {
      position: absolute;
      top: 0;
      bottom: 0;
      width: 100%;
      height: 100%;
  }
</style>

<div id="map-container">
  <div id="map"></div>
</div>
