---
theme: dashboard
title: Global Housing Dashboard
toc: false
---
<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Serif+Display:ital@0;1&family=Lato:ital,wght@0,100;0,300;0,400;0,700;0,900;1,100;1,300;1,400;1,700;1,900&display=swap');
</style>

<div class="hero">
  <h1>Where Can I Realistically Afford To Live</h1>
  <h3>Explore global housing affordability with interactive visualizations. Compare countries by house prices, rent, mortgage rates, and see where your budget can realistically get you a home.</h3>
</div>

<!-- Load and transform the data -->
```js
const housingRaw = await FileAttachment("data/housingextended.csv").csv({typed: true});
const unitRaw = await FileAttachment("data/ThunderbitGlobalHousing.csv").csv({typed: true});

const housingData = housingRaw.map((d) => ({
  country: d.Country,
  year: +d.Year,
  house_price_index: +d["House Price Index"],
  rent_index: +d["Rent Index"],
  affordability_ratio: +d["Affordability Ratio"],
  mortgage_rate: +d["Mortgage Rate (%)"]
}));

const countryDomain = Array.from(new Set(housingData.map((d) => d.country))).sort(d3.ascending);
const countryColours = [
  "#b46b68",
  "#b47868",
  "#d49e5f",
  "#d7a76e",
  "#ddc56f",
  "#d8d088",
  "#bfc86a",
  "#8fc36b",
  "#8eaf76",
  "#63af99",
  "#4da0b5",
  "#4d8ec2",
  "#4d7da2",
  "#6b78b7",
  "#766192",
  "#8d689b",
  "#b466ab",
  "#cf81ba",
  "#e39ea9",
  "#d4828d"
];

const unitData = unitRaw
  .flatMap((d) => [
    {country: d["Country/City"], bedrooms: 1, price: +d["1-Bed Price (USD)"]},
    {country: d["Country/City"], bedrooms: 2, price: +d["2-Bed Price (USD)"]},
    {country: d["Country/City"], bedrooms: 3, price: +d["3-Bed Price (USD)"]}
  ])
  .filter((d) => Number.isFinite(d.price))
  .map((d) => ({
    ...d,
    country: String(d.country ?? "")
      .split(",")[0]
      .replace(/^the\s+/i, "")
      .trim()
  }));
```

<!-- Unit Size/Bedroom x Price (Map Chart) -->
```js
import * as topojson from "topojson-client";
import * as Inputs from "@observablehq/inputs";

// Load world map TopoJSON
const world = await FileAttachment("data/countries-110m.json").json();
const countries = topojson.feature(world, world.objects.countries);

const mapBudgetInput = (() => {
  const el = Inputs.range([30000, 13000000], {
    value: 10000,
    step: 10000
  });
  el.classList.add("slider-control", "budget-control");
  el.querySelector('input[type="range"]')?.classList.add("slider");
  el.querySelector('input[type="number"]')?.classList.add("slider-number");
  return el;
})();
const mapBudget = view(mapBudgetInput);

const mapBedroomsInput = (() => {
  const root = document.createElement("div");
  root.className = "button-group";
  root.value = 1;

  const label = document.createElement("div");
  label.className = "button-group-label";
  root.appendChild(label);

  const buttons = [1, 2, 3].map((n) => {
    const b = document.createElement("button");
    b.type = "button";
    b.textContent = String(n);
    b.className = "button-group-btn";
    b.addEventListener("click", () => {
      root.value = n;
      updateActive();
      root.dispatchEvent(new Event("input"));
    });
    root.appendChild(b);
    return b;
  });

  function updateActive() {
    buttons.forEach((b, i) => {
      b.classList.toggle("is-active", i + 1 === root.value);
    });
  }

  updateActive();
  return root;
})();
const mapBedrooms = view(mapBedroomsInput);

// Build the choropleth
function priceChoropleth(unitData, countries, maxPrice, minBedrooms, {width} = {}) {
  const countryPrice = d3.rollup(
    unitData.filter((d) => d.price <= maxPrice && d.bedrooms >= minBedrooms),
    (v) => d3.mean(v, (d) => d.price),
    (d) => d.country
  );

  return Plot.plot({
    projection: "equal-earth",
    width,
    height: width / 2,
    color: {
      legend: true,
      position: 'bottom',
      type: "sequential",
      range: ["#8eaf76","#ddc56f", "#d7a76e", "#b46b68"],
      unknown: "#eee",
      domain: [30000, d3.max(Array.from(countryPrice.values()))]
    },
    marks: [
      Plot.geo(countries, {
        fill: (d) => countryPrice.get(d.properties.name),
        title: (d) =>
          `${d.properties.name}\n$${countryPrice.get(d.properties.name)?.toLocaleString() ?? "No data"}`
      }),
      Plot.geo(topojson.mesh(world, world.objects.countries, (a, b) => a !== b), {
        stroke: "white",
        strokeWidth: 1
      })
    ]
  });
}
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>Countries That Fits Requirements</h2>
    <div class="grid grid-cols-2 controls-row">
      <div class="card">
        <h2>Housing Budget (USD)</h2>
        ${mapBudgetInput}
      </div>
      <div class="card">
        <h2>Amount of Bedrooms</h2>
        ${mapBedroomsInput}
      </div>
    </div>
    ${resize(width => priceChoropleth(unitData, countries, mapBudget, mapBedrooms, {width}))}
  </div>
</div>

<!-- Affordability x Year x Country (Input + Bar Chart) -->
```js
import {Generators} from "@observablehq/runtime";

