<script>
  import * as d3 from 'd3'
  import { onMount } from 'svelte'
  export let data
  export let tooltip
  export let conf
  if (!conf.margins) {
    conf.margins = {
      top: 0,
      right: 50,
      bottom: 30,
      left: 50,
    }
  }

  function updateTimeLabels(input) {
    let d = input.split(' ')
    let formatDate = null
    if (d[0] === 'Jul-Sep') {
      formatDate = 'at September ' + d[1]
    } else if (d[0] === 'Jan-Mar') {
      formatDate = 'at March ' + d[1]
    } else if (d[0] === 'Apr-Jun') {
      formatDate = 'at June ' + d[1]
    } else if (d[0] === 'Oct-Dec') {
      formatDate = 'at December ' + d[1]
    }
    return formatDate
  }
  $: curDate = updateTimeLabels(data[data.length - 1].months)

  $: conf.title = `Youth (15-24 years of age) unemployed ${curDate}`
  function randomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1) + min)
  }
  let chartId = 'chart-' + randomInt(1000, 10000)

  let xValue = conf.xValue
  let yValue = conf.yValue

  function showTooltip() {
    tooltip.style.display = 'block'
  }
  function hideTooltip() {
    tooltip.style.display = 'none'
  }

  function addBarChart(data) {
    let chart = document.getElementById(chartId)
    let chartWidth = chart.offsetWidth
    let chartHeight = chartWidth * 0.6
    let width = chartWidth - conf.margins.left - conf.margins.right
    let height = chartHeight - conf.margins.top - conf.margins.bottom

    let svg = d3
      .select('#' + chartId)
      .append('svg')
      .attr('width', width + conf.margins.left + conf.margins.right)
      .attr('height', height + conf.margins.top + conf.margins.bottom)
      .append('g')
      .attr('transform', `translate(${conf.margins.left}, ${conf.margins.top})`)

    // Check for limits on the x-axis
    let xRange = []
    data.forEach((d) => {
      xRange.push(d[xValue])
    })
    let xLabels = []
    if (conf.xLabelCount) {
      data.forEach((d, i) => {
        if (i % conf.xLabelCount === 0) {
          xLabels.push(d[xValue])
        }
      })
    } else {
      xLabels = xRange
    }

    let yRange
    if (!conf.yRange) {
      yRange = [0, d3.max(data, (d) => d[yValue] * 1.2)]
    } else {
      yRange = conf.yRange
    }

    let x = d3.scaleBand().range([0, width]).domain(xRange)

    let y = d3.scaleLinear().range([height, 0]).domain(yRange)

    function x_axis() {
      return d3.axisBottom(x)
      // return d3.axisBottom(x).tickFormat(d3.timeFormat('%e %b')) Format time axis
    }

    var formatPercent = function (d) {
      return d3.format('')(d) + '%'
    }

    function y_axis() {
      return d3.axisLeft(y).tickFormat(formatPercent)
      //return d3.axisLeft(y).tickFormat(d3.format('.1s')) Format y axis
    }

    function y_grid() {
      return d3.axisLeft(y).tickFormat('')
    }

    svg
      .append('g')
      .attr('class', 'y-grid')
      .call(
        y_grid()
          .ticks(5)
          .tickSize(-width + 10, 0, 0)
          .tickFormat('')
      )

    svg
      .append('g')
      .attr('class', 'axis x-axis')
      .attr('transform', `translate(0,${height})`)
      .call(x_axis().tickValues(xLabels))

    svg.append('g').attr('class', ' axis y-axis').call(y_axis().ticks(5))

    // add bars

    let bars = svg
      .selectAll('.bars')
      .data(data)
      .enter()
      .append('rect')
      .attr('fill', 'blue')
      .attr('class', 'chart-bar')
      .attr('x', (d) => x(d[xValue]))
      .attr('y', (d) => y(d[yValue]))
      .attr('width', width / data.length - 2)
      .attr('height', (d) => height - y(d[yValue]))
    // .on('mouseover', function (d) {
    //   d3.select(this).style('display', 'none')
    // })

    if (conf.barFillColor) {
      bars.style('fill', conf.barFillColor)
    }

    // Add hover lines
    let indicatorLine = svg
      .append('line')
      .attr('x1', 10)
      .attr('x2', 10)
      .attr('y1', 0)
      .attr('y2', height)
      .attr('class', 'indicatorLine')

    let hoverLines = svg
      .selectAll('.hoverLines')
      .data(data)
      .enter()
      .append('rect')
      .attr('x', (d) => x(d[xValue]) - width / data.length / 2)
      .attr('y', 0)
      .attr('height', height)
      .attr('width', width / data.length)
      .style('fill', '#ffffff00')
      // .style('stroke', 'gray')
      .style('opacity', 0.3)
      .on('mouseover', function (d, b) {
        indicatorLine
          .attr('x1', x(b[xValue]))
          .attr('x2', x(b[xValue]))
          .style('display', 'block')

        // add hover labels
        // set color
        let labelColor = '#000'
        if (conf.barFillColor) {
          labelColor = conf.barFillColor
        }
        if (conf.labelColor) {
          labelColor = conf.labelColor
        }

        let tt = `<div class="tt-title">${b[xValue]}</div>`
        tt += `<div class="tt-row"><div class="tt-label" style="color: ${labelColor};">${b[xValue]}</div><div class="tt-value" style="color: ${labelColor};">${b[yValue]}%</div></div>`

        tooltip.innerHTML = tt
        tooltip.style.top = d.pageY - 10 + 'px'
        tooltip.style.left = d.pageX + 10 + 'px'

        showTooltip()
      })
      .on('mouseout', function (d, b) {
        hideTooltip()
        indicatorLine.style('display', 'none')
      })

    if (conf.showEndValue) {
      let offsetX = 0
      let offsetY = 0
      if (conf.offsetX) {
        offsetX = conf.offsetX
      }
      if (conf.offsetY) {
        offsetY = conf.offsetY
      }

      let label = svg
        .append('text')
        .attr('x', x(data[data.length - 1][xValue]) + offsetX)
        .attr('y', y(data[data.length - 1][conf.yValue]) + offsetY)
        .text(data[data.length - 1][conf.yValue] + '%')
        .attr('class', 'chart-label')
        .style('text-anchor', 'start')

      if (conf.color) {
        label.style('fill', conf.color)
      }
    }
  }

  ;(function () {
    onMount(() => {
      tooltip = document.querySelector('.tooltip')
      addBarChart(data)
    })
  })()
</script>

<figure>
  <h5 class="chart-title">{conf.title}</h5>
  <div id={chartId} class="chart" />
  <div class="chart-source">Based on the expanded unemployment definition</div>
</figure>

<style>
  .chart {
    position: relative;
    width: 100%;
    margin: 0 auto;
  }
  .chart-title {
    margin-left: 20px;
  }

  /* .latest-data {
    position: absolute;
    top: 20px;
    left: 20px;
    line-height: 100%;
    left: -10000px;
  } */
  /* .latest-number {
    background: var(--darkgreen);
    color: #fff;
    padding: 4px 6px;
    font-size: 80%;
    border-radius: 5px;
  } */
  /* .latest-date {
    font-size: 70%;
    margin-top: 2px;
    margin-bottom: 3px;
  } */
</style>
