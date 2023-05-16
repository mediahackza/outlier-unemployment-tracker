<script>
  import * as d3 from 'd3'
  let parseTime = d3.timeParse('%Y-%m-%d')
  let formatDate = d3.timeFormat('%e %B, %Y')

  import LineChart from './OutlierCharts/LineChart.svelte'
  import BarChart from './OutlierCharts/BarChart.svelte'
  export let data = []
  export let margins

  export let currentUnemployment = 0
  export let previousUnemployment = 0
  export let quarterChange
  export let womenUnemployed
  export let menUnemployed
  export let updateDate = ''
  export let labourForce = 0
  export let notActive = 0
  export let endMonth = ''
  export let percentDiscouraged = 0
  export let totalDiscouraged = 0
  export let currentYouth = 0
  export let previousYouth = 0
  export let youthChange;
  export let latestYear = 2022;

  $: quarterChange = calculateChange(currentUnemployment, previousUnemployment)
  $: youthChange = calculateYouthChange(currentYouth, previousYouth)

  function calculateYouthChange(currentYouth, previousYouth) {
    let text = ''
    let change = currentYouth - previousYouth
    if (change < 0) {
      text = `decreased to ${currentYouth}% from last quarter's ${previousYouth}%`
    } else {
      text = `increased to ${currentYouth}% from last quarter's ${previousYouth}%`
    }
    return text
  }

  function calculateChange(currentUnemployment, previousUnemployment) {
    let text = ''
    if (currentUnemployment > 0) {
      let change = (currentUnemployment - previousUnemployment).toFixed(1)
      if (change < 0) {
        text = `${change * -1} percentage points lower`
      } else {
        text = `${change} percentage points higher`
      }
    }
    return text
  }

  async function getData() {
    await fetch(
      // 'https://api.datadesk.co.za/csvjson.php?table=unemployment_overview_695402'
      // 'https://api.datadesk.co.za/json.php?table=dd_unemployment_4493905'
      'https://api.datadesk.co.za/csvjson.php?table=dd_unemployment_may_31_2022_1936608' // dd_unemployment_may_31_2022_1936608
    )
      .then((response) => response.json())
      .then((response) => {
        latestYear = response[response.length - 1].year
        response.forEach((d) => {
          // d['15_to_24_years_unemployment_rate'] =
          //   +d['15_to_24_years_unemployment_rate']
          d.percent_unemployed = (+d.percent_unemployed.replace(
            '%',
            ''
          )).toFixed(1)
          d.percent_not_active = +d.percent_not_active.replace('%', '')
          d.women_unemployment_rate = +d.women_unemployment_rate.replace(
            '%',
            ''
          )
          d.men_unemployment_rate = +d.men_unemployment_rate.replace('%', '')
        })

        data = response.filter((d) => d.months !== '')

        // set initial values
        currentYouth =
          +data[data.length - 1]['15_to_24_years_unemployment_rate']
        previousYouth =
          +data[data.length - 2]['15_to_24_years_unemployment_rate']

        currentUnemployment = data[data.length - 1].percent_unemployed
        previousUnemployment = data[data.length - 2].percent_unemployed
        womenUnemployed = data[data.length - 1].women_unemployment_rate
        menUnemployed = data[data.length - 1].men_unemployment_rate
        updateDate = formatDate(
          parseTime(data[data.length - 1].date_report_released)
        )
        labourForce = (
          +data[data.length - 1]['population_15_64_yrs'] / 1000
        ).toFixed(1)
        notActive = (+data[data.length - 1]['total_not_active'] / 1000).toFixed(
          1
        )
        let pD = +data[data.length - 1].percent_disaffected.replace('%', '')
        percentDiscouraged = pD.toFixed(1)

        totalDiscouraged = data[data.length - 1].total_disaffected
        // calculate endMonth
        let eM = data[data.length - 1].months
        let eMS = eM.split('-')
        let eMD = eMS[1].split(' ')
        let m = 'March'
        if (eMD[0] === 'Jun') {
          m = 'June'
        } else if (eMD[0] === 'Sep') {
          m = 'September'
        } else if (eMD[0] === 'Dec') {
          m = 'December'
        }
        endMonth = m + ' ' + eMD[1]

        // console.log(currentUnemployment)
        // console.log(previousUnemployment)
      })
  }

  let promise = Promise.all([getData()]).then(() => {
    // console.log(data)
  })

  let chart1 = {
    type: 'linechart',
    xValue: 'months',
    yValue: 'women_unemployment_rate',
    lines: [
      {
        value: 'women_unemployment_rate',
        lineWeight: 3,
        showEndValue: true,
        offsetX: 5,
        offsetY: 0,
        color: 'var(--red)',
        label: 'Women',
      },
      {
        value: 'men_unemployment_rate',
        lineWeight: 3,
        showEndValue: true,
        offsetX: 5,
        offsetY: 10,
        color: 'var(--blue)',
        label: 'Men',
      },
      {
        value: 'percent_unemployed',
        lineWeight: 0,
        color: 'var(--black)',
        showEndValue: true,
        offsetX: 5,
        offsetY: 5,
        label: 'Overall',
      },
    ],
    xLabelCount: 12,
    title: `Unemployment `,
    chartFill: true,
    chartFillColor: 'var(--darkGreen)',
    chartFillValue: 'percent_unemployed',
    source: 'Source: Stats SA Quarterly Labour Force',
  }

  let chart2 = {
    type: 'barchart',
    xValue: 'months',
    yValue: '15_to_24_years_unemployment_rate',
    xLabelCount: 8,
    title: `Youth (15-24 years of age) unemployed `,
    barFillColor: 'var(--green)',
    labelColor: 'var(--blue)',
    showEndValue: true,
    offsetX: 10,
    offsetY: 0,
    yRange: [0, 110],
  }

  function removeScroll() {
    document.querySelector('.scroll-table').classList.remove('scroll')
  }
  function addScroll() {
    document.querySelector('.scroll-table').classList.add('scroll')
  }
