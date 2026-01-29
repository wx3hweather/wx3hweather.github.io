<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>North Carolina Wind Forecast (NWS)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Leaflet -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <style>
    body { margin: 0; font-family: Arial, sans-serif; }
    #map { height: 85vh; }
    #controls {
      height: 15vh;
      padding: 10px;
      background: #f3f3f3;
      display: flex;
      align-items: center;
      gap: 15px;
      flex-wrap: wrap;
    }
    #timeLabel { font-weight: bold; }
    .wind-arrow {
      transform-origin: center center;
      font-weight: bold;
    }
    #legend {
      position: absolute;
      bottom: 20px;
      right: 20px;
      background: white;
      padding: 10px;
      font-size: 12px;
      box-shadow: 0 0 5px rgba(0,0,0,0.3);
      z-index: 1000;
    }
    .legend-row { display: flex; align-items: center; gap: 6px; }
    .legend-color { width: 16px; height: 16px; }
  </style>
</head>
<body>

<div id="map"></div>
<div id="controls">
  <label for="timeSlider">Forecast Time:</label>
  <input type="range" id="timeSlider" min="0" max="0" value="0" step="1" />
  <span id="timeLabel">Loading…</span>
  <button id="playBtn">▶ Play</button>
  <label><input type="checkbox" id="toggleArrows" checked> Arrows</label>
  <label><input type="checkbox" id="toggleBg" checked> Speed Colors</label>
</div>
<div id="legend"></div>

<script>
// ---------------- MAP SETUP ----------------
const map = L.map('map', {
  preferCanvas: true
}).setView([35.5, -79.0], 7);('map').setView([35.5, -79.0], 7);

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap contributors'
}).addTo(map);

// ---------------- WIND DATA ----------------
// Wind speed color scale (mph)
function windColor(speed) {
  const v = parseInt(speed);
  if (v < 5) return '#a6cee3';
  if (v < 10) return '#1f78b4';
  if (v < 15) return '#33a02c';
  if (v < 20) return '#ff7f00';
  return '#e31a1c';
}

// Sample grid points across NC (you can add more)
// Reduced grid density for faster rendering
const gridPoints = [
  { name: 'Asheville', lat: 35.5951, lon: -82.5515 },
  { name: 'Charlotte', lat: 35.2271, lon: -80.8431 },
  { name: 'Greensboro', lat: 36.0726, lon: -79.7920 },
  { name: 'Raleigh', lat: 35.7796, lon: -78.6382 },
  { name: 'Fayetteville', lat: 35.0527, lon: -78.8784 },
  { name: 'Wilmington', lat: 34.2257, lon: -77.9447 }
];

let forecastPeriods = [];
let windMarkers = [];
let windBackgrounds = [];
let showArrows = true;
let showBackgrounds = true;
let playInterval = null;
let lastUpdate = 0;

// Convert compass direction to degrees
const dirMap = {
  'N': 0, 'NNE': 22.5, 'NE': 45, 'ENE': 67.5,
  'E': 90, 'ESE': 112.5, 'SE': 135, 'SSE': 157.5,
  'S': 180, 'SSW': 202.5, 'SW': 225, 'WSW': 247.5,
  'W': 270, 'WNW': 292.5, 'NW': 315, 'NNW': 337.5
};

// Cache forecasts to avoid repeated network calls
const forecastCache = {};

async function fetchForecast(lat, lon) {
  const key = `${lat},${lon}`;
  if (forecastCache[key]) return forecastCache[key];

  const pointResp = await fetch(`https://api.weather.gov/points/${lat},${lon}`);
  const pointData = await pointResp.json();

  const forecastResp = await fetch(pointData.properties.forecastHourly);
  const forecastData = await forecastResp.json();

  forecastCache[key] = forecastData.properties.periods;
  return forecastCache[key];
}(lat, lon) {
  const pointResp = await fetch(`https://api.weather.gov/points/${lat},${lon}`);
  const pointData = await pointResp.json();

  const forecastResp = await fetch(pointData.properties.forecastHourly);
  const forecastData = await forecastResp.json();

  return forecastData.properties.periods;
}

function clearMarkers() {
  windMarkers.forEach(m => map.removeLayer(m));
  windBackgrounds.forEach(b => map.removeLayer(b));
  windMarkers = [];
  windBackgrounds = [];
}

