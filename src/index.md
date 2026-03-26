```js
display(html`<h1 style="color: #fff; font-size: 28px; margin-bottom: 10px;">
  Emisiones Tóxicas en Puerto Rico 1987-2023 bla bla
</h1>`);
```
```js
display(html`<h2 style="color: #fff; font-size: 11px; margin-bottom: 30px;">
    Fuente:"https://www.epa.gov/toxics-release-inventory-tri-program/tri-basic-data-files-calendar-years-1987-present" 
    </h2>`);

```

```js 
import * as L from "npm:leaflet";
```

```js
import * as aq from "npm:arquero";
```

```js
async function readData(year)
{
  let csvURL = 'https://cdat.uprh.edu/~eramos/data/datos_EPA/${year}_pr.csv';
  return aq.fromCSV(await fetch(csvURL).ten(res => res.text))
}
```

```js
display(html`<style>
  /* Forzar el fondo oscuro en todo el body */
  body, html { 
    background-color: #121212 !important; 
    color: #fff8f8 !important; 
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    margin: 0;
    padding: 0;
  }

  /* Contenedor principal del dashboard */
  .dashboard-container { 
    background-color: #181818; 
    padding: 25px; 
    border-radius: 12px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.5);
  }

  /* Ajuste para los inputs de Observable (Sliders y texto) */
  .observablehq--input {
    color: black !important;
  }
  
  label {
    color: #fdfbfb !important;
    font-weight: bold;
  }
</style>`);
```
```js
display(html`<style>
  /* Tu CSS existente... */
  
  /* Control de zoom de Leaflet - fondo negro y letras blancas */
  .leaflet-control-zoom {
    border: 2px solid #333 !important;
    border-radius: 5px !important;
    background-color: #000 !important;
    box-shadow: 0 2px 8px rgba(0,0,0,0.8) !important;
  }
  
  .leaflet-control-zoom a {
    background-color: #000 !important;
    color: #fff !important;
    border: none !important;
    line-height: 24px !important;
    font-weight: bold !important;
  }
  
  .leaflet-control-zoom a:hover {
    background-color: #333 !important;
    color: #fff !important;
  }
</style>`);

```

```js
const dataCruda = await FileAttachment("data/2024_pr (3).csv")
    .text()
    .then(aq.fromCSV)
```

```js
function selectData(data) {
  return data
    .select(
      "1. YEAR",
      "4. FACILITY NAME",
      "6. CITY",
      "12. LATITUDE",
      "13. LONGITUDE",
      "23. INDUSTRY SECTOR",
      "37. CHEMICAL"
    )
    .rename({
      "1. YEAR": "YEAR",
      "4. FACILITY NAME": "FACILITY",
      "6. CITY": "CITY",
      "12. LATITUDE": "LATITUDE",
      "13. LONGITUDE": "LONGITUDE",
      "23. INDUSTRY SECTOR": "INDUSTRY",
      "37. CHEMICAL": "CHEMICAL"
    });
}
```

```js
function plotHistogram(data, variable, cantidad) {
  const dataAgrupada = data
    .groupby(variable)
    .count()
    .orderby(aq.desc("count"))
    .reify()
    .slice(0, cantidad);

  const maxCount = Math.max(...dataAgrupada.objects().map((d) => d.count));

  return Plot.plot({
    height: 400,
    marginLeft: 100,
    marginRight: 10,
    width: 700,
    style: {
      padding: 20,
      backgroundColor: "#171615"
    },
    x: {
      grid: true,
      label: "FRECUENCIAS",
      labelAnchor: "center",
      labelArrow: "none",
      domain: [0, maxCount * 1.2]
    },
    y: {
      grid: true,
      label: "MUNICIPIO"
    },

    marks: [
      Plot.frame({ stroke: "black", strokeWidth: -2 }),
      Plot.barX(dataAgrupada.objects(), {
        x: "count",
        y: variable,
        fill: variable,
        sort: { y: "-x" }
      })
    ]
  });
}
```

```js
function plotHistogram1(data, variable, cantidad) {
  const dataAgrupada = data
    .groupby(variable)
    .count()
    .orderby(aq.desc("count"))
    .reify()
    .slice(0, cantidad);

  const maxCount = Math.max(...dataAgrupada.objects().map((d) => d.count));

  return Plot.plot({
    height: 400,
    marginLeft: 170,
    marginRight: 10,
    width: 700,
    style: {
      padding: 20,
      backgroundColor: "#171514"
    },
    x: {
      grid: true,
      label: "FRECUENCIAS",
      labelAnchor: "center",
      labelArrow: "none",
      domain: [0, maxCount * 1.2]
    },
    y: {
      grid: true,
      label: "SECTOR INDUSTRIAL"
    },

    marks: [
      Plot.frame({ stroke: "black", strokeWidth: -2 }),
      Plot.barX(dataAgrupada.objects(), {
        x: "count",
        y: variable,
        fill: variable,
        sort: { y: "-x" }
      })
    ]
  });
}
```

```js
const data = selectData(dataCruda)
```

