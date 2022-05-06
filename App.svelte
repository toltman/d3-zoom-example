<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";
	import AxisHorizontal from "./AxisHorizontal.svelte";
	import AxisVertical from "./AxisVertical.svelte";
	import Tooltip from "./Tooltip.svelte";

	// access data
	let data = [];
	d3.csv(
	  "https://raw.githubusercontent.com/the-pudding/data/master/mars-weather/mars-weather.csv"
	).then(res => {
	  data = res.slice(0, 100);
	});

	let yMetric = "max_temp";

	$: metricOptions = Object.keys(data[0] || {});

	const xAccessor = d => new Date(d.terrestrial_date);
	$: yAccessor = d => parseInt(d[yMetric]);

	// create chart dimensions
	let margin = {
	  left: 30,
	  top: 0,
	  right: 0,
	  bottom: 30
	};
	let width = 400;
	let height = 300;
	$: boundsWidth = width - margin.left - margin.right;
	$: boundsHeight = height - margin.top - margin.bottom;

	let xRange = [0, 0];
	$: boundsWidth, (xRange = [0, boundsWidth]);
	// create scales
	$: xScale = d3
	  .scaleTime()
	  .domain(d3.extent(data, xAccessor))
	  .range(xRange);

	$: yScale = d3
	  .scaleLinear()
	  .domain(d3.extent(data, yAccessor))
	  .range([boundsHeight, 0])
	  .nice(true);

	// set up interactions
	let hoveredPoint = null;
	$: quadtree = d3
	  .quadtree()
	  .x(d => xScale(xAccessor(d)))
	  .y(d => yScale(yAccessor(d)))
	  .addAll(data);

	let svgElement; // hook onto the svg element in the DOM
	const addZoom = () => {
	  const zoomGenerator = d3
	    .zoom()
	    .extent([[0, 0], [width, height]])
	    .scaleExtent([1, 8]) // restrict zooming
      .translateExtent([  // restrict panning
        [0, 0],
        [boundsWidth, boundsHeight]
      ])
	    .on("zoom", event => {
	      // update the range of our xScale
	      xRange = [event.transform.applyX(0), event.transform.applyX(boundsWidth)];
	    });

	  // d3 uses "d3 selection objects" to work with DOM elements
	  // https://github.com/d3/d3-selection
	  // we need to create one to pass to our zoom generator
	  const d3SelectionOfSvg = d3.select(svgElement);
	  zoomGenerator(d3SelectionOfSvg);
	};
	// whenever our svgElement changes,
	// add the zoom behavior
	$: svgElement, addZoom();
</script>

<select bind:value={yMetric}>
	{#each metricOptions as option}
    <option value={option}>{option}</option>
	{/each}
</select>

<h2>Weather on Mars over time</h2>

<figure>
  <div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
    <svg
      width={width}
      height={height}
      bind:this={svgElement}>

      <g transform="translate({margin.left}, {margin.top})">

        {#each data as d,i (d.id)}
          <circle
            cx={xScale(xAccessor(d))}
            cy={yScale(yAccessor(d))}
            r="5"
            style="
            "
		  			fill={hoveredPoint === d ? "skyblue" : "sienna"}
            stroke={hoveredPoint === d ? "black" : "none"}
          />
        {/each}

				<rect
					width={boundsWidth}
					height={boundsHeight}
					fill="transparent"
					on:mousemove={e => {
						const pos = d3.pointer(e)
						const x = pos[0]
						const y = pos[1]
						const closestPoint = quadtree.find(x, y)
						if (!closestPoint) return
						const hoveredPointPosition = [
							xScale(xAccessor(closestPoint)),
							yScale(yAccessor(closestPoint))
						]
						// don't highlight if too far away
						// a^2 + b^2 = c^2
						const distance = Math.sqrt(
							(x - hoveredPointPosition[0]) ** 2
							+ (y - hoveredPointPosition[1]) ** 2
						)
						if (distance < 50) {
							hoveredPoint = closestPoint
						} else {
							hoveredPoint = null
						}
					}}
				/>

      </g>
      <g transform="translate({margin.left}, {boundsHeight + margin.top})">
        <AxisHorizontal
          scale={xScale}
          count="5"
          format={d3.timeFormat("%-m/%Y")}
        />
      </g>
      <g transform="translate({margin.left}, 0)">
        <AxisVertical
          count="5"
          scale={yScale}
        />
      </g>
    </svg>

    <div
      class="label"
      style="transform: translate({margin.left + 6}px, -10px)">
      {yMetric}
    </div>

		{#if hoveredPoint}
			<Tooltip
				x={margin.left + xScale(xAccessor(hoveredPoint))}
				y={margin.top + yScale(yAccessor(hoveredPoint))}
				placement={[
					xScale(xAccessor(hoveredPoint)) > boundsWidth - 50 ? -90 : -50,
					yScale(yAccessor(hoveredPoint)) < 80 ? 0 : -100,
				]}
			>
				<strong>
					{d3.timeFormat("%b %-d, %Y")(xAccessor(hoveredPoint))}
				</strong>
				<div>
					max temp: {hoveredPoint.max_temp}°C
				</div>
			</Tooltip>
		{/if}
  </div>
</figure>

<style>
	select {
	  width: 100%;
	  padding: 0.5em 1em;
	}

	.wrapper {
	  position: relative;
	  margin: 0;
	  font-family: sans-serif;
	  width: 100%;
	  height: 300px;
	  min-width: 0;
	}

	svg {
	  overflow: visible;
	}

	h2 {
	  text-align: center;
	  font-family: sans-serif;
	}

	.label {
	  position: absolute;
	  top: 0;
	  left: 0;
	  max-width: 4em;
	  font-size: 0.6em;
	}
</style>