const yearInputEl = (() => {
  const years = Array.from(new Set(housingData.map((d) => d.year))).sort(d3.ascending);
  const minYear = years[0];
  const maxYear = years[years.length - 1];

  const root = document.createElement("div");
  root.className = "slider-control year-control";
  root.value = maxYear;

  const label = document.createElement("label");
  root.appendChild(label);

  const range = document.createElement("input");
  range.type = "range";
  range.min = String(minYear);
  range.max = String(maxYear);
  range.step = "1";
  range.value = String(maxYear);
  range.className = "slider";
  root.appendChild(range);

  const ticks = document.createElement("div");
  ticks.className = "year-ticks";
  ticks.style.gridTemplateColumns = `repeat(${years.length}, 1fr)`;
  years.forEach((y) => {
    const t = document.createElement("span");
    t.className = "year-tick";
    t.textContent = String(y);
    ticks.appendChild(t);
  });
  root.appendChild(ticks);

  function syncValue(v) {
    const next = Math.min(maxYear, Math.max(minYear, Math.round(+v || maxYear)));
    root.value = next;
    range.value = String(next);
    root.dispatchEvent(new Event("input"));
  }

  range.addEventListener("input", (e) => syncValue(e.target.value));

  return root;
})();
const yearInput = view(yearInputEl);

function affordabilityBar(data, year, {width} = {}) {
  const filtered = data.filter(d => d.year === year);
  return Plot.plot({
    width,
    height: 500,
    y: {label: null, axis: null, domain: filtered.map((d) => d.country)},
    x: {grid: true, label: "Affordability Ratio"},
    color: {type: "categorical", legend: false, domain: countryDomain, range: countryColours},
    marks: [
      Plot.barX(filtered, {x: "affordability_ratio", y: "country", fill: "country", tip: true}),
      Plot.text(filtered, {
        x: "affordability_ratio",
        y: "country",
        text: "country",
        dx: -6,
        textAnchor: "end",
        fill: "white"
      }),
      Plot.ruleX([0])
    ]
  });
}
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>Affordability by Country</h2>
    <div class="card year-card">
      <h2>Year</h2>
      ${yearInputEl}
    </div>
    ${resize(width => affordabilityBar(housingData, yearInput, {width}))} 
    </div>
</div>

<!-- House Price Index x Year x Country (Line Chart) -->
```js
function housePriceTrend(data, {width} = {}) {
  return Plot.plot({
    width,
    height: 300,
    y: {grid: true, label: "House Price Index", domain: [80, 180]},
    x: {label: "Year", tickFormat: d3.format("d")},
    color: {type: "categorical", legend: true, domain: countryDomain, range: countryColours},
    marks: [
      Plot.line(data, {x: "year", y: "house_price_index", stroke: "country", tip: true})
    ]
  });
}
```

<div class="grid grid-cols-1"> 
  <div class="card">
    <h2>House Price Index Over Time</h2>
    ${resize(width => housePriceTrend(housingData, {width}))}
  </div> 
</div>

<!-- Mortgage Rate x Affordability (Scatter Chart) -->
```js
function mortgageScatter(data, {width} = {}) {
  return Plot.plot({
    width,
    height: 300,
    x: {label: "Mortgage Rate (%)"},
    y: {label: "Affordability Ratio", grid: true},
    color: {type: "categorical", legend: true, domain: countryDomain, range: countryColours},
    marks: [
      Plot.dot(data, {
        x: "mortgage_rate", 
        y: "affordability_ratio", 
        fill: "country",
        opacity: 0.5, 
        tip: true
      }),
      Plot.linearRegressionY(data, {
        x: "mortgage_rate",
        y: "affordability_ratio",
        stroke: "country",
        strokeWidth: 2,
        fillOpacity: 0
      })
    ]
  });
}
```

<div class="grid grid-cols-1"> 
  <div class="card"> 
    <h2>Mortgage Rate vs Housing Affordability</h2>
    ${resize(width => mortgageScatter(housingData, {width}))} 
  </div>
</div>