function addWindMarker(lat, lon, period) {
  const speed = parseInt(period.windSpeed);
  const gust = parseInt(period.windGust || speed);
  const dirText = period.windDirection;
  const deg = dirMap[dirText] ?? 0;

  if (showBackgrounds) {
    const bg = L.circle([lat, lon], {
      radius: 25000 + speed * 1500,
      fillColor: windColor(speed),
      fillOpacity: 0.45,
      stroke: false
    }).addTo(map);
    windBackgrounds.push(bg);
  }

  if (showArrows) {
    const size = 14 + speed;
    const icon = L.divIcon({
      className: 'wind-arrow',
      html: `<div style="transform: rotate(${deg}deg); font-size:${size}px">➤</div>`
    });

    const marker = L.marker([lat, lon], { icon })
      .addTo(map)
      .bindPopup(`
        <strong>${new Date(period.startTime).toLocaleString()}</strong><br>
        Sustained: ${speed} mph<br>
        Gusts: ${gust || 'N/A'} mph<br>
        Direction: ${dirText}
      `);

    windMarkers.push(marker);
  }
}
}
}

function updateMap(timeIndex) {
  clearMarkers();
  const period = forecastPeriods[timeIndex];

  document.getElementById('timeLabel').textContent =
    new Date(period.startTime).toLocaleString();

  // Limit redraw rate for performance
  gridPoints.forEach(pt => {
    addWindMarker(pt.lat, pt.lon, period);
  });
}(timeIndex) {
  clearMarkers();
  const period = forecastPeriods[timeIndex];

  document.getElementById('timeLabel').textContent =
    new Date(period.startTime).toLocaleString();

  gridPoints.forEach(pt => addWindMarker(pt.lat, pt.lon, period));
}

function buildLegend() {
  const legend = document.getElementById('legend');
  legend.innerHTML = `
    <strong>Wind Speed (mph)</strong>
    <div class="legend-row"><div class="legend-color" style="background:#a6cee3"></div><span><5</span></div>
    <div class="legend-row"><div class="legend-color" style="background:#1f78b4"></div><span>5–9</span></div>
    <div class="legend-row"><div class="legend-color" style="background:#33a02c"></div><span>10–14</span></div>
    <div class="legend-row"><div class="legend-color" style="background:#ff7f00"></div><span>15–19</span></div>
    <div class="legend-row"><div class="legend-color" style="background:#e31a1c"></div><span>20+</span></div>
  `;
}

function togglePlay() {
  const slider = document.getElementById('timeSlider');
  if (playInterval) {
    clearInterval(playInterval);
    playInterval = null;
    playBtn.textContent = '▶ Play';
  } else {
    playBtn.textContent = '⏸ Pause';
    playInterval = setInterval(() => {
      const now = Date.now();
      if (now - lastUpdate < 1200) return;
      lastUpdate = now;
      slider.value = (parseInt(slider.value) + 1) % forecastPeriods.length;
      updateMap(slider.value);
    }, 1500);
  }
}() {
  const slider = document.getElementById('timeSlider');
  if (playInterval) {
    clearInterval(playInterval);
    playInterval = null;
    playBtn.textContent = '▶ Play';
  } else {
    playBtn.textContent = '⏸ Pause';
    playInterval = setInterval(() => {
      slider.value = (parseInt(slider.value) + 1) % forecastPeriods.length;
      updateMap(slider.value);
    }, 1500);
  }
}

// ---------------- INIT ----------------
(async function init() {
  // Use Raleigh as reference for timeline
  forecastPeriods = await fetchForecast(35.7796, -78.6382);

  const slider = document.getElementById('timeSlider');
  slider.max = forecastPeriods.length - 1;

  slider.addEventListener('input', e => updateMap(e.target.value));

document.getElementById('toggleArrows').addEventListener('change', e => {
  showArrows = e.target.checked;
  updateMap(slider.value);
});

document.getElementById('toggleBg').addEventListener('change', e => {
  showBackgrounds = e.target.checked;
  updateMap(slider.value);
});

document.getElementById('playBtn').addEventListener('click', togglePlay);

buildLegend();

  updateMap(0);
})();
</script>

</body>
</html>
