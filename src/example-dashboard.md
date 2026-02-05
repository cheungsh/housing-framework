---
theme: dashboard
title: Global Housing Dashboard
toc: false
---

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

const unitData = unitRaw
  .flatMap((d) => [
    {country: d["Country/City"], bedrooms: 1, price: +d["1-Bed Price (USD)"]},
    {country: d["Country/City"], bedrooms: 2, price: +d["2-Bed Price (USD)"]},
    {country: d["Country/City"], bedrooms: 3, price: +d["3-Bed Price (USD)"]}
  ])
  .filter((d) => Number.isFinite(d.price));
```

<!-- Cards with big numbers -->

<div class="grid grid-cols-4">
  <div class="card">
    <h2>Total Countries 🌎</h2>
    <span class="big">${new Set(housingData.map((d) => d.country)).size}</span>
  </div>
  <div class="card">
    <h2>Most Expensive 🏙️</h2>
    <span class="big"> ${d3.max(housingData, d => d.house_price_index)?.toLocaleString()} </span>
  </div>
  <div class="card">
    <h2>Most Affordable 🏡</h2>
    <span class="big"> ${d3.min(housingData, d => d.affordability_ratio)?.toLocaleString()} </span>
  </div>
  <div class="card">
    <h2>Average Mortgage Rate % 💰</h2>
    <span class="big">${d3.mean(housingData, d => d.mortgage_rate)?.toFixed(2)} </span>
  </div>
</div>

<!-- Plot of launch history -->

```js
function affordabilityTimeline(data, {width} = {}) {
  return Plot.plot({
    title: "Affordability Over Time",
    width,
    height: 300,
    y: {grid: true, label: "Affordability Ratio"},
    x: {label: "Year"},
    color: {type: "categorical", legend: true},
    marks: [
      Plot.line(data, {x: "year", y: "affordability_ratio", stroke: "country", tip: true}),
      Plot.ruleY([0])
    ]
  });
}
```

<div class="grid grid-cols-1"> 
  <div class="card"> ${resize(width => affordabilityTimeline(housingData, {width}))}</div> 
</div>

```js
function priceScatter(data, {width} = {}) {
  return Plot.plot({
    title: "Bedrooms vs Price",
    width,
    height: 600,
    x: {label: "Bedrooms"},
    y: {label: "Price (USD)", grid: true},
    color: {type: "categorical", legend: true},
    marks: [
      Plot.dot(data, {x: "bedrooms", y: "price", fill: "country", tip: true})
    ]
  });
}
```

<div class="grid grid-cols-1"> 
  <div class="card"> ${resize(width => priceScatter(unitData, {width}))} </div> 
</div>

```js
const budget = view(
  Inputs.range([30000, 13000000], {value: 500000, step: 10000, label: "Budget (USD)"})
);
const size = view(Inputs.range([1, 3], {value: 1, step: 1, label: "Bedrooms"}));

function budgetLookup(data, budget, size, {width} = {}) {
  const filtered = data.filter((d) => d.price <= budget && d.bedrooms >= size);
  
  return Plot.plot({
    title: `Places You Can Afford (Budget ≤ $${budget}, Bedrooms ≥ ${size})`,
    width,
    height: 300,
    x: {label: "Country"},
    y: {label: "Price (USD)"},
    marks: [
      Plot.barY(filtered, {x: "country", y: "price", fill: "country", tip: true}),
      Plot.ruleY([0])
    ]
  });
}
```

<div class="grid grid-cols-1">
  <div class="card"> ${resize(width => budgetLookup(unitData, budget, size, {width}))} </div>
</div>
