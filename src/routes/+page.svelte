<script>
    import * as d3 from 'd3';
    import { onMount } from 'svelte';
    import SeasonLineChart from '$lib/SeasonLineChart.svelte';
    import MinutesOutliersChart from '$lib/MinutesOutliersChart.svelte';
    import LogoScatterChart from '$lib/LogoScatterChart.svelte';

    let dat = $state([]);
    let pvoDat = $state([]);
    let minDat = $state([]);
    let logoDat = $state([]);
    let pvoSeason = $state(2026);
    let selectedYear = $state(2025);
    let selectedMonth = $state(9); // October — start of 2025-26 season

    const MONTHS = ['January','February','March','April','May','June',
                    'July','August','September','October','November','December'];
    const DAYS = ['Sun','Mon','Tue','Wed','Thu','Fri','Sat'];

    // Season month order: Oct, Nov, Dec, Jan … Sep
    const SEASON_MONTHS       = [9, 10, 11, 0, 1, 2, 3, 4, 5, 6, 7, 8];
    const SEASON_MONTH_LABELS = ['Oct','Nov','Dec','Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep'];

    // season column uses end year: 2002 → "2001-2002"
    function seasonLabel(s) {
        return `${s - 1}-${s}`;
    }

    // Trade deadlines keyed by season end year. Stored as UTC midnight strings for safe comparison.
    const TRADE_DEADLINES = new Map([
        [2002, '2002-02-21'], [2003, '2003-02-20'], [2004, '2004-02-19'],
        [2005, '2005-02-24'], [2006, '2006-02-23'], [2007, '2007-02-22'],
        [2008, '2008-02-21'], [2009, '2009-02-19'], [2010, '2010-02-18'],
        [2011, '2011-02-24'], [2012, '2012-03-15'], [2013, '2013-02-21'],
        [2014, '2014-02-20'], [2015, '2015-02-19'], [2016, '2016-02-18'],
        [2017, '2017-02-23'], [2018, '2018-02-08'], [2019, '2019-02-07'],
        [2020, '2020-02-06'], [2021, '2021-03-25'], [2022, '2022-02-10'],
        [2023, '2023-02-09'], [2024, '2024-02-08'], [2025, '2025-02-06'],
        [2026, '2026-02-05'],
    ]);

    // Bar chart SVG dimensions (height is dynamic)
    const B = { w: 285, mt: 6, mr: 14, mb: 30, ml: 52 };
    const Biw = B.w - B.ml - B.mr;
    const BAR_BAND = 22;   // px per band
    const BAR_BAR_H = BAR_BAND * 0.6;

    // Line chart SVG dimensions (mt=42 reserves space for hover stats above the chart)
    const LC = { w: 285, h: 186, mt: 42, mr: 14, mb: 22, ml: 40 };

    // Playoff vs non-playoff monthly win rate chart
    const PVO = { w: 630, h: 180, mt: 8, mr: 16, mb: 28, ml: 45 };
    const Pvoiw = PVO.w - PVO.ml - PVO.mr;
    const Pvoih = PVO.h - PVO.mt - PVO.mb;
    const PVO_MONTHS     = [10, 11, 12, 1, 2, 3, 4, 5];
    const PVO_MONTH_LBLS = ['Oct','Nov','Dec','Jan','Feb','Mar','Apr','May'];
    const pvoXScale = d3.scalePoint().domain(PVO_MONTH_LBLS).range([0, Pvoiw]).padding(0.5);

    onMount(async () => {
        [dat, pvoDat, minDat, logoDat] = await Promise.all([
            d3.csv('/data/calendar_alltime.csv', d3.autoType),
            d3.csv('/data/playoff_vs_non.csv', d3.autoType),
            d3.csv('/data/minutes_date.csv', d => ({
                game_date: new Date(d.game_date + 'T12:00:00'),
                total_min_outliers: +d.total_min_outliers,
            })),
            d3.csv('/data/minutes_logo.csv', d => ({
                team_display_name: d.team_display_name,
                season: +d.season,
                total_min_outliers: +d.total_min_outliers,
                playoff_team: d.playoff_team === 'TRUE',
                wins: +d.wins,
                rank: +d.rank,
                logo: d.logo,
                logo2: d.logo2,
            })),
        ]);
        // Start at October of the most recent season (season col is end year, so start year = season - 1)
        const latestSeason = dat[dat.length - 1].season;
        selectedYear = latestSeason - 1;
        selectedMonth = 9;
    });

    // season end year derived from calendar position: Oct-Dec → year+1, Jan-Sep → year
    let selectedSeason = $derived(
        selectedMonth >= 9 ? selectedYear + 1 : selectedYear
    );

    let seasons = $derived(
        [...new Set(dat.map(d => d.season))].sort((a, b) => a - b)
    );

    const colorScale = d3.scaleSequential(d3.interpolateReds).domain([1, 25]).clamp(true);

    let dataMap = $derived(
        new Map(dat.map(d => [
            `${d.date.getFullYear()}-${d.date.getMonth()}-${d.date.getDate()}`,
            d
        ]))
    );

    let weeks = $derived.by(() => {
        const startDow = new Date(selectedYear, selectedMonth, 1).getDay();
        const daysInMonth = new Date(selectedYear, selectedMonth + 1, 0).getDate();
        const cells = Array(startDow).fill(null);
        for (let i = 1; i <= daysInMonth; i++) cells.push(i);
        while (cells.length % 7 !== 0) cells.push(null);
        const out = [];
        for (let i = 0; i < cells.length; i += 7) out.push(cells.slice(i, i + 7));
        return out;
    });

    function getDayData(day) {
        if (!day) return null;
        const d = dataMap.get(`${selectedYear}-${selectedMonth}-${day}`) ?? null;
        return d && d.avg_mov !== 0 ? d : null;
    }

    function prevMonth() {
        if (selectedMonth === 0) { selectedMonth = 11; selectedYear--; }
        else selectedMonth--;
    }

    function nextMonth() {
        if (selectedMonth === 11) { selectedMonth = 0; selectedYear++; }
        else selectedMonth++;
    }

    // Clicking a bar navigates to that month in the current season
    // season end year: Oct-Dec are in selectedSeason-1, Jan-Sep are in selectedSeason
    function clickBarMonth(month) {
        selectedMonth = month;
        selectedYear = month >= 9 ? selectedSeason - 1 : selectedSeason;
    }

    // Clicking a season dot navigates to October of that season's start year
    function clickSeason(season) {
        selectedYear = season - 1;
        selectedMonth = 9;
    }

    let barMetric  = $state('mov');   // 'mov' | 'blowout'
    let lineMetric = $state('mov');   // 'mov' | 'blowout'
    let diffMetric = $state('mov');   // 'mov' | 'blowout'

    let tooltip = $state(null);

    function handleMouseEnter(event, day) {
        const data = getDayData(day);
        if (!data) return;
        const rect = event.currentTarget.getBoundingClientRect();
        tooltip = { data, x: rect.left + rect.width / 2, y: rect.top - 8 };
    }

    function handleMouseLeave() {
        tooltip = null;
    }

    // ── Chart data ───────────────────────────────────────────────────────────

    // Monthly weighted avg MoV + blowout rate for current season, only months with data
    let monthlyData = $derived.by(() => {
        if (!dat.length) return [];
        const filtered = dat.filter(d => d.season === selectedSeason && d.avg_mov !== 0);
        const byMonth = d3.rollup(
            filtered,
            v => ({
                mov:     d3.sum(v, d => d.avg_mov * d.games_played) / d3.sum(v, d => d.games_played),
                blowout: d3.sum(v, d => d.blowouts) / d3.sum(v, d => d.games_played)
            }),
            d => d.date.getMonth()
        );
        return SEASON_MONTHS
            .map((m, i) => {
                const val = byMonth.get(m);
                return { month: m, name: SEASON_MONTH_LABELS[i], mov: val?.mov ?? null, blowout: val?.blowout ?? null };
            })
            .filter(d => d.mov !== null);
    });

    // Dynamic bar SVG height based on number of months with data
    let barInnerH = $derived(Math.max(monthlyData.length, 1) * BAR_BAND);
    let barSvgH   = $derived(B.mt + barInnerH + B.mb);

    // Pre- and post-deadline weighted avg MoV + blowout totals per season
    let prePostData = $derived.by(() => {
        if (!dat.length) return [];
        const bySeason = d3.group(dat.filter(d => d.avg_mov !== 0), d => d.season);
        const result = [];
        for (const [season, rows] of bySeason) {
            const dl = TRADE_DEADLINES.get(season);
            if (!dl) continue;
            const pre  = rows.filter(d => d.date.toISOString().slice(0, 10) <  dl);
            const post = rows.filter(d => d.date.toISOString().slice(0, 10) >= dl);
            const wMov = arr => arr.length ? d3.sum(arr, d => d.avg_mov * d.games_played) / d3.sum(arr, d => d.games_played) : null;
            const tBlowout = arr => { const g = d3.sum(arr, d => d.games_played); return g > 0 ? d3.sum(arr, d => d.blowouts) / g : null; };
            result.push({ season, preMov: wMov(pre), postMov: wMov(post), preBlowout: tBlowout(pre), postBlowout: tBlowout(post) });
        }
        return result.sort((a, b) => a.season - b.season);
    });

    // Weighted avg MoV + blowout rate per season across entire dataset
    let seasonData = $derived.by(() => {
        if (!dat.length) return [];
        const filtered = dat.filter(d => d.avg_mov !== 0);
        const bySeason = d3.rollup(
            filtered,
            v => ({
                mov:     d3.sum(v, d => d.avg_mov * d.games_played) / d3.sum(v, d => d.games_played),
                blowout: d3.sum(v, d => d.blowouts) / d3.sum(v, d => d.games_played)
            }),
            d => d.season
        );
        return Array.from(bySeason, ([season, val]) => ({ season, ...val }))
            .sort((a, b) => a.season - b.season);
    });

    // Stable x domains: max across all season-months for each metric
    let barXMaxMov = $derived.by(() => {
        if (!dat.length) return 15;
        const filtered = dat.filter(d => d.avg_mov !== 0);
        const bySeasonMonth = d3.rollup(
            filtered,
            v => d3.sum(v, d => d.avg_mov * d.games_played) / d3.sum(v, d => d.games_played),
            d => `${d.season}-${d.date.getMonth()}`
        );
        return (d3.max(Array.from(bySeasonMonth.values())) ?? 15) * 1.1;
    });

    let barXMaxBlowout = $derived.by(() => {
        if (!dat.length) return 0.5;
        const filtered = dat.filter(d => d.avg_mov !== 0);
        const bySeasonMonth = d3.rollup(
            filtered,
            v => d3.sum(v, d => d.blowouts) / d3.sum(v, d => d.games_played),
            d => `${d.season}-${d.date.getMonth()}`
        );
        return (d3.max(Array.from(bySeasonMonth.values())) ?? 0.5) * 1.1;
    });

    let barXScale = $derived(
        barMetric === 'mov'
            ? d3.scaleLinear().domain([0, barXMaxMov]).range([0, Biw]).nice()
            : d3.scaleLinear().domain([0, barXMaxBlowout]).range([0, Biw]).nice()
    );

    let lineMainData = $derived(
        seasonData.map(d => ({ season: d.season, value: lineMetric === 'mov' ? d.mov : d.blowout }))
    );

    let linePreData = $derived(
        prePostData
            .map(d => ({ season: d.season, value: lineMetric === 'mov' ? d.preMov : d.preBlowout }))
            .filter(d => d.value != null)
    );

    let linePostData = $derived(
        prePostData
            .map(d => ({ season: d.season, value: lineMetric === 'mov' ? d.postMov : d.postBlowout }))
            .filter(d => d.value != null)
    );

    let diffData = $derived(
        prePostData
            .map(d => {
                const v = diffMetric === 'mov'
                    ? (d.postMov != null && d.preMov != null ? d.postMov - d.preMov : null)
                    : (d.postBlowout != null && d.preBlowout != null ? d.postBlowout - d.preBlowout : null);
                return { season: d.season, value: v };
            })
            .filter(d => d.value != null)
    );

    // Playoff vs non-playoff monthly win rate
    let pvoSeasons = $derived(
        [...new Set(pvoDat.map(d => d.season))].sort((a, b) => a - b)
    );

    let pvoLines = $derived.by(() => {
        if (!pvoDat.length) return { playoff: [], nonPlayoff: [] };
        const byMonth = new Map(
            pvoDat.filter(d => d.season === pvoSeason).map(d => [d.month, d.win_pct])
        );
        const playoff = [], nonPlayoff = [];
        PVO_MONTHS.forEach((m, i) => {
            const val = byMonth.get(m);
            if (val !== undefined) {
                nonPlayoff.push({ label: PVO_MONTH_LBLS[i], value: val });
                playoff.push({ label: PVO_MONTH_LBLS[i], value: 1 - val });
            }
        });
        return { playoff, nonPlayoff };
    });

    // Stable y scale across all seasons
    let pvoYScale = $derived.by(() => {
        if (!pvoDat.length) return d3.scaleLinear().domain([0, 1]).range([Pvoih, 0]);
        const allPcts = pvoDat.map(d => d.win_pct);
        const [lo, hi] = d3.extent([...allPcts, ...allPcts.map(v => 1 - v)]);
        return d3.scaleLinear()
            .domain([Math.max(0, lo - 0.03), Math.min(1, hi + 0.03)])
            .range([Pvoih, 0]).nice();
    });

    let pvoPlayoffPath = $derived(
        pvoLines.playoff.length > 0
            ? d3.line().x(p => pvoXScale(p.label)).y(p => pvoYScale(p.value))(pvoLines.playoff)
            : ''
    );
    let pvoNonPlayoffPath = $derived(
        pvoLines.nonPlayoff.length > 0
            ? d3.line().x(p => pvoXScale(p.label)).y(p => pvoYScale(p.value))(pvoLines.nonPlayoff)
            : ''
    );

    // All-season averages per month (faded reference lines)
    let pvoAvgLines = $derived.by(() => {
        if (!pvoDat.length) return { playoff: [], nonPlayoff: [] };
        const byMonth = d3.rollup(pvoDat, v => d3.mean(v, d => d.win_pct), d => d.month);
        const playoff = [], nonPlayoff = [];
        PVO_MONTHS.forEach((m, i) => {
            const val = byMonth.get(m);
            if (val !== undefined) {
                nonPlayoff.push({ label: PVO_MONTH_LBLS[i], value: val });
                playoff.push({ label: PVO_MONTH_LBLS[i], value: 1 - val });
            }
        });
        return { playoff, nonPlayoff };
    });

    let pvoAvgPlayoffPath = $derived(
        pvoAvgLines.playoff.length > 0
            ? d3.line().x(p => pvoXScale(p.label)).y(p => pvoYScale(p.value))(pvoAvgLines.playoff)
            : ''
    );
    let pvoAvgNonPlayoffPath = $derived(
        pvoAvgLines.nonPlayoff.length > 0
            ? d3.line().x(p => pvoXScale(p.label)).y(p => pvoYScale(p.value))(pvoAvgLines.nonPlayoff)
            : ''
    );
