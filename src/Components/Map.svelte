<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { csv } from "d3-fetch";

  let map;
  let accidentData = [];
  let accidentLocations = {};
  let stationData = [];
  let stationMarkers;
  let isDaytime = true;
  let totalAccidents = 0;
  const stationsFile = "https://raw.githubusercontent.com/dsc-courses/dsc106-wi24/gh-pages/resources/data/lab6_station_info.json";

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
      const geojsonData = await fetch('/export.geojson').then(response => response.json());

      map.addSource("camden_roads", {
        type: "geojson",
        data: geojsonData,
      });

      // Add the road lines first
      map.addLayer({
        id: "camden_roads",
        type: "line",
        source: "camden_roads",
        paint: {
          "line-color": "#ADD8E6",  // Light blue color for the roads
          "line-width": 2,
        },
      });

      accidentData = await csv("https://raw.githubusercontent.com/brybrycha/crash/main/public/Road_Collision_Vehicles_In_Camden.csv");

      await fetch(stationsFile)
        .then((response) => response.json())
        .then((d) => stationData = d.data.stations);

      updateAccidentData();

      // Initial clustering of accidents
      updateAccidentClusters();

      map.on('zoomend', () => {
        updateAccidentClusters();
        updateVisibleAccidents();
      });

      map.on('moveend', updateVisibleAccidents);

      // Create SVG container for station markers
      const markerContainer = d3.select(map.getCanvasContainer())
        .append("svg")
        .attr("width", "100%")
        .attr("height", "100%")
        .style("position", "absolute")
        .style("z-index", 2); // Ensure SVG is on top

      createStationMarkers(stationData);

      map.on('viewreset', positionStationMarkers);
      map.on('move', positionStationMarkers);
      map.on('moveend', positionStationMarkers);

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

      updateVisibleAccidents();
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
      const accidentTime = new Date(d.Date).getHours() + new Date(d.Date).getMinutes() / 60;
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
    updateVisibleAccidents(); // Ensure the initial count is correct
  }

  function createStationMarkers(stationData) {
    stationMarkers = d3.select("svg")
      .selectAll("circle")
      .data(stationData)
      .enter()
      .append("circle")
      .attr("r", 5)
      .style("fill", "#808080")
      .attr("stroke", "#808080")
      .attr("stroke-width", 1)
      .attr("fill-opacity", 0.4)
      .attr("name", function (d) {
        return d["name"];
      });

    positionStationMarkers();
  }

  function positionStationMarkers() {
    stationMarkers
      .attr("cx", function (d) {
        return project(d).x;
      })
      .attr("cy", function (d) {
        return project(d).y;
      });
  }

  function project(d) {
    return map.project(new mapboxgl.LngLat(+d.lon, +d.lat));
  }

  function toggleDaytime() {
    isDaytime = true;
    map.setStyle(getStyleBasedOnTime(isDaytime));
    map.once('styledata', () => {
      updateAccidentData();
      d3.selectAll("circle")
        .transition()
        .duration(1000)
        .attr("r", d => scaleRadiusTrafficVolume(d));
    });
  }

  function toggleNighttime() {
    isDaytime = false;
    map.setStyle(getStyleBasedOnTime(isDaytime));
    map.once('styledata', () => {
      updateAccidentData();
      d3.selectAll("circle")
        .transition()
        .duration(1000)
        .attr("r", d => scaleRadiusTrafficVolume(d));
    });
  }

  function scaleRadiusTrafficVolume(traffic) {
    const scaleRadius = d3.scaleSqrt()
      .domain([0, 1, 20])
      .range([0, 5, 15]);
    return scaleRadius(traffic);
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