```js
const cantidad = 20;
```
```js
const añoSeleccionado = view(Inputs.range([1987, 2023], {value: 2023, step: 1, label: "Año"}));
```

```js
const zoomMapa = view(Inputs.range([7, 18], { value: 9, step: 0.1, label: "Zoom" }));
```

```js 
const Leyenda = view(Inputs.toggle({label: "Leyenda", value: true}));
```


```js
    const colorMap = {
      Chemicals: "red",
      "Petroleum Bulk Terminals": "blue",
      "Electric Utilities": "green",
      "Electrical Equipment": "yellow",
      Food: "purple",
      "Miscellaneous Manufacturing": "orange",
      "Fabricated Metals": "pink",
      Petroleum: "brown",
      "Chemical Wholesalers": "gray",
      Beverages: "cyan",
      Machinery: "magenta",
      "Wood Products": "lime",
      "Primary Metals": "indigo",
      "Transportation Equipment": "teal",
      Paper: "violet",
      Other: "turquoise",
      "Hazardous Waste": "maroon",
      Tobacco: "olive",
      "Plastics and Rubber": "silver",
      "Computers and Electronic Products": "gold"
    };
```
```js
function renderMapWithLegend(z, mostrarLeyenda) {
  const container = html`<div style="height: 500px; width: 100%; border-radius: 10px;"></div>`;
  
  Promise.resolve().then(() => {
    const map = L.map(container).setView([18.2208, -66.5901], z);

    const token = 'pk.eyJ1IjoibWVjb2JpIiwiYSI6IjU4YzVlOGQ2YjEzYjE3NTcxOTExZTI2OWY3Y2Y1ZGYxIn0.LUg7xQhGH2uf3zA57szCyw';
    const linkMapa = 'https://api.mapbox.com/styles/v1/mapbox/satellite-streets-v12/tiles/512/{z}/{x}/{y}@2x?access_token=';
    
    L.tileLayer(linkMapa + token, {
      attribution: '© Mapbox',
      tileSize: 512,
      zoomOffset: -1
    }).addTo(map);

    if (mostrarLeyenda) {
      const legend = L.control({ position: "bottomright" });

      legend.onAdd = function() {
        const div = L.DomUtil.create("div", "info legend");
        div.style.backgroundColor = "rgba(10, 6, 6, 0.85)"; 
        div.style.color = "white";
        div.style.padding = "10px";
        div.style.borderRadius = "5px";
        div.style.maxHeight = "800px";
        div.style.border = "1px solid #fdfcfc";
        div.style.fontSize = "11px";

        let labels = ['<h4 style="margin: 0 0 8px 0; font-size: 13px; color: white;">Sector Industrial</h4>'];
        
        for (const [sector, color] of Object.entries(colorMap)) {
          labels.push(
            `<div style="display: flex; align-items: center; margin-bottom: 4px;">
              <i style="background:${color}; width: 12px; height: 12px; border-radius: 50%; display: inline-block; margin-right: 8px; border: 1px solid #fff;"></i> 
              ${sector}
            </div>`
          );
        }

        div.innerHTML = labels.join("");
        return div;
      };

      legend.addTo(map);
    }
    

    const lat = data.array("LATITUDE");
    const lon = data.array("LONGITUDE");
    const ind = data.array("INDUSTRY");
    const fac = data.array("FACILITY");

    lat.forEach((l, i) => {
      if (!l || !lon[i]) return;
      const sector = ind[i] || "Other";
      
      L.circleMarker([l, lon[i]], {
        radius: 7,
        fillColor: colorMap[sector] || "turquoise",
        color: "#fff",
        weight: 1,
        fillOpacity: 0.9
      })
      .addTo(map)
      .bindPopup(`<b>${fac[i]}</b><br>${sector}`);
    });
  });

  return container;
}

display(html`<div style="display: grid; grid-template-columns: 2fr 1fr; gap: 10px; width: 150%; height: 700px; margin: 0 auto;">
  <div style="background-color: #1e1e1e; border-radius: 15px; border: 2px solid #0a0909; padding: 25px; box-shadow: 0 8px 65px rgba(0,0,0,0.6); overflow: hidden; display: flex; align-items: center; justify-content: center;">
    ${renderMapWithLegend(zoomMapa, Leyenda)}
  </div>

  <div style="display: flex; flex-direction:  column; gap: 10px; height: 100%; width: 100%;">
    <div style="background-color: #1e1e1e; border-radius: 15px; border: 2px solid #070606; padding: 25px; box-shadow: 0 8px 25px rgba(0,0,0,0.6); overflow: hidden; flex: 1; display: flex; align-items: center; justify-content: center;">
      ${plotHistogram(data, "CITY", cantidad)}
    </div>
    
    <div style="background-color: #1e1e1e; border-radius: 15px; border: 2px solid #111010; padding: 25px; box-shadow: 0 8px 25px rgba(0,0,0,0.6); overflow: hidden; flex: 1; display: flex; align-items: center; justify-content: center;">
      ${plotHistogram1(data, "INDUSTRY", cantidad)}
    </div>
  </div>
</div>`);

```