</script>

<main>
    <div class="article-text">
        <h1>NBA Tanking is the Worst it has ever been</h1>
        <h5>By: Kyle Suelflow</h5>
        <p>Tanking in the NBA, the process where teams intentionally try to lose games in an effort to increase their odds at a better draft pick, has reached new highs in 2026 (or lows, depending on your viewpoint). Fans have become increasingly dissatisfied with the prevalence of tanking, and the league office has taken notice. Adam Silver is taking action, with the league proposing a new anti-taking draft reform, dubbed the “3-2-1 lottery”. This reform is the latest in a sequence of several changes to the draft lottery and playoff structure that have attempted to curb tanking. The tanking era started in 2013, when Sam Hinkie and the 76ers embarked on a 4 year long journey known as “The Process”.</p>
        <p>Starting with the 2019 draft lottery, the team with the worst record in the NBA <a href = https://www.nba.com/news/nba-draft-lottery-explainer>  no longer had a 25% chance at getting the 1st overall pick.</a> Instead, the odds were flattened and the worst 3 teams all had the same percentage, 14%, of receiving the 1st overall pick. This change attempted to disincentivize teams from trying to have the worst record, although that hasn’t made much of a difference as this article will show. </p>
        <p>A second change the NBA took was in the 2020 season when they first introduced the play-in tournament. With 4 more teams now a part of the playoff bracket, this would incentivize more teams to try to win. While this has undoubtedly been a good change, with more teams involved in the playoffs, this did not address the tanking issue. </p>
        <p>To show how tanking has impacted the gameplay of the NBA, we’ll first take a look at the prevalence of blowouts and the average margin of victory (MoV) in the NBA since 2002. Flip through the calendar to explore each season.</p>
    </div>

    <div class="layout">

        <!-- LEFT: Calendar -->
        <div class="calendar-panel">
            <div class="controls">
                <select
                    value={selectedSeason}
                    onchange={e => clickSeason(+e.currentTarget.value)}
                >
                    {#each seasons as s}
                        <option value={s}>{seasonLabel(s)}</option>
                    {/each}
                </select>
                <div class="month-nav">
                    <button onclick={prevMonth}>&#8592;</button>
                    <span class="month-label">{MONTHS[selectedMonth]}</span>
                    <button onclick={nextMonth}>&#8594;</button>
                </div>
            </div>

            <div class="calendar">
                <div class="day-headers">
                    {#each DAYS as d}
                        <div class="day-header">{d}</div>
                    {/each}
                </div>
                {#each weeks as week}
                    <div class="week">
                        {#each week as day}
                            {#if day}
                                {@const data = getDayData(day)}
                                <div
                                    class="cell"
                                    role="button"
                                    tabindex="0"
                                    style="background: {data ? colorScale(data.avg_mov) : '#f5f5f5'}"
                                    onmouseenter={e => handleMouseEnter(e, day)}
                                    onmouseleave={handleMouseLeave}
                                >
                                    <span class="day-num">{day}</span>
                                </div>
                            {:else}
                                <div class="cell empty"></div>
                            {/if}
                        {/each}
                    </div>
                {/each}
            </div>

            {#if dat.length > 0}
                <div class="legend">
                    <span class="legend-label">MoV 1</span>
                    <div class="legend-bar">
                        {#each d3.range(11) as i}
                            <div class="legend-swatch" style="background: {colorScale(1 + 24 * i / 10)}"></div>
                        {/each}
                    </div>
                    <span class="legend-label">MoV 25+</span>
                </div>
                <p class="caption"><strong>MoV</strong> (margin of victory) is the average point differential in games played that day. A <strong>blowout</strong> is a win by 20 or more points.</p>
            {/if}
        </div>

        <!-- RIGHT: Charts -->
        <div class="charts-panel">

            <!-- Monthly bar chart -->
            <div class="chart-block">
                <div class="chart-header">
                    <h2>{barMetric === 'mov' ? 'Monthly Avg MoV' : 'Monthly Blowout %'} — {seasonLabel(selectedSeason)}</h2>
                    <div class="toggle">
                        <button class:active={barMetric === 'mov'} onclick={() => barMetric = 'mov'}>MoV</button>
                        <button class:active={barMetric === 'blowout'} onclick={() => barMetric = 'blowout'}>Blowouts</button>
                    </div>
                </div>
                <svg width={B.w} height={barSvgH}>
                    <g transform="translate({B.ml},{B.mt})">
                        {#each barXScale.ticks(4) as tick}
                            <line x1={barXScale(tick)} y1={0} x2={barXScale(tick)} y2={barInnerH} stroke="#ebebeb" stroke-width="1" />
                            <text x={barXScale(tick)} y={barInnerH + 14} text-anchor="middle" font-size="10" fill="#999">
                                {barMetric === 'mov' ? tick : `${(tick * 100).toFixed(0)}%`}
                            </text>
                        {/each}

                        {#each monthlyData as { month, name, mov, blowout }, i}
                            {@const val = barMetric === 'mov' ? mov : blowout}
                            {@const valStr = barMetric === 'mov' ? mov.toFixed(1) : `${(blowout * 100).toFixed(1)}%`}
                            {@const y = i * BAR_BAND + (BAR_BAND - BAR_BAR_H) / 2}
                            {@const active = month === selectedMonth}
                            <rect
                                x={0} y={y}
                                width={barXScale(val)}
                                height={BAR_BAR_H}
                                fill={active ? '#c0392b' : '#d0d0d0'}
                                rx="2"
                                role="button"
                                tabindex="0"
                                style="cursor: pointer"
                                onclick={() => clickBarMonth(month)}
                                onkeydown={e => e.key === 'Enter' && clickBarMonth(month)}
                            />
                            <text
                                x={-6} y={y + BAR_BAR_H / 2 + 4}
                                text-anchor="end" font-size="11"
                                fill={active ? '#c0392b' : '#666'}
                                font-weight={active ? '700' : '400'}
                                role="button"
                                tabindex="0"
                                style="cursor: pointer"
                                onclick={() => clickBarMonth(month)}
                                onkeydown={e => e.key === 'Enter' && clickBarMonth(month)}
                            >{name}</text>
                            <text
                                x={barXScale(val) + 4} y={y + BAR_BAR_H / 2 + 4}
                                font-size="10" fill="#888"
                            >{valStr}</text>
                        {/each}

                        <line x1={0} y1={barInnerH} x2={Biw} y2={barInnerH} stroke="#ccc" />
                        <text x={Biw / 2} y={barInnerH + 22} text-anchor="middle" font-size="10" fill="#aaa">
                            {barMetric === 'mov' ? 'Avg Margin of Victory' : 'Blowout Rate'}
                        </text>
                    </g>
                </svg>
            </div>

            <!-- Season line chart -->
            <div class="chart-block">
                <div class="chart-header">
                    <h2>{lineMetric === 'mov' ? 'Avg MoV by Season' : 'Blowouts by Season'}</h2>
                    <div class="toggle">
                        <button class:active={lineMetric === 'mov'} onclick={() => lineMetric = 'mov'}>MoV</button>
                        <button class:active={lineMetric === 'blowout'} onclick={() => lineMetric = 'blowout'}>Blowouts</button>
                    </div>
                </div>
                <SeasonLineChart
                    data={lineMainData}
                    preData={linePreData}
                    postData={linePostData}
                    selectedSeason={selectedSeason}
                    onClickSeason={clickSeason}
                    yFloor={lineMetric === 'mov' ? 9 : 0}
                    formatYTick={v => lineMetric === 'mov' ? v.toFixed(1) : `${(v * 100).toFixed(0)}%`}
                    w={LC.w}
                    h={LC.h}
                    mt={LC.mt}
                    mr={LC.mr}
                    mb={LC.mb}
                    ml={LC.ml}
                    showLegend={true}
                />
            </div>

        </div>
    </div>

    <div class="article-text">
        <p>The data show a clear trend in the NBA: more and more games are not close. Since 2002, there has been a clear uptrend in the percentage of games ending in a blowout, going from 13% in 2002 to 22% in 2026. Additionally, the average MoV has gone from just under 11 points in 2003 to just over 13 points in 2026. Zooming in on the 2026 season, it is clear that this has been the most egregious season of tanking yet. In April, the average MoV was a staggering 16.8 points, the highest ever in a single month. Take April 2nd, for example, where 8 games were played with an average MoV of 23.5 points! As teams decide that they aren’t able to make the playoffs, they quickly pivot to tanking, where losing big is seen as a win. Certainly, this is not what the NBA wants.</p>
        <p>At what point in the season do teams start to decide that they aren’t good enough to make the playoffs? It varies, of course, but a logical place to start is the trade deadline. This is a time where teams might trade their veteran players away for draft picks or young players if they don’t believe they can contend for a championship. The Chicago Bulls are a great example of this. At the trade deadline, they were 24-27, not a great team but certainly still alive in the playoff race. They traded away Coby White, Nikola Vucevic, and Ayo Dosunmu, 3 veteran contributors, and proceeded to end the year with a record of 31-51, going 7-24 after the trade deadline. This is just one example supporting the idea that the trade deadline can be a separator between a more competitive season (pre-deadline) and a less competitive season (post-deadline). </p>
    </div>

    <!-- Diff chart: post-deadline minus pre-deadline, full width -->
    <div class="chart-block diff-block">
        <div class="chart-header">
            <h2>Post-deadline minus Pre-deadline {diffMetric === 'mov' ? 'MoV' : 'Blowout %'} by Season</h2>
            <div class="toggle">
                <button class:active={diffMetric === 'mov'} onclick={() => diffMetric = 'mov'}>MoV</button>
                <button class:active={diffMetric === 'blowout'} onclick={() => diffMetric = 'blowout'}>Blowouts</button>
            </div>
        </div>
        <SeasonLineChart
            data={diffData}
            selectedSeason={selectedSeason}
            onClickSeason={clickSeason}
            showZeroLine={true}
            yFloor={null}
            formatYTick={v => diffMetric === 'mov' ? v.toFixed(1) : `${(v * 100).toFixed(0)}%`}
            mainColor="#9b59b6"
            w={630}
            h={120}
            mt={8}
            mr={16}
            mb={22}
            ml={45}
            showLegend={false}
        />
    </div>

     <div class="article-text">
        <p>The data show that this is the case. 2026 had the biggest difference between pre and post deadline since 2002, with a 10% higher blowout rate and an average MoV 2 points higher. Throughout the 25 years of data, most years have a higher blowout rate after the trade deadline, but not as extreme as this season.</p>
    </div>

    <!-- Playoff vs non-playoff monthly win rate -->
    <div class="chart-block wp-block">
        <div class="chart-header">
            <h2>Monthly Win % — Playoff vs. Non-Playoff Teams</h2>
            <select value={pvoSeason} onchange={e => pvoSeason = +e.currentTarget.value}>
                {#each pvoSeasons as s}
                    <option value={s}>{seasonLabel(s)}</option>
                {/each}
            </select>
        </div>
        <svg width={PVO.w} height={PVO.h}>
            <g transform="translate({PVO.ml},{PVO.mt})">
                {#each pvoYScale.ticks(4) as tick}
                    <line x1={0} y1={pvoYScale(tick)} x2={Pvoiw} y2={pvoYScale(tick)} stroke="#ebebeb" stroke-width="1" />
                    <text x={-5} y={pvoYScale(tick) + 4} text-anchor="end" font-size="10" fill="#999">{(tick * 100).toFixed(0)}%</text>
                {/each}

                {#each PVO_MONTH_LBLS as lbl}
                    <line x1={pvoXScale(lbl)} y1={Pvoih} x2={pvoXScale(lbl)} y2={Pvoih + 5} stroke="#ccc" stroke-width="1" />
                    <text x={pvoXScale(lbl)} y={Pvoih + 15} text-anchor="middle" font-size="10" fill="#999">{lbl}</text>
                {/each}

                <path d={pvoAvgNonPlayoffPath} fill="none" stroke="#e74c3c" stroke-width="1.5" opacity="0.25" />
                <path d={pvoAvgPlayoffPath}    fill="none" stroke="#2980b9" stroke-width="1.5" opacity="0.25" />
                <path d={pvoNonPlayoffPath} fill="none" stroke="#e74c3c" stroke-width="1.5" />
                <path d={pvoPlayoffPath}    fill="none" stroke="#2980b9" stroke-width="1.5" />

                {#each pvoLines.playoff as { label, value }}
                    <circle cx={pvoXScale(label)} cy={pvoYScale(value)} r="3" fill="#2980b9" stroke="white" stroke-width="1" />
                {/each}
                {#each pvoLines.nonPlayoff as { label, value }}
                    <circle cx={pvoXScale(label)} cy={pvoYScale(value)} r="3" fill="#e74c3c" stroke="white" stroke-width="1" />
                {/each}

                <line x1={0} y1={0} x2={0} y2={Pvoih} stroke="#ccc" />
                <line x1={0} y1={Pvoih} x2={Pvoiw} y2={Pvoih} stroke="#ccc" />
            </g>
        </svg>
        <div class="wp-legend">
            <div class="wp-legend-item">
                <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#2980b9" stroke-width="1.5"/></svg>
                <span>Playoff teams</span>
            </div>
            <div class="wp-legend-item">
                <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#e74c3c" stroke-width="1.5"/></svg>
                <span>Non-playoff teams</span>
            </div>
            <div class="wp-legend-item">
                <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#2980b9" stroke-width="1.5" opacity="0.25"/></svg>
                <span>Playoff avg (2002–2026)</span>
            </div>
            <div class="wp-legend-item">
                <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#e74c3c" stroke-width="1.5" opacity="0.25"/></svg>
                <span>Non-playoff avg (2002–2026)</span>
            </div>
        </div>
        <p class="caption">Showing win percentages from games involving 1 playoff team and 1 non-playoff team. A playoff team is defined as a team in the playoffs at the <em>end</em> of the season, including play-in teams.</p>
    </div>

    {#if tooltip}
        <div class="tooltip" style="left: {tooltip.x}px; top: {tooltip.y}px">
            <strong>{tooltip.data.date.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}</strong>
            <span>{tooltip.data.games_played} games</span>
            <span>Avg MoV: {tooltip.data.avg_mov.toFixed(1)}</span>
        </div>
    {/if}

    <div class="article-text">
        <p>Next, we’ll look at how playoff and non-playoff teams perform against each other throughout the course of a season. We’d expect that, during the first few months of the season, both groups are still trying to win games as the playoff race is undecided, and their averaged win percentages would be relatively consistent. Then, perhaps around the trade deadline in February, non-playoff teams might start losing even more games as they start to tank because the playoffs are no longer in reach. Over the last 25 years, non-playoff teams lose 10% more games by the end of the season in April than they do at the beginning in October. However, in the 2025-2026 season, non-playoff teams lost 20% more games between the beginning and end of the year, showcasing the increase in tanking in the NBA this year. In April of this year, non-playoff teams won less than 10% of games against playoff teams. No fan will want to watch these games between non-playoff teams and playoff teams if they already know the result!</p>
    </div>

    <div class="article-text">
        <p>Another aspect of tanking that decreases enjoyment for fans is the increase in minutes for unknown players. While this may be enjoyable for the avid observer, most fans want to see the best players play. We’ll show this increase through a metric called “Outlier Player-Minutes”. For a given game, if a player plays more than 20 minutes in that game while having a pre-trade deadline minutes average of 5 or less, then it is considered an outlier. The idea is that after the trade deadline, teams who are tanking start to play players who have previously not played much, if at all, in hopes of losing. Shown below are two plots showing the prevalence of these situations.</p>
    </div>

    <!-- Cumulative outlier minutes by season -->
    <div class="chart-block diff-block fit-block">
        <div class="chart-header">
            <h2>Cumulative Outlier Player-Minutes by Season</h2>
        </div>
        <MinutesOutliersChart data={minDat} />
    </div>

    <!-- Outlier minutes vs. team rank, by season -->
    <div class="chart-block diff-block fit-block">
        <div class="chart-header">
            <h2>Outlier Player-Minutes by Team Rank</h2>
        </div>
        <LogoScatterChart data={logoDat} />
    </div>

    <div class="article-text">
        <p>In the line plot, we observe that the 2025-2026 season had the highest amount of OPM at over 500 instances, more than double that of the 2021-2022 and 2022-2023 seasons. This suggests that a new strategy of tanking teams is to play big minutes to previously unknown players to increase their chances of losing. Yes, these players are often young and so part of the strategy is to give them chances to develop, but it is almost certainly not the most competitive group of players that a team could field. Looking at the second plot, the Wizards have been a big abuser of this strategy in the last few years. And even further back, in 2021-2022 the Thunder utilized this strategy far more than any team that year. They are the ones generally credited with the development of this strategy, using 2-way and 10 day contract players proficiently. </p>
    </div>

    <div class="article-text">
        <p>The NBA has a tanking problem, and hopefully the new rule changes in the draft lottery will start to dissuade teams from tanking. As I’ve shown, the gameplay of the NBA last year was significantly impacted by tanking, and the experience of fans was worse because of it. There were more blowouts, less competitive games, and more random players. Will next year be any different? Adam Silver and the league office certainly hope so. </p>
    </div>
</main>

<style>
    main {
        max-width: 900px;
        margin: 2.5rem auto;
        font-family: sans-serif;
        padding: 0 6rem;
    }

    .article-text {
        max-width: 580px;
        margin: 0 auto 1.75rem;
    }

    h1 {
        font-size: 1.4rem;
        margin-bottom: 0.6rem;
    }

    .article-text p {
        font-size: 0.88rem;
        line-height: 1.7;
        color: #333;
    }

    h2 {
        font-size: 0.8rem;
        font-weight: 600;
        color: #444;
        margin: 0;
    }

    .chart-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        margin-bottom: 0.5rem;
    }

    .toggle {
        display: flex;
        border: 1px solid #ddd;
        border-radius: 4px;
        overflow: hidden;
        flex-shrink: 0;
    }

    .toggle button {
        padding: 0.18rem 0.5rem;
        font-size: 0.72rem;
        border: none;
        background: white;
        cursor: pointer;
        color: #666;
    }

    .toggle button:not(:last-child) {
        border-right: 1px solid #ddd;
    }

    .toggle button.active {
        background: #2c3e50;
        color: white;
    }

    .layout {
        display: grid;
        grid-template-columns: 1fr 305px;
        gap: 1.5rem;
        align-items: start;
    }

    /* ── Calendar ── */

    .controls {
        display: flex;
        align-items: center;
        gap: 1rem;
        margin-bottom: 0.75rem;
    }

    select {
        padding: 0.2rem 0.45rem;
        font-size: 0.85rem;
        border: 1px solid #ccc;
        border-radius: 4px;
    }

    .month-nav {
        display: flex;
        align-items: center;
        gap: 0.4rem;
    }

    .month-nav button {
        padding: 0.2rem 0.6rem;
        font-size: 0.85rem;
        cursor: pointer;
        border: 1px solid #ccc;
        border-radius: 4px;
        background: white;
    }

    .month-nav button:hover { background: #f0f0f0; }

    .month-label {
        font-size: 0.92rem;
        font-weight: 600;
        min-width: 90px;
        text-align: center;
    }

    .calendar {
        border: 1px solid #ddd;
        border-radius: 6px;
        overflow: hidden;
    }

    .day-headers {
        display: grid;
        grid-template-columns: repeat(7, 1fr);
        background: #2c3e50;
        color: white;
    }

    .day-header {
        text-align: center;
        padding: 0.35rem 0;
        font-size: 0.7rem;
        font-weight: 600;
        letter-spacing: 0.05em;
    }

    .week {
        display: grid;
        grid-template-columns: repeat(7, 1fr);
    }

    .cell {
        min-height: 50px;
        padding: 0.28rem 0.32rem;
        border-right: 1px solid rgba(0,0,0,0.07);
        border-bottom: 1px solid rgba(0,0,0,0.07);
        display: flex;
        flex-direction: column;
    }

    .cell.empty { background: #fafafa; }

    .day-num {
        font-size: 0.68rem;
        font-weight: 900;
        color: #000;
    }

    .legend {
        display: flex;
        align-items: center;
        gap: 0.6rem;
        margin-top: 0.75rem;
        font-size: 0.72rem;
        color: #555;
    }

    .legend-bar {
        display: flex;
        border-radius: 3px;
        overflow: hidden;
    }

    .legend-swatch {
        width: 18px;
        height: 11px;
    }

    .caption {
        font-size: 0.7rem;
        color: #888;
        margin-top: 0.4rem;
        line-height: 1.5;
    }

    /* ── Charts ── */

    .charts-panel {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .chart-block {
        background: #fafafa;
        border: 1px solid #e8e8e8;
        border-radius: 6px;
        padding: 0.6rem 0.6rem 0.4rem;
    }

    .diff-block, .wp-block {
        margin-top: 1rem;
    }

    .fit-block {
        width: fit-content;
    }

    .wp-legend {
        display: flex;
        gap: 0.75rem;
        font-size: 0.68rem;
        color: #666;
        margin-top: 0.2rem;
        padding-left: 45px;
    }

    .wp-legend-item {
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }

    /* ── Tooltip ── */

    .tooltip {
        position: fixed;
        transform: translate(-50%, -100%);
        background: rgba(0, 0, 0, 0.82);
        color: white;
        padding: 0.4rem 0.6rem;
        border-radius: 5px;
        font-size: 0.78rem;
        pointer-events: none;
        display: flex;
        flex-direction: column;
        gap: 2px;
        white-space: nowrap;
        z-index: 10;
    }
</style>