</script>

<div class="body">
  <main>
    <div class="tooltip shadow">Tooltip</div>
    <!-- <h1>SA Employment Tracker</h1> -->
    <div class="tracker-update">
      This page is updated quarterly. Last updated: <span class="highlight"
        >{updateDate}
      </span>
    </div>
    <!-- {#await promise}
      <p class="loading">
        Fetching the most recent data ...<br />
        <img src="assets/loading.svg" class="loader" alt="loading" />
      </p>
    {:then} -->
    <section>
      <p>
        Unemployment in South Africa is currently <span class="highlight"
          >{currentUnemployment}%</span
        >. This is <span class="highlight">{quarterChange}</span> than the
        <span class="highlight">{previousUnemployment}%</span> rate in the previous
        quarter.
      </p>

      <p>
        <span class="highlight">{womenUnemployed}% of women</span> compared to
        <span class="highlight">{menUnemployed}% of men</span> are unemployed.
      </p>

      <p>
        In January 2008, only 26.6% of women and 20.5% of men were unemployed.
      </p>
    </section>
    <div>
      {#await promise}
        ...
      {:then}
        <LineChart bind:data bind:margins bind:conf={chart1} />
      {/await}
    </div>
    <section>
      <p>
        For the youth aged 15 to 24 years, the unemployment rate in {endMonth}
        <span class="highlight">{youthChange}</span>. According to the chief
        director of labour statistics at Stats SA, Malerato Mosiane, the reason
        the youth unemployment statistics are counted from 15-years-old instead
        of 18 is based on the South African Schools Act where children must
        attend school until the age of 15 or grade 9, whichever occurs first.
        And the Basic Conditions of Employment Act states that it is a criminal
        offence to employ a person younger than 15.
      </p>
    </section>
    {#await promise}
      ...
    {:then}
      <BarChart bind:data bind:margins bind:conf={chart2} />
    {/await}
    <section>
      <h4>Not economically active</h4>

      <p>
        The employed and unemployed between the ages of 15 and 64 make up the
        labour force, and it has grown from <span class="highlight"
          >31.5 million</span
        >
        people in 2008 to
        <span class="highlight">{labourForce} million in {latestYear}</span>.
      </p>

      <p>
        But there are still <span class="highlight">{notActive} million</span>
        of working age who are "not economically active".
      </p>

      <p>
        Not economically active is the broad definition for people between the
        ages of 15 and 64 who were neither employed or unemployed a week before
        they were interviewed by StatsSA for the QLFS.
      </p>

      <p>
        The group includes: early retirees, homemakers and students. It also
        includes discouraged work-seekers - people who want to work but have
        given up looking for work because there are "no jobs available in the
        area" or they are "unable to find work requiring his/her skills" and
        have "lost hope of finding any kind of work."
      </p>

      <p>
        As of <span class="highlight">{endMonth}</span>,
        <span class="highlight"
          >{percentDiscouraged}% of South Africans were discouraged job seekers</span
        >, an increase from 2008 when 3.8% had given up looking for work.
      </p>
    </section>

    <figure>
      <h5 class="chart-title">Unemployment Jan 2008 onward</h5>
      <table class="scroll scroll-table">
        <tr
          ><th>Quarter</th><th>Total</th><th>Women</th><th>Men</th><th
            >Youth(15-24)</th
          ></tr
        >
        {#each data as d}
          <tr>
            <td>{d.months}</td>
            <td>{d.percent_unemployed}%</td><td>{d.women_unemployment_rate}%</td
            >
            <td>{d.men_unemployment_rate}%</td>
            <td>{d['15_to_24_years_unemployment_rate']}%</td>
          </tr>
        {/each}
      </table>
      <div class="scroll-note">
        Scroll for more or click to <span
          class="click-link-remove"
          on:click={removeScroll}>see full table</span
        ><span class="click-link-add" on:click={addScroll}>trim table</span>
      </div>
    </figure>

    <h4>Process</h4>

    <p>
      Each quarter, Statistics South Africa releases the <a
        href="http://www.statssa.gov.za/publications/P0211/P02111stQuarter2021.pdf"
        target="_blank">Quarterly Labour Force Survey</a
      >.
    </p>

    <p>
      The data is collected typically through in-person interviews with <span
        class="highlight">30,000</span
      > households based on a dataset from the 2011 Census.
    </p>

    <p>
      Stats SA suspended in-person interviews in favour of telephonic ones in
      March 2020 because of the Covid-19 pandemic.
    </p>
    <!-- {/await} -->
  </main>
</div>

<style>
  @media (max-width: 600px) {
  }
</style>
