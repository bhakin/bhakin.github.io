<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  import data from '..//data/gil.json';

  const dispatch = createEventDispatcher();

  let selectedPair = ["FF", "CH"];
  const pitchPairs = [
    ["FF", "CH"],
    ["FF", "SL"],
    ["FF", "FC"],
    ["CH", "SL"],
    ["CH", "FC"],
    ["SL", "FC"]
  ];
  let sliders = { w0: 0, w1: 0.5, w2: 0.5 };

  function handleSelection(event) {
    selectedPair = JSON.parse(event.target.value);
    console.log('Selected pair:', selectedPair);
    dispatch('selectedPairChanged', selectedPair);
    drawChart();
  }

  onMount(() => {
    console.log('Fetched data:', data);
    drawChart();
  });

  function drawChart() {
    const margin = { top: 10, right: 30, bottom: 50, left: 60 };
    const width = 460 - margin.left - margin.right;
    const height = 400 - margin.top - margin.bottom;

    d3.select("#scatterplot").select("svg").remove();

    const svg = d3.select("#scatterplot")
      .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    let filteredData = data.filter(d => selectedPair.includes(d.pitch_type));

    const x = d3.scaleLinear()
      .domain([d3.min(filteredData, d => d.release_spin_rate) - 100, d3.max(filteredData, d => d.release_spin_rate) + 100])
      .range([0, width]);
    svg.append("g")
      .attr("transform", "translate(0," + height + ")")
      .call(d3.axisBottom(x));

    svg.append("text")
      .attr("x", width / 2)
      .attr("y", height + margin.bottom - 10)
      .style("text-anchor", "middle")
      .text("Release Spin Rate");

    const y = d3.scaleLinear()
      .domain([d3.min(filteredData, d => d.release_speed) - 5, d3.max(filteredData, d => d.release_speed) + 5])
      .range([height, 0]);
    svg.append("g")
      .call(d3.axisLeft(y));

    svg.append("text")
      .attr("transform", "rotate(-90)")
      .attr("y", -margin.left + 20)
      .attr("x", -height / 2)
      .attr("dy", "-1em")
      .style("text-anchor", "middle")
      .text("Release Speed");

    const color = d3.scaleOrdinal()
      .domain(["FF", "CH", "SL", "FC"])
      .range(["#FF0000", "#0000FF", "#800080", '#FFA500']);

    const tooltip = d3.select("#tooltip");

    const highlight = function(event, d) {
      let selected_pitch_type = d.pitch_type;

      d3.selectAll(".dot")
        .transition()
        .duration(200)
        .style("fill", "lightgrey")
        .attr("r", 3);

      d3.selectAll("." + selected_pitch_type)
        .transition()
        .duration(200)
        .style("fill", color(selected_pitch_type))
        .attr("r", 7);

      tooltip.style("display", "block")
        .html(`Pitch Type: ${d.pitch_type}`)
        .style("left", event.pageX + 10 + "px")
        .style("top", event.pageY + "px");
    };

    const doNotHighlight = function(event, d) {
      d3.selectAll(".dot")
        .transition()
        .duration(200)
        .style("fill", d => color(d.pitch_type))
        .attr("r", 5);
      tooltip.style("display", "none");
    };

    svg.append('g')
      .selectAll("dot")
      .data(filteredData)
      .enter()
      .append("circle")
      .attr("class", d => "dot " + d.pitch_type)
      .attr("cx", d => x(d.release_spin_rate))
      .attr("cy", d => y(d.release_speed))
      .attr("r", 5)
      .style("fill", d => color(d.pitch_type))
      .on("mouseover", highlight)
      .on("mouseleave", doNotHighlight);

    svg.selectAll(".decision-boundary").remove();

    const line = d3.line()
      .x(d => x(d[0]))
      .y(d => y(d[1]));

    const decisionBoundary = [
      [x.domain()[0], (-sliders.w0 - sliders.w1 * x.domain()[0]) / sliders.w2],
      [x.domain()[1], (-sliders.w0 - sliders.w1 * x.domain()[1]) / sliders.w2]
    ];

    console.log('Decision Boundary:', decisionBoundary);

    svg.append("path")
      .datum(decisionBoundary)
      .attr("class", "decision-boundary")
      .attr("d", line)
      .attr("stroke", "black")
      .attr("stroke-width", 2)
      .attr("fill", "none");
  }

  $: drawChart();
</script>

<div id='dropdown-container'>
  <div>
    <label for='pitchPairSelect'>Select Pitch Pair:</label>
    <select id='pitchPairSelect' on:change={handleSelection}>
      {#each pitchPairs as pair}
        <option value={JSON.stringify(pair)}>{pair.join(" & ")}</option>
      {/each}
    </select>
  </div>
</div>

<div id="chart-container">
  <div id="scatterplot"></div>
  <div id="tooltip" style="display: none; position: absolute;"></div>
</div>

<div id="slider-container">
  <label for="w0-slider">w0:</label>
  <input type="range" id="w0-slider" min="-50" max="50" step="1" bind:value={sliders.w0} />

  <label for="w1-slider">w1:</label>
  <input type="range" id="w1-slider" min="-1" max="1" step="0.01" bind:value={sliders.w1} />

  <label for="w2-slider">w2:</label>
  <input type="range" id="w2-slider" min="-1" max="1" step="0.01" bind:value={sliders.w2} />
</div>

<style>
  #chart-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 50vh;
  }
  #dropdown-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 10vh;
  }
</style>

