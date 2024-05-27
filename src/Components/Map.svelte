<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import { csv } from "d3-fetch";
  
    let map;
    let accidentData = [];
    let accidentLocations = {};
    let selectedYearRange = [2014, 2015]; 
  
    onMount(async () => {
      mapboxgl.accessToken = "pk.eyJ1IjoiYnJjaGEiLCJhIjoiY2x3bGFxcDRlMW5wODJzb2N0eHFmM2FjeCJ9.Q47OaoGh6RIGs5d54fPiqw";
      map = new mapboxgl.Map({
        container: "map",
        style: getStyleBasedOnTime(),
        center: [-0.1426, 51.5413], 
        zoom: 15,
        minZoom: 12,
        maxZoom: 25,
      });
  
      map.on("load", async () => {
        const geojsonData = await fetch('/export.geojson').then(response => response.json());
  
        accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");
  
        updateAccidentData();
  
        map.addSource("camden_roads", {
          type: "geojson",
          data: geojsonData,
        });
  
        // Add the road lines first
        // have t work on it soonn... it does not work yet
        map.addLayer({
          id: "camden_roads",
          type: "line",
          source: "camden_roads",
          paint: {
            "line-color": "#ADD8E6",  // Light blue color for the roads
            "line-width": 3,
          },
        });
  
        // Initial clustering of accidents
        updateAccidentClusters();
  
        map.on('zoomend', updateAccidentClusters);
  
        // Tooltip
        const tooltip = new mapboxgl.Popup({
          closeButton: false,
          closeOnClick: false
        });
  
        map.on('mouseenter', 'accident_points', (e) => {
          map.getCanvas().style.cursor = 'pointer';
          
          const coordinates = e.features[0].geometry.coordinates.slice();
          const accidents = e.features[0].properties.accidents;
  
          tooltip
            .setLngLat(coordinates)
            .setHTML(`<strong>Accidents:</strong> ${accidents}`)
            .addTo(map);
        });
  
        map.on('mouseleave', 'accident_points', () => {
          map.getCanvas().style.cursor = '';
          tooltip.remove();
        });
      });
    });
  
    function getStyleBasedOnTime() {
      const hour = new Date().getHours();
      return hour >= 6 && hour < 18
        ? "mapbox://styles/mapbox/light-v11"
        : "mapbox://styles/mapbox/dark-v11";
    }
  
    function processAccidentData(data, yearRange) {
      const accidents = {};
      data.forEach(d => {
        const accidentYear = new Date(d.Date).getFullYear();
        if (accidentYear >= yearRange[0] && accidentYear <= yearRange[1]) {
          const key = `${d.Latitude},${d.Longitude}`;
          if (accidents[key]) {
            accidents[key]++;
          } else {
            accidents[key] = 1;
          }
        }
      });
      return accidents;
    }
  
    function createAccidentGeoJson(accidentLocations) {
      const features = Object.keys(accidentLocations).map(key => {
        const [lat, lng] = key.split(',').map(Number);
        return {
          type: "Feature",
          geometry: {
            type: "Point",
            coordinates: [lng, lat]
          },
          properties: {
            accidents: accidentLocations[key]
          }
        };
      });
  
      return {
        type: "FeatureCollection",
        features: features
      };
    }
  
    function updateAccidentClusters() {
      const zoom = map.getZoom();
      const clusterRadius = 0.00002 * Math.pow(2, 20 - zoom); 
  
      const accidentGeoJson = createAccidentGeoJson(accidentLocations);
  
      const clusters = accidentGeoJson.features.reduce((acc, feature) => {
        const { coordinates } = feature.geometry;
        const key = `${Math.round(coordinates[0] / clusterRadius) * clusterRadius},${Math.round(coordinates[1] / clusterRadius) * clusterRadius}`;
        if (!acc[key]) acc[key] = { ...feature, properties: { accidents: 0 } };
        acc[key].properties.accidents += feature.properties.accidents;
        return acc;
      }, {});
  
      const clusteredFeatures = Object.values(clusters);
  
      const clusteredGeoJson = {
        type: "FeatureCollection",
        features: clusteredFeatures
      };
  
      if (map.getSource("accident_points")) {
        // Add a transition to the accident points
        // have to adjust.... 
        map.getSource("accident_points").setData(clusteredGeoJson);
        map.getLayer('accident_points').paint['circle-radius-transition'] = { duration: 1000 };
        map.getLayer('accident_points').paint['circle-color-transition'] = { duration: 1000 };
      } else {
        map.addSource("accident_points", {
          type: "geojson",
          data: clusteredGeoJson,
        });
  
        map.addLayer({
          id: "accident_points",
          type: "circle",
          source: "accident_points",
          paint: {
            "circle-color": [
              "interpolate",
              ["linear"],
              ["get", "accidents"],
              1, "#FFC0CB",  // Bright pink 
              20, "#8B0000"  // Dark red 
            ],
            "circle-radius": [
              "interpolate",
              ["linear"],
              ["get", "accidents"],
              1, 5,
              20, 15
            ],
            "circle-radius-transition": { duration: 1000 },
            "circle-color-transition": { duration: 1000 },
          }
        });
      }
    }
  
    function updateAccidentData() {
      accidentLocations = processAccidentData(accidentData, selectedYearRange);
      updateAccidentClusters();
    }
  
  
    function getBiYearLabel(yearRange) {
      return `${yearRange[0]}-${yearRange[1]}`;
    }
  </script>
  
  <style>
    #map-container {
      width: 100%; /* Adjust as needed */
      height: 600px; /* Adjust as needed */
      position: relative;
    }
    #map {
      position: absolute;
      top: 0;
      bottom: 0;
      width: 100%;
      height: 100%;
    }
    .year-selector {
      position: absolute;
      top: 10px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 1;
      background: white;
      padding: 5px;
      border-radius: 5px;
      display: flex;
      align-items: center;
    }
    .year-selector label {
      margin-left: 10px;
    }
  </style>
  
  <div id="map-container">
    <div class="year-selector">
      <input
        type="range"
        min="2014"
        max="2018"
        step="1"
        value={selectedYearRange[0]}
        on:input={(e) => {
          const startYear = +e.target.value;
          selectedYearRange = [startYear, startYear + 1];
          updateAccidentData();
        }}
      />
      <label>{getBiYearLabel(selectedYearRange)}</label>
    </div>
    <div id="map"></div>
  </div>
