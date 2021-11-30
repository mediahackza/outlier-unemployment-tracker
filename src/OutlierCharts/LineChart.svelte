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
      formatDate = ' - September ' + d[1]
    } else if (d[0] === 'Jan-Mar') {
      formatDate = ' - March ' + d[1]
    } else if (d[0] === 'Apr-Jun') {
      formatDate = ' - June ' + d[1]
    } else if (d[0] === 'Oct-Dec') {
      formatDate = ' - December ' + d[1]
    }
    return formatDate
  }

  $: curDate = updateTimeLabels(data[data.length - 1].months)
  $: conf.title = `Unemployment ${curDate}`
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

  function addLineChart(data) {
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

    let x = d3.scaleBand().range([0, width]).domain(xRange)

    let y = d3
      .scaleLinear()
      .range([height, 0])
      .domain([0, d3.max(data, (d) => d[yValue] * 1.2)])

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

    // Add area background
    if (conf.chartFill) {
      let area = svg
        .datum(data)
        .append('path')
        .attr('class', 'chart-area-fill')
        .attr(
          'd',
          d3
            .area()
            .x((d) => x(d.months))
            .y0((d) => y(d[conf.chartFillValue]))
            .y1(height)
        )

      if (conf.chartFillColor) {
        area.style('fill', conf.chartFillColor)
      }
      if (conf.chartFillOpacity) {
        area.style('opacity', conf.chartFillOpacity)
      }
    }

    // add lines
    conf.lines.forEach((yVal) => {
      let line = svg
        .datum(data)
        .append('path')
        .attr('fill', 'none')
        .attr('class', 'chart-line')
        .attr(
          'd',
          d3
            .line()
            .curve(d3.curveCardinal.tension(0.02))
            .x((d) => x(d[xValue]))
            .y((d) => y(d[yVal.value]))
        )
        .style('stroke-width', yVal.lineWeight)

      if (yVal.color) {
        line.style('stroke', yVal.color)
      }
      // add line end value
      if (yVal.showEndValue) {
        let offsetX = 0
        let offsetY = 0
        if (yVal.offsetX) {
          offsetX = yVal.offsetX
        }
        if (yVal.offsetY) {
          offsetY = yVal.offsetY
        }

        let label = svg
          .append('text')
          .attr('x', x(data[data.length - 1][xValue]) + offsetX)
          .attr('y', y(data[data.length - 1][yVal.value]) + offsetY)
          .text(data[data.length - 1][yVal.value] + '%')
          .attr('class', 'chart-label')
          .style('text-anchor', 'start')

        if (yVal.color) {
          label.style('fill', yVal.color)
        }
      }
    })

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
        let tt = `<div class="tt-title">${b[xValue]}</div>`
        conf.lines.forEach((l) => {
          tt += `<div class="tt-row"><div class="tt-label" style="color: ${
            l.color
          };">${l.label}</div><div class="tt-value" style="color: ${
            l.color
          };">${b[l.value]}%</div></div>`
        })
        tooltip.innerHTML = tt
        tooltip.style.top = d.pageY - 10 + 'px'
        tooltip.style.left = d.pageX + 10 + 'px'

        showTooltip()
      })
      .on('mouseout', function (d, b) {
        hideTooltip()
        indicatorLine.style('display', 'none')
      })

    // add label to end of line
    // if (conf.endYLabel) {

    // }

    // svg
    //   .selectAll('circle')
    //   .data(data)
    //   .enter()
    //   .append('circle')
    //   .attr('cx', (d) => x(d[xValue]))
    //   .attr('cy', (d) => y(d[yValue]))
    //   .attr('r', 7)
    //   .style('fill', 'var(--transparent)')
    //   .on('mouseover', function (d, b) {
    //     tooltip.innerHTML = `<div class="tt-row">${b[yValue]}%</div>`
    //     showTooltip()
    //     d3.select(this)
    //       .style('fill', 'var(--red)')
    //       .style('stroke', '#fff')
    //       .style('stroke-width', 3)
    //   })
    //   .on('mousemove', (d) => {
    //     tooltip.style.left = d.pageX + 10 + 'px'
    //     tooltip.style.top = d.pageY - tooltip.offsetHeight - 20 + 'px'
    //   })
    //   .on('mouseout', function (d) {
    //     d3.select(this)
    //       .style('fill', 'var(--transparent)')
    //       .style('stroke', 'none')
    //     hideTooltip()
    //   })
  }

  ;(function () {
    onMount(() => {
      tooltip = document.querySelector('.tooltip')
      addLineChart(data)
    })
  })()
</script>

<figure>
  <h5 class="chart-title">{conf.title}</h5>
  <div id={chartId} class="chart" />
  <div class="chart-source">{conf.source}</div>
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
