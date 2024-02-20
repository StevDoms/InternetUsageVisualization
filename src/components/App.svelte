<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tempData = [];
  let svg, x, y, width, height;
  let regionInput = '';

  onMount(async () => {
    const res = await fetch('Internet_data.csv');
    const csv = await res.text();
    tempData = d3.csvParse(csv, d3.autoType);
    renderGraph();
    filterData(); // Initial call to filterData to handle initial rendering with no input
  });

  function filterData() {
    const filteredData = regionInput.trim() === '' ? tempData : tempData.filter(d =>
      d.Region && typeof d.Region === 'string' && d.Region.toLowerCase().includes(regionInput.toLowerCase())
    );

    updateGraph(filteredData);
  }

  function renderGraph() {
    const margin = { top: 60, right: 100, bottom: 60, left: 100 };
    width = 960 - margin.left - margin.right;
    height = 500 - margin.top - margin.bottom;

    svg = d3.select('#my_dataviz')
      .append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    x = d3.scaleTime()
      .domain(d3.extent(tempData, d => new Date(d.Year, 0, 1)))
      .range([0, width]);

    y = d3.scaleLinear()
      .domain([0, d3.max(tempData, d => d.Value)])
      .range([height, 0]);

    svg.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(x));

    // X axis label
    svg.append('text')
      .attr('transform', `translate(${width / 2}, ${height + margin.bottom - 20})`)
      .style('text-anchor', 'middle')
      .style('font-family', 'Arial, Helvetica, sans-serif')
      .text('Years');

    svg.append('g')
      .call(d3.axisLeft(y));

    // Y axis label
    svg.append('text')
      .attr('transform', 'rotate(-90)')
      .attr('y', 0 - margin.left)
      .attr('x', 0 - (height / 2))
      .attr('dy', '1em')
      .style('text-anchor', 'middle')
      .style('font-family', 'Arial, Helvetica, sans-serif')
      .text('Internet Usage Percentage %');
  }

  function updateGraph(filteredData) {
    const dataByRegion = d3.groups(filteredData, d => d.Region);
    const line = d3.line()
      .x(d => x(new Date(d.Year, 0, 1)))
      .y(d => y(d.Value));

    const lines = svg.selectAll('.line')
      .data(dataByRegion, d => d[0]);

    lines.enter()
      .append('path')
        .attr('class', 'line')
        .attr('fill', 'none')
        .attr('stroke', d => getColor(d[0]))
        .attr('stroke-width', 3)
      .merge(lines)
      .attr('d', d => line(d[1]))
      .on('mouseenter', (event, d) => {
        d = d[1]

        const [xPosition, yPosition] = d3.pointer(event);
        const date = x.invert(xPosition);

        const bisector = d3.bisector(d => new Date(d.Year, 0, 1)).left;
        const index = bisector(d, date, 1);
        const start = d[index - 1];
        const end = d[index];

        const day = date.getDate();
        const month = date.getMonth() + 1;
        const year = date.getFullYear();
        const outputDate = `${month}/${day}/${year}`;

        let predictedValue = getPercentage(start, end, date).toFixed(2);

        let tooltip = d3.select('#tooltip');
        tooltip
          .style('visibility', 'visible')
          .html(`
            <h1 style="color: #000; font-size: 18px;">${d[0].Region}</h1>
            <p style="color: #000; font-size: 14px;">Date: ${outputDate}</p>
            <p style="color: #000; font-size: 14px;">Value: ${predictedValue}</p>
          `)
          .style('left', `${event.pageX + 15}px`)
          .style('top', `${event.pageY - 28}px`);
      })
      .on('mousemove', function(event, d) {
        d = d[1]

        const [xPosition, yPosition] = d3.pointer(event);
        const date = x.invert(xPosition);

        const bisector = d3.bisector(d => new Date(d.Year, 0, 1)).left;
        const index = bisector(d, date, 1);
        const start = d[index - 1];
        const end = d[index];

        const day = date.getDate();
        const month = date.getMonth() + 1;
        const year = date.getFullYear();
        const outputDate = `${month}/${day}/${year}`;

        let predictedValue = getPercentage(start, end, date).toFixed(2);

        let tooltip = d3.select('#tooltip');
        tooltip
          .style('visibility', 'visible')
          .html(`
            <h1 style="color: #000; font-size: 18px;">${d[0].Region}</h1>
            <p style="color: #000; font-size: 14px;">Date: ${outputDate}</p>
            <p style="color: #000; font-size: 14px;">Value: ${predictedValue}</p>
          `)
          .style('left', `${event.pageX + 15}px`)
          .style('top', `${event.pageY - 28}px`);
      })
      .on('mouseout', (event) => { 
        let tooltip = d3.select('#tooltip');
        tooltip.style('visibility', 'hidden');
      });

    lines.exit().remove();
  }

  function getColor(region) {
    const colors = d3.scaleOrdinal(d3.schemeCategory10);
    return colors(region);
  }

  function getPercentage(start, end, date) {
    const timestamp1 = new Date(start.Year, 0, 1).getTime();
    const timestamp2 = new Date(end.Year, 0, 1).getTime();

    const slope = (end.Value - start.Value) / (timestamp2 - timestamp1);
    const intercept = start.Value - (slope * timestamp1);

    return slope * date.getTime() + intercept
  }

  function handleKeydown(event) {
    if (event.keyCode === 13) {  // Check if the key pressed is Enter
      filterData();
    }
  }

</script>

<style>

  .container {
    display: flex;
    flex-direction: column;   
    justify-content: center;
    align-items: center;    
    height: 100vh;          
  }

  .title {
    position: absolute;
    top: 0;
    left: 50%;
    transform: translateX(-50%);
    font-size: 30px;
    font-weight: bold;
    font-family: 'Arial', 'Helvetica', sans-serif;
    margin-top: 20px;
  }

  .filter {
    display: flex;
    height: 40px;
    justify-content: center;
    align-items: center;    
  }

  .input {
    padding: 10px;
    margin: 5px 0;
    border: 2px solid #ddd;
    border-radius: 4px;
    font-size: 16px;
    width: calc(100% - 24px);
    margin-right: 10px;
  }

  .button {
    padding: 10px 20px;
    background-color: #4682B4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
  }

  .button:hover {
    background-color: #4682B4;
    opacity: 0.7;
  }

  #tooltip {
    position: absolute;
    visibility: hidden;
    pointer-events: none;
    background-color: lightsteelblue;
    padding: 10px;
    padding-top: 0px;    
    padding-bottom: 0px; 
    border-radius: 5px;
    font-family: Arial, Helvetica, sans-serif;
  }
</style>

<main>
  <div class="container">
    <h1 class="title">
      Interactive Visualization of Country/Region's Internet Usage
    </h1>
    <div class="filter">
      <input type="text" bind:value={regionInput} placeholder="Enter region to filter..." on:keydown={handleKeydown} class='input'/>
      <button on:click={filterData} class='button'>Filter</button>
    </div>
    <div class="graph">
      <div id="my_dataviz"></div>
      <div id="tooltip"></div>
    </div>
  </div>
</main>
