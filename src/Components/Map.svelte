<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";

  let map;
  let accidentData = [];
  let accidentLocations = {};
  let isDaytime = true;
  let totalAccidents = 0;

  onMount(async () => {
    mapboxgl.accessToken = "pk.eyJ1IjoiYnJjaGEiLCJhIjoiY2x3bGFxcDRlMW5wODJzb2N0eHFmM2FjeCJ9.Q47OaoGh6RIGs5d54fPiqw";
    map = new mapboxgl.Map({
      container: "map",
      style: getStyleBasedOnTime(isDaytime),
      center: [-0.1426, 51.5413], 
      zoom: 15,
      minZoom: 12,
      maxZoom: 25,
    });

    map.on("load", async () => {
      accidentData = await csv("https://raw.githubusercontent.com/brybrycha/Crash_Camden_UK/main/public/Cleaned_Road_Collision_Vehicles_In_Camden.csv");

      updateAccidentData();
      updateAccidentClusters();
      updateVisibleAccidents(); // Ensure initial update

      map.on('zoomend', () => {
        updateAccidentClusters();
        updateVisibleAccidents();
      });

      map.on('moveend', updateVisibleAccidents);

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

  function getStyleBasedOnTime(isDaytime) {
    return isDaytime
      ? "mapbox://styles/mapbox/light-v11"
      : "mapbox://styles/mapbox/dark-v11";
  }

  function processAccidentData(data) {
    const accidents = {};
    data.forEach(d => {
      const accidentTime = parseFloat(d.Time);
      const isDayAccident = accidentTime >= 6 && accidentTime < 18;
      if ((isDaytime && isDayAccident) || (!isDaytime && !isDayAccident)) {
        const key = `${d.Latitude},${d.Longitude}`;
        if (accidents[key]) {
          accidents[key].count++;
        } else {
          accidents[key] = { count: 1 };
        }
      }
    });
    return accidents;
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
      map.getSource("accident_points").setData(clusteredGeoJson);
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

    updateVisibleAccidents();
  }

  function updateAccidentData() {
    accidentLocations = processAccidentData(accidentData);
    updateAccidentClusters();
    updateVisibleAccidents(); // Ensure accident count is updated
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
          accidents: accidentLocations[key].count
        }
      };
    });

    return {
      type: "FeatureCollection",
      features: features
    };
  }

  function toggleDaytime() {
    isDaytime = true;
    map.setStyle(getStyleBasedOnTime(isDaytime));
    map.once('styledata', () => {
      updateAccidentData();
    });
  }

  function toggleNighttime() {
    isDaytime = false;
    map.setStyle(getStyleBasedOnTime(isDaytime));
    map.once('styledata', () => {
      updateAccidentData();
    });
  }

  function updateVisibleAccidents() {
    const bounds = map.getBounds();
    const visibleFeatures = map.queryRenderedFeatures({
      layers: ['accident_points']
    }).filter(feature => {
      const [lng, lat] = feature.geometry.coordinates;
      return lng >= bounds.getWest() && lng <= bounds.getEast() && lat >= bounds.getSouth() && lat <= bounds.getNorth();
    });

    totalAccidents = visibleFeatures.reduce((sum, feature) => {
      return sum + feature.properties.accidents;
    }, 0);
  }
</script>

<style>
  #map-container {
    width: 100%;
    height: 600px;
    position: relative;
  }
  #map {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
    height: 100%;
  }
  .button-container {
    margin-top: 10px;
    display: flex;
    justify-content: center;
    z-index: 1;
  }
  .button-container button {
    margin: 0 5px;
    padding: 10px 20px;
    font-size: 16px;
  }
  .info-box {
    position: absolute;
    top: 50px;
    left: 10px;
    background: white;
    padding: 10px;
    border-radius: 5px;
    z-index: 1;
  }
</style>

<div id="map-container">
  <div class="info-box">
    <strong>Total Accidents in View:</strong> {totalAccidents}
  </div>
  <div id="map"></div>
</div>

<div class="button-container">
  <button on:click={toggleDaytime}>Daytime</button>
  <button on:click={toggleNighttime}>Nighttime</button>
</div>