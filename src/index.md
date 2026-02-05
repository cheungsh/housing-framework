---
theme: dashboard
title: Global Housing Dashboard
toc: false
---
<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Serif+Display:ital@0;1&family=Lato:ital,wght@0,100;0,300;0,400;0,700;0,900;1,100;1,300;1,400;1,700;1,900&display=swap');
</style>

<div class="page-grid">
<div class="hero layout-hero">
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
  mortgage_rate: +d["Mortgage Rate (%)"],
  inflation_rate: +d["Inflation Rate (%)"]
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
  const el = Inputs.range([0, 20000000], {
    value: 1000,
    step: 1000
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
function priceChoropleth(unitData, countries, budget, minBedrooms, {width} = {}) {
  const countryPrice = d3.rollup(
    unitData.filter((d) => d.bedrooms >= minBedrooms),
    (v) => d3.mean(v, (d) => d.price),
    (d) => d.country
  );

  const affordabilityScore = (name) => {
    const price = countryPrice.get(name);
    if (!Number.isFinite(price) || price <= 0) return null;
    return budget / price;
  };

  const colorScale = d3
    .scaleLinear()
    .domain([0.5, 1.0, 1.5])
    .range(["#b46b68", "#ddc56f", "#8eaf76"])
    .clamp(true);

  return Plot.plot({
    projection: "equal-earth",
    width,
    height: width / 2,
    marks: [
      Plot.geo(countries, {
        fill: (d) => {
          const score = affordabilityScore(d.properties.name);
          return score == null ? "#eee" : colorScale(score);
        },
        title: (d) => {
          const name = d.properties.name;
          const price = countryPrice.get(name);
          const score = affordabilityScore(name);
          if (!Number.isFinite(price)) return `${name}\nNo data`;
          return `${name}\nAvg price: $${price.toLocaleString()}\nAffordability: ${d3.format(".2f")(score)}`;
        }
      }),
      Plot.geo(topojson.mesh(world, world.objects.countries, (a, b) => a !== b), {
        stroke: "white",
        strokeWidth: 1
      })
    ]
  });
}
```

<div class="layout-map">
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
    <div class="map-legend">
      <div class="map-legend-bar"></div>
      <div class="map-legend-labels">
        <span>Unaffordable</span>
        <span>Somewhat Affordable</span>
        <span>Affordable</span>
      </div>
    </div>
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

  const track = document.createElement("div");
  track.className = "year-track";
  root.appendChild(track);

  const range = document.createElement("input");
  range.type = "range";
  range.min = String(minYear);
  range.max = String(maxYear);
  range.value = String(maxYear);
  range.className = "slider";
  track.appendChild(range);
  

  const ticks = document.createElement("div");
  ticks.className = "year-ticks";
  ticks.style.gridTemplateColumns = `repeat(${years.length}, 1fr)`;
  years.forEach((y) => {
    const t = document.createElement("span");
    t.className = "year-tick";
    t.textContent = String(y);
    ticks.appendChild(t);
  });
  track.appendChild(ticks);

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

<div class="layout-affordability">
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
    y: {grid: true, label: "House Price Index", domain: [70, 190]},
    x: {label: "Year", tickFormat: d3.format("d")},
    color: {type: "categorical", legend: true, domain: countryDomain, range: countryColours},
    marks: [
      Plot.line(data, {x: "year", y: "house_price_index", stroke: "country", tip: true})
    ]
  });
}
```

<div class="layout-hpi"> 
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
    x: {label: "Mortgage Rate (%)"},
    y: {label: "Affordability Ratio", grid: true, domain: [2, 13]},
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

<div class="layout-mortgage"> 
  <div class="card"> 
    <h2>Mortgage Rate vs Housing Affordability</h2>
    ${resize(width => mortgageScatter(housingData, {width}))} 
  </div>
</div>

<!-- Inflation Rates x Affordability (Treemap) -->
```js

const inflationTreemapData = (data, year) => ({
  name: "root",
  children: data
    .filter((d) => d.year === year && Number.isFinite(d.inflation_rate))
    .map((d) => ({
      country: d.country,
      inflation_rate: d.inflation_rate
    }))
});

const formatInflation = d3.format(".1f");

function affordabilityTreemap(data, { // data is either tabular (array of objects) or hierarchy (nested objects)
  path,
  id = Array.isArray(data) ? d => d.id : null, // if tabular data, given a d in data, returns a unique identifier (string)
  parentId = Array.isArray(data) ? d => d.parentId : null, // if tabular data, given a node d, returns its parent’s identifier
  children, // if hierarchical data, given a d in data, returns its children
  value, // given a node d, returns a quantitative value (for area encoding; null for count)
  sort = (a, b) => d3.descending(a.value, b.value), // how to sort nodes prior to layout
  label, // given a leaf node d, returns the name to display on the rectangle
  group, // given a leaf node d, returns a categorical value (for color encoding)
  title, // given a leaf node d, returns its hover text
  link, // given a leaf node d, its link (if any)
  linkTarget = "_blank", // the target attribute for links (if any)
  tile = d3.treemapBinary, // treemap strategy
  width = 640, // outer width, in pixels
  height = 593, // outer height, in pixels
  margin = 0, // shorthand for margins
  marginTop = margin, // top margin, in pixels
  marginRight = margin, // right margin, in pixels
  marginBottom = margin, // bottom margin, in pixels
  marginLeft = margin, // left margin, in pixels
  padding = 1, // shorthand for inner and outer padding
  paddingInner = padding, // to separate a node from its adjacent siblings
  paddingOuter = padding, // shorthand for top, right, bottom, and left padding
  paddingTop = paddingOuter, // to separate a node’s top edge from its children
  paddingRight = paddingOuter, // to separate a node’s right edge from its children
  paddingBottom = paddingOuter, // to separate a node’s bottom edge from its children
  paddingLeft = paddingOuter, // to separate a node’s left edge from its children
  round = true, // whether to round to exact pixels
  colors = d3.schemeTableau10, // array of colors
  zDomain, // array of values for the color scale
  fill = "#ccc", // fill for node rects (if no group color encoding)
  fillOpacity = group == null ? null : 0.6, // fill opacity for node rects
  stroke, // stroke for node rects
  strokeWidth, // stroke width for node rects
  strokeOpacity, // stroke opacity for node rects
  strokeLinejoin, // stroke line join for node rects
} = {}) {

  // If id and parentId options are specified, or the path option, use d3.stratify
  // to convert tabular data to a hierarchy; otherwise we assume that the data is
  // specified as an object {children} with nested objects (a.k.a. the “flare.json”
  // format), and use d3.hierarchy.

  // We take special care of any node that has both a value and children, see
  // https://observablehq.com/@d3/treemap-parent-with-value.
  const stratify = data => (d3.stratify().path(path)(data)).each(node => {
    if (node.children?.length && node.data != null) {
      const child = new d3.Node(node.data);
      node.data = null;
      child.depth = node.depth + 1;
      child.height = 0;
      child.parent = node;
      child.id = node.id + "/";
      node.children.unshift(child);
    }
  });
  const root = path != null ? stratify(data)
      : id != null || parentId != null ? d3.stratify().id(id).parentId(parentId)(data)
      : d3.hierarchy(data, children);

  // Compute the values of internal nodes by aggregating from the leaves.
  value == null ? root.count() : root.sum(d => Math.max(0, d ? value(d) : null));

  // Prior to sorting, if a group channel is specified, construct an ordinal color scale.
  const leaves = root.leaves();
  const G = group == null ? null : leaves.map(d => group(d.data, d));
  if (zDomain === undefined) zDomain = G;
  zDomain = new d3.InternSet(zDomain);
  const color = group == null ? null : d3.scaleOrdinal(zDomain, colors);

  // Compute labels and titles.
  const L = label == null ? null : leaves.map(d => label(d.data, d));
  const T = title === undefined ? L : title == null ? null : leaves.map(d => title(d.data, d));

  // Sort the leaves (typically by descending value for a pleasing layout).
  if (sort != null) root.sort(sort);

  // Compute the treemap layout.
  d3.treemap()
      .tile(tile)
      .size([width - marginLeft - marginRight, height - marginTop - marginBottom])
      .paddingInner(paddingInner)
      .paddingTop(paddingTop)
      .paddingRight(paddingRight)
      .paddingBottom(paddingBottom)
      .paddingLeft(paddingLeft)
      .round(round)
    (root);

  const svg = d3.create("svg")
      .attr("viewBox", [-marginLeft, -marginTop, width, height])
      .attr("width", width)
      .attr("height", height)
      .attr("style", "max-width: 100%; height: auto; height: intrinsic;")
      .attr("font-family", "Lato")
      .attr("font-size", 10);

  const node = svg.selectAll("a")
    .data(leaves)
    .join("a")
      .attr("xlink:href", link == null ? null : (d, i) => link(d.data, d))
      .attr("target", link == null ? null : linkTarget)
      .attr("transform", d => `translate(${d.x0},${d.y0})`);

  node.append("rect")
      .attr("fill", color ? (d, i) => color(G[i]) : fill)
      .attr("fill-opacity", fillOpacity)
      .attr("stroke", stroke)
      .attr("stroke-width", strokeWidth)
      .attr("stroke-opacity", strokeOpacity)
      .attr("stroke-linejoin", strokeLinejoin)
      .attr("width", d => d.x1 - d.x0)
      .attr("height", d => d.y1 - d.y0);

  if (T) {
    node.append("title").text((d, i) => T[i]);
  }

  if (L) {
    // A unique identifier for clip paths (to avoid conflicts).
    const uid = `O-${Math.random().toString(16).slice(2)}`;

    node.append("clipPath")
       .attr("id", (d, i) => `${uid}-clip-${i}`)
     .append("rect")
       .attr("width", d => d.x1 - d.x0)
       .attr("height", d => d.y1 - d.y0);

    node.append("text")
        .attr("clip-path", (d, i) => `url(${new URL(`#${uid}-clip-${i}`, location)})`)
      .selectAll("tspan")
      .data((d, i) => `${L[i]}`.split(/\n/g))
      .join("tspan")
        .attr("x", 3)
        .attr("y", (d, i, D) => `${(i === D.length - 1) * 0.3 + 1.1 + i * 0.9}em`)
        .attr("fill-opacity", (d, i, D) => i === D.length - 1 ? 0.7 : null)
        .text(d => d);   
  }

  return Object.assign(svg.node(), {scales: {color}});
}
```

<div class="layout-inflation">
  <div class="card"> 
    <h2>Inflation's Impact on Housing Affordability</h2>
    ${resize(width => affordabilityTreemap(inflationTreemapData(housingData, yearInput), {
      children: (d) => d.children,
      value: (d) => d.inflation_rate,
      label: (d) => `${d.country}\n${formatInflation(d.inflation_rate)}%`,
      group: (d) => d.country,
      title: (d) => `${d.country}\nInflation Rate: ${formatInflation(d.inflation_rate)}%`,
      width
    }))}
  </div>
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
  font-size: 64px;
  font-weight: 900;
  line-height: 1;
  background: #000;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.page-grid {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  gap: 1rem;
  align-items: stretch;
}

.layout-hero {
  grid-column: 1 / span 5;
  grid-row: 1;
}

.layout-affordability {
  grid-column: 6 / span 7;
  grid-row: 1;
}

.layout-map {
  grid-column: 1 / span 7;
  grid-row: 2;
}

.layout-inflation {
  grid-column: 8 / span 5;
  grid-row: 2;
}

.layout-hpi {
  grid-column: 1 / span 6;
  grid-row: 3;
}

.layout-mortgage {
  grid-column: 7 / span 6;
  grid-row: 3;
}

@media (max-width: 1100px) {
  .page-grid {
    grid-template-columns: 1fr;
  }

  .page-grid > * {
    grid-column: 1 / -1;
    grid-row: auto;
  }
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

.year-track {
  width: 100%;
  padding-left: calc(var(--thumb-size, 25px) / 2);
  padding-right: calc(var(--thumb-size, 25px) / 2);
  box-sizing: border-box;
}

.year-track input[type="range"] {
  width: 100%;
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
  margin-top: 0.8rem;
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

.map-legend {
  margin-top: 0.8rem;
}

.map-legend-bar {
  height: 14px;
  border-radius: 999px;
  background: linear-gradient(90deg, #b46b68, #ddc56f, #8eaf76);
  border: 1px solid #000;
}

.map-legend-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  margin-top: 0.35rem;
  color: #000;
  font-family: "Lato";
}

.year-track .year-ticks {
  display: flex;
  justify-content: space-between;
  margin-top: 0.6rem;
  font-size: 0.75rem;
  color: #000;
  width: 100%;
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
