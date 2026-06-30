<script>
    import * as d3 from 'd3';

    let { data = [] } = $props();

    const W = 630, H = 340, MT = 10, MR = 8, MB = 30, ML = 40;
    const iw = W - ML - MR;
    const ih = H - MT - MB;
    const LOGO_SIZE = 26;
    const PLAYOFF_COLOR = '#2980b9';
    const NON_PLAYOFF_COLOR = '#e74c3c';

    function slabel(s) { return `${s - 1}–${s}`; }

    let seasons = $derived(
        [...new Set(data.map(d => d.season))].sort((a, b) => a - b)
    );

    let selectedSeason = $state(null);

    $effect(() => {
        if (selectedSeason === null && seasons.length) {
            selectedSeason = seasons[seasons.length - 1];
        }
    });

    let seasonData = $derived(
        data.filter(d => d.season === selectedSeason).sort((a, b) => a.rank - b.rank)
    );

    // Stable y domain across all seasons so switching years doesn't rescale the axis
    let maxVal = $derived(
        data.length ? d3.max(data, d => d.total_min_outliers) * 1.1 : 50
    );

    let xScale = $derived(
        d3.scalePoint().domain(d3.range(1, 31)).range([0, iw]).padding(0.5)
    );

    let yScale = $derived(
        d3.scaleLinear().domain([0, maxVal]).range([ih, 0]).nice()
    );
</script>

<div class="controls">
    <label for="logo-scatter-season">Season</label>
    <select id="logo-scatter-season" value={selectedSeason} onchange={e => selectedSeason = +e.currentTarget.value}>
        {#each seasons as s}
            <option value={s}>{slabel(s)}</option>
        {/each}
    </select>
</div>

<svg width={W} height={H}>
    <g transform="translate({ML},{MT})">
        {#each yScale.ticks(5) as tick}
            <line x1={0} y1={yScale(tick)} x2={iw} y2={yScale(tick)} stroke="#ebebeb" stroke-width="1" />
            <text x={-5} y={yScale(tick) + 4} text-anchor="end" font-size="10" fill="#999">{tick}</text>
        {/each}

        {#each d3.range(1, 31) as r}
            {#if r % 2 === 1 || r === 30}
                <text x={xScale(r)} y={ih + 16} text-anchor="middle" font-size="9" fill="#999">{r}</text>
            {/if}
        {/each}

        {#each seasonData as team}
            {@const cx = xScale(team.rank)}
            {@const cy = yScale(team.total_min_outliers)}
            <circle
                cx={cx} cy={cy}
                r={LOGO_SIZE / 2 + 2}
                fill="white"
                stroke={team.playoff_team ? PLAYOFF_COLOR : NON_PLAYOFF_COLOR}
                stroke-width="1.5"
            />
            <image
                href={team.logo2}
                x={cx - LOGO_SIZE / 2}
                y={cy - LOGO_SIZE / 2}
                width={LOGO_SIZE}
                height={LOGO_SIZE}
            >
                <title>{team.team_display_name}: {team.total_min_outliers} outlier minutes (rank {team.rank})</title>
            </image>
        {/each}

        <line x1={0} y1={0} x2={0} y2={ih} stroke="#ccc" />
        <line x1={0} y1={ih} x2={iw} y2={ih} stroke="#ccc" />
        <text x={iw / 2} y={ih + 26} text-anchor="middle" font-size="10" fill="#aaa">Rank (1 = best record)</text>
    </g>
</svg>

<div class="legend">
    <div class="legend-item">
        <span class="legend-ring" style="border-color: {PLAYOFF_COLOR}"></span>
        <span>Playoff team</span>
    </div>
    <div class="legend-item">
        <span class="legend-ring" style="border-color: {NON_PLAYOFF_COLOR}"></span>
        <span>Non-playoff team</span>
    </div>
</div>

<style>
    .controls {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        margin-bottom: 0.4rem;
    }

    label {
        font-size: 0.78rem;
        color: #555;
    }

    select {
        padding: 0.2rem 0.45rem;
        font-size: 0.85rem;
        border: 1px solid #ccc;
        border-radius: 4px;
    }

    .legend {
        display: flex;
        gap: 0.75rem;
        font-size: 0.68rem;
        color: #666;
        margin-top: 0.3rem;
        padding-left: 40px;
    }

    .legend-item {
        display: flex;
        align-items: center;
        gap: 0.35rem;
    }

    .legend-ring {
        width: 12px;
        height: 12px;
        border-radius: 50%;
        border: 1.5px solid;
        box-sizing: border-box;
        background: white;
    }
</style>
