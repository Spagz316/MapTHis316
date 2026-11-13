# MapTHis316
Personal KML Mapping tools 

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>My KML Map</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<link rel="stylesheet" href="https://unpkg.com/@geoman-io/leaflet-geoman-free@2/dist/leaflet-geoman.css">
<style>
  html, body { margin:0; height:100%; font-family:sans-serif; }
  #map { height:100%; }
  #bar { position:absolute; top:10px; right:10px; z-index:1000; display:flex; gap:6px; flex-wrap:wrap;
         background:#fff; padding:8px; border-radius:8px; box-shadow:0 1px 4px rgba(0,0,0,.3); }
  #bar button, #bar label { font-size:14px; padding:6px 10px; border:1px solid #ccc;
         border-radius:6px; background:#f7f7f7; cursor:pointer; }
</style>
</head>
<body>
<div id="bar">
  <button id="locate">📍 Me</button>
  <button id="export">Export KML</button>
  <label>Import KML<input type="file" id="import" accept=".kml" hidden></label>
  <button id="clear">Clear</button>
</div>
<div id="map"></div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://unpkg.com/@geoman-io/leaflet-geoman-free@2/dist/leaflet-geoman.js"></script>
<script>
const map = L.map('map').setView([37.7749, -122.4194], 12);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap contributors'
}).addTo(map);

const esc = s => String(s).replace(/[<>&'"]/g, c =>
  ({'<':'&lt;','>':'&gt;','&':'&amp;',"'":'&apos;','"':'&quot;'}[c]));

// Everything you draw lives in this group
const drawn = L.featureGroup().addTo(map);
map.pm.addControls({ position:'topleft', drawCircle:false, drawCircleMarker:false,
                     drawText:false, cutPolygon:false });
map.pm.setGlobalOptions({ layerGroup: drawn });

const save = () => localStorage.setItem('myMap', JSON.stringify(drawn.toGeoJSON()));

function setup(layer) {
  const p = layer.feature.properties;
  layer.bindPopup(`<b>${esc(p.name)}</b>${p.desc ? '<br>' + esc(p.desc) : ''}`);
  layer.on('pm:edit pm:dragend', save);
}
function load(geojson) {
  L.geoJSON(geojson).eachLayer(l => { drawn.addLayer(l); setup(l); });
}
const zoomToAll = () => { if (drawn.getLayers().length) map.fitBounds(drawn.getBounds(), { maxZoom:15 }); };

// Restore saved map
const saved = localStorage.getItem('myMap');
if (saved) { load(JSON.parse(saved)); zoomToAll(); }

// New shape drawn → ask for name and notes
map.on('pm:create', e => {
  const name = prompt('Name?') || 'Untitled';
  const desc = prompt('Notes?') || '';
  e.layer.feature = { type:'Feature', properties:{ name, desc } };
  setup(e.layer); save();
});
map.on('pm:remove', save);

// Export to KML (GeoJSON and KML both use lon,lat order)
const coordStr = cs => cs.map(c => `${c[0]},${c[1]},0`).join(' ');
function geomKml(g) {
  if (g.type === 'Point') return `<Point><coordinates>${g.coordinates[0]},${g.coordinates[1]},0</coordinates></Point>`;
  if (g.type === 'LineString') return `<LineString><tessellate>1</tessellate><coordinates>${coordStr(g.coordinates)}</coordinates></LineString>`;
  if (g.type === 'Polygon') return `<Polygon><outerBoundaryIs><LinearRing><coordinates>${coordStr(g.coordinates[0])}</coordinates></LinearRing></outerBoundaryIs></Polygon>`;
  return '';
}
document.getElementById('export').onclick = () => {
  const body = drawn.toGeoJSON().features.map(f => `
  <Placemark><name>${esc(f.properties.name || 'Untitled')}</name><description>${esc(f.properties.desc || '')}</description>${geomKml(f.geometry)}</Placemark>`).join('');
  const kml = `<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://www.opengis.net/kml/2.2"><Document><name>My Map</name>${body}
</Document></kml>`;
  const a = document.createElement('a');
  a.href = URL.createObjectURL(new Blob([kml], { type:'application/vnd.google-earth.kml+xml' }));
  a.download = 'my-map.kml'; a.click();
};

// Import KML: points, lines and areas
document.getElementById('import').onchange = async e => {
  const file = e.target.files[0]; if (!file) return;
  const xml = new DOMParser().parseFromString(await file.text(), 'text/xml');
  const parse = el => el.textContent.trim().split(/\s+/).map(t => t.split(',').slice(0, 2).map(Number));
  const features = [];
  xml.querySelectorAll('Placemark').forEach(pm => {
    const properties = { name: pm.querySelector('name')?.textContent || 'Untitled',
                         desc: pm.querySelector('description')?.textContent || '' };
    const add = geometry => features.push({ type:'Feature', properties:{ ...properties }, geometry });
    pm.querySelectorAll('Point coordinates').forEach(c => add({ type:'Point', coordinates: parse(c)[0] }));
    pm.querySelectorAll('LineString coordinates').forEach(c => add({ type:'LineString', coordinates: parse(c) }));
    pm.querySelectorAll('Polygon outerBoundaryIs coordinates').forEach(c => add({ type:'Polygon', coordinates: [parse(c)] }));
  });
  load({ type:'FeatureCollection', features }); save(); zoomToAll();
  e.target.value = '';
};

// Show my location
document.getElementById('locate').onclick = () => map.locate({ setView:true, maxZoom:16 });
map.on('locationfound', e => L.circleMarker(e.latlng, { radius:8, pmIgnore:true }).addTo(map));
map.on('locationerror', e => alert('Location unavailable: ' + e.message));

// Clear everything
document.getElementById('clear').onclick = () => {
  if (confirm('Delete everything on this map?')) { drawn.clearLayers(); save(); }
};
</script>
</body>
</html>
```
