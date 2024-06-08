 <script>
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  import SVM from 'ml-svm';
  import data from '../data/gil.json';

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

  function handleSelection(event) {
    selectedPair = JSON.parse(event.target.value);
    dispatch('selectedPairChanged', selectedPair);
    drawChart();
  }

  onMount(() => {
    drawChart();
  });

  $: drawChart();

  function drawChart() {
    const margin = { top: 10, right: 30, bottom: 50, left: 60 };
    const width = 600;
    const height = 400;

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

    // Tooltip setup
    const tooltip = d3.select("#scatterplot")
      .append("div")
      .style("position", "absolute")
      .style("background", "white")
      .style("padding", "5px")
      .style("border", "1px solid black")
      .style("border-radius", "5px")
      .style("pointer-events", "none")
      .style("opacity", 0);

    svg.selectAll("circle")
      .data(filteredData)
      .enter()
      .append("circle")
      .attr("cx", d => x(d.release_spin_rate))
      .attr("cy", d => y(d.release_speed))
      .attr("r", 5)
      .style("fill", "steelblue")
      .on("mouseover", (event, d) => {
        tooltip.transition().duration(200).style("opacity", .9);
        tooltip.html(`Release Spin Rate: ${d.release_spin_rate}<br/>Release Speed: ${d.release_speed}`)
          .style("left", (event.pageX + 5) + "px")
          .style("top", (event.pageY - 28) + "px");
      })
      .on("mouseout", () => {
        tooltip.transition().duration(500).style("opacity", 0);
      });

    const color = d3.scaleOrdinal()
      .domain(["FF", "CH", "SL", "FC"])
      .range(["#FF0000", "#0000FF", "#800080", '#FFA500']);


    const doNotHighlight = function(event, d) {
      d3.selectAll(".dot")
        .transition()
        .duration(200)
        .style("fill", d => color(d.pitch_type))
        .attr("r", 5);
      tooltip.style("display", "none");
    };

    // Prepare the data for SVM training
    var labelMapping = { [selectedPair[0]]: 1, [selectedPair[1]]: -1 };
    var features = filteredData.map(d => [d.release_spin_rate, d.release_speed]);
    var labels = filteredData.map(d => labelMapping[d.pitch_type]);

    // Train the SVM model

    const svm = new SVM({
      kernel: 'linear',
      type: 'C_SVC',
      cost: 1
    });

    svm.train(features, labels);

    const w = svm.W
    const b = svm.b

    const xScale = d3.scaleLinear().domain([0,6]).range([0, width])
    const yScale = d3.scaleLinear().domain([0,6]).range([height,0])

    const x1 = xScale.domain();
    const x2 = x1.map(x => (-w[0] * x - b) / w[1]);

    svg.append('line')
      .attr('x1', xScale(x1[0] + 2.75))
      .attr('y1', yScale(x2[0] + 2.75))
      .attr('x2', xScale(x1[1] + 2.75))
      .attr('y2', yScale(x2[1] + 2.75))
      .attr('stroke', 'green')
      .attr('stroke-width', 2);
    

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
  }
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