<!-- Inflation Rates x Affordability (Treemap) -->
```js
function affordabilityTreemap(data, year, {width} = {}) {
  const height = width;
  const format = d3.format(",.2f");
  const filtered = data.filter(
    (d) =>
      d.year === year &&
      Number.isFinite(d.affordability_ratio) &&
      String(d.country ?? "").trim() !== ""
  );

  const root = d3
    .treemap()
    .tile(d3.treemapSquarify)
    .size([width, height])
    .padding(1)
    .round(true)(
      d3
        .hierarchy({children: filtered.map((d) => ({name: d.country, value: d.affordability_ratio}))})
        .sum((d) => d.value)
        .sort((a, b) => b.value - a.value)
    );

  const color = d3.scaleOrdinal(countryDomain, countryColours);

  const svg = d3
    .create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .attr("style", "max-width: 100%; height: auto; font: 10px sans-serif;");

  const leaf = svg
    .selectAll("g")
    .data(root.leaves())
    .join("g")
    .attr("transform", (d) => `translate(${d.x0},${d.y0})`);

  leaf
    .append("title")
    .text((d) => `${d.data.name}\n${format(d.value)}`);

  leaf
    .append("rect")
    .attr("id", (d, i) => `leaf-${i}`)
    .attr("fill", (d) => color(d.data.name))
    .attr("fill-opacity", 0.7)
    .attr("width", (d) => d.x1 - d.x0)
    .attr("height", (d) => d.y1 - d.y0);

  leaf
    .append("clipPath")
    .attr("id", (d, i) => `clip-${i}`)
    .append("use")
    .attr("xlink:href", (d, i) => `#leaf-${i}`);

  leaf
    .append("text")
    .attr("clip-path", (d, i) => `url(#clip-${i})`)
    .selectAll("tspan")
    .data((d) => d.data.name.split(/\s+/g).concat(format(d.value)))
    .join("tspan")
    .attr("x", 3)
    .attr("y", (d, i, nodes) => `${(i === nodes.length - 1) * 0.3 + 1.1 + i * 0.9}em`)
    .attr("fill-opacity", (d, i, nodes) => (i === nodes.length - 1 ? 0.7 : null))
    .text((d) => d);

  return svg.node();
}
```

<div class="grid grid-cols-1">
  <div class="card"> 
    <h2>Inflation's Impact on Housing Affordability</h2>
    ${resize(width => affordabilityTreemap(housingData, yearInput, {width}))}
  </div>
</div>


<!-- CSS -->
<style>
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: "Lato", var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  font-family: "DM Serif Display";
  letter-spacing: 0.2px;
  font-style: normal;
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 80px;
  font-weight: 900;
  line-height: 1;
  background: #000;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}


.button-group {
  display: grid;
  gap: 0.5rem;
}

.button-group-label {
  font-family: "Lato";
  font-weight: 600;
}

label {
  font-family: "Lato";
}

.button-group-btn {
  border: 1px solid #000;
  border-radius: 100px;
  background: transparent;
  color: #000;
  padding: 0.35rem 0.8rem;
  cursor: pointer;
}

.button-group-btn.is-active {
  background: #000;
  color: #fff;
}

.controls-row {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
  align-items: stretch;
}

.controls-row .card {
  display: flex;
  align-items: stretch;
  height: auto;
}

.controls-row .card > * {
  width: 100%;
}

.slider-control {
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

.year-control {
  --thumb-size: 25px;
}

.slider-control input[type="number"] {
  order: 1;
  border: 1px solid #000;
  border-radius: 100px;
  background: transparent;
  color: #000;
  padding: 0.35rem 0.8rem;
  cursor: pointer;

}

.slider-control input[type="range"] {
  width: 100%;
  margin-top: 0.6rem;
  order: 2;
}

.budget-control {
  display: flex;
  flex-direction: column;
}

.budget-control label {
  order: 1;
}

.budget-control input[type="number"] {
  order: 2;
  width: 80%;
}

.budget-control input[type="range"] {
  order: 3;
  width: 90%;
}

.year-ticks {
  display: grid;
  margin-top: 0.4rem;
  font-size: 0.75rem;
  color: #000;
  justify-items: center;
  padding: 0 calc(var(--thumb-size, 25px) / 2);
}

.slider {
  -webkit-appearance: none;
  width: 100%;
  height: 15px;
  border-radius: 100px;
  border: 1px solid #000;
  border-radius: 100px;
  background: transparent;
  -webkit-transition: 0.2s;
  transition: opacity 0.2s;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 25px;
  height: 25px;
  border-radius: 50%;
  background: #000;
  cursor: pointer;
}

.slider::-moz-range-thumb {
  width: 25px;
  height: 25px;
  border-radius: 50%;
  background: #000;
  cursor: pointer;
}

h3 {
  font-family: "Lato";
  letter-spacing: 0.2px;
  font-style: normal;
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-weight: 500;
  line-height: 1.5;
  color: #000;
}

h2 {
  font-family: "DM Serif Display";
  letter-spacing: 0.2px;
  font-style: normal;
  margin: 0;
  margin-bottom: 10px;
  font-size: 24px;
  font-weight: 500;
  line-height: 1.5;
  color: #000;
}


</style>
