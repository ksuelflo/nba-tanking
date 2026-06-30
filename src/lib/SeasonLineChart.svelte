<script>
    import * as d3 from 'd3';

    let {
        data = [],
        preData = null,
        postData = null,
        selectedSeason = null,
        onClickSeason = () => {},
        showZeroLine = false,
        yFloor = 0,
        formatYTick = v => v.toFixed(1),
        mainColor = '#bbb',
        w = 285,
        h = 150,
        mt = 6,
        mr = 14,
        mb = 22,
        ml = 40,
        showLegend = true,
    } = $props();

    let hoveredSeason = $state(null);

    let iw = $derived(w - ml - mr);
    let ih = $derived(h - mt - mb);

    let xScale = $derived(
        data.length > 0
            ? d3.scaleLinear().domain(d3.extent(data, d => d.season)).range([0, iw])
            : d3.scaleLinear().domain([2002, 2026]).range([0, iw])
    );

    let xTicks = $derived(
        data.length > 0
            ? d3.range(
                Math.ceil(data[0].season / 5) * 5,
                data[data.length - 1].season + 1,
                5
              )
            : []
    );

    let yScale = $derived.by(() => {
        const allVals = [
            ...data.map(d => d.value),
            ...(preData ?? []).map(d => d.value),
            ...(postData ?? []).map(d => d.value),
        ].filter(v => v != null);
        const maxVal = allVals.length > 0 ? d3.max(allVals) * 1.15 : 15;
        const minRaw = allVals.length > 0 ? d3.min(allVals) : 0;
        const minVal = yFloor !== null ? yFloor : Math.min(0, minRaw) * 1.15;
        return d3.scaleLinear().domain([minVal, maxVal]).range([ih, 0]);
    });

    let mainPath = $derived(
        data.length > 0
            ? d3.line()
                .defined(d => d.value != null)
                .x(d => xScale(d.season))
                .y(d => yScale(d.value))
                (data)
            : ''
    );

    let prePath = $derived(
        preData && preData.length > 0
            ? d3.line()
                .defined(d => d.value != null)
                .x(d => xScale(d.season))
                .y(d => yScale(d.value))
                (preData)
            : ''
    );

    let postPath = $derived(
        postData && postData.length > 0
            ? d3.line()
                .defined(d => d.value != null)
                .x(d => xScale(d.season))
                .y(d => yScale(d.value))
                (postData)
            : ''
    );

    let hoverPre  = $derived(hoveredSeason != null && preData  ? (preData.find(d => d.season === hoveredSeason) ?? null) : null);
    let hoverPost = $derived(hoveredSeason != null && postData ? (postData.find(d => d.season === hoveredSeason) ?? null) : null);
    let hoverDiff = $derived(hoverPre && hoverPost ? hoverPost.value - hoverPre.value : null);

    function slabel(s) { return `${s - 1}–${s}`; }

    function handleMouseMove(event) {
        if (!data.length) return;
        const rect = event.currentTarget.getBoundingClientRect();
        const mouseX = event.clientX - rect.left - ml;
        if (mouseX < -20 || mouseX > iw + 20) { hoveredSeason = null; return; }
        const clamped = Math.max(0, Math.min(iw, mouseX));
        const seasonGuess = xScale.invert(clamped);
        hoveredSeason = data.reduce((best, d) =>
            Math.abs(d.season - seasonGuess) < Math.abs(best.season - seasonGuess) ? d : best
        , data[0]).season;
    }

    function handleMouseLeave() {
        hoveredSeason = null;
    }
</script>

<svg width={w} height={h}
     onmousemove={handleMouseMove}
     onmouseleave={handleMouseLeave}
     style="display: block"
>
    <!-- Stats annotation in top margin (only for pre/post charts) -->
    {#if preData && hoveredSeason != null}
        <g transform="translate({ml}, 2)">
            <text x={0} y={14} font-size="10" font-weight="700" fill="#333">{slabel(hoveredSeason)}</text>
            <text y={29} font-size="9.5">
                {#if hoverPre}
                    <tspan x={0} fill="#3498db">Pre: {formatYTick(hoverPre.value)}</tspan>
                {/if}
                {#if hoverPost}
                    <tspan dx="10" fill="#e67e22">Post: {formatYTick(hoverPost.value)}</tspan>
                {/if}
                {#if hoverDiff != null}
                    <tspan dx="10" fill={hoverDiff > 0 ? '#c0392b' : '#27ae60'}>Δ {hoverDiff > 0 ? '+' : ''}{formatYTick(hoverDiff)}</tspan>
                {/if}
            </text>
        </g>
    {/if}

    <g transform="translate({ml},{mt})">
        {#each yScale.ticks(4) as tick}
            <line x1={0} y1={yScale(tick)} x2={iw} y2={yScale(tick)} stroke="#ebebeb" stroke-width="1" />
            <text x={-5} y={yScale(tick) + 4} text-anchor="end" font-size="10" fill="#999">
                {formatYTick(tick)}
            </text>
        {/each}

        {#each xTicks as tick}
            <line x1={xScale(tick)} y1={ih} x2={xScale(tick)} y2={ih + 5} stroke="#ccc" stroke-width="1" />
            <text x={xScale(tick)} y={ih + 15} text-anchor="middle" font-size="10" fill="#999">{tick}</text>
        {/each}

        {#if showZeroLine}
            {@const zy = yScale(0)}
            {#if zy >= 0 && zy <= ih}
                <line x1={0} y1={zy} x2={iw} y2={zy} stroke="#aaa" stroke-width="1" stroke-dasharray="3,3" />
            {/if}
        {/if}

        {#if prePath}
            <path d={prePath} fill="none" stroke="#3498db" stroke-width="1.5" stroke-dasharray="4,3" />
        {/if}
        {#if postPath}
            <path d={postPath} fill="none" stroke="#e67e22" stroke-width="1.5" stroke-dasharray="4,3" />
        {/if}
        <path d={mainPath} fill="none" stroke={mainColor} stroke-width="1.5" />

        <!-- Vertical hover indicator -->
        {#if hoveredSeason != null}
            <line
                x1={xScale(hoveredSeason)} y1={0}
                x2={xScale(hoveredSeason)} y2={ih}
                stroke="#888" stroke-width="1" stroke-dasharray="3,3"
                pointer-events="none"
            />
        {/if}

        {#each data as { season, value }}
            {#if value != null}
                {@const active = season === selectedSeason}
                {@const hovered = season === hoveredSeason}
                <circle
                    cx={xScale(season)}
                    cy={yScale(value)}
                    r={active ? 5 : hovered ? 4 : 3}
                    fill={active ? '#c0392b' : hovered ? '#555' : mainColor}
                    stroke="white"
                    stroke-width="1"
                    role="button"
                    tabindex="0"
                    style="cursor: pointer"
                    onclick={() => onClickSeason(season)}
                    onkeydown={e => e.key === 'Enter' && onClickSeason(season)}
                />
            {/if}
        {/each}

        <!-- Highlighted dots on pre/post lines for hovered season -->
        {#if hoverPre}
            <circle cx={xScale(hoverPre.season)} cy={yScale(hoverPre.value)} r="4" fill="#3498db" stroke="white" stroke-width="1.5" pointer-events="none" />
        {/if}
        {#if hoverPost}
            <circle cx={xScale(hoverPost.season)} cy={yScale(hoverPost.value)} r="4" fill="#e67e22" stroke="white" stroke-width="1.5" pointer-events="none" />
        {/if}

        <line x1={0} y1={0} x2={0} y2={ih} stroke="#ccc" />
        <line x1={0} y1={ih} x2={iw} y2={ih} stroke="#ccc" />
    </g>
</svg>

{#if showLegend}
    <div class="line-legend">
        <div class="line-legend-item">
            <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke={mainColor} stroke-width="1.5"/></svg>
            <span>Season avg</span>
        </div>
        <div class="line-legend-item">
            <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#3498db" stroke-width="1.5" stroke-dasharray="4,3"/></svg>
            <span>Pre-trade deadline</span>
        </div>
        <div class="line-legend-item">
            <svg width="18" height="10"><line x1="0" y1="5" x2="18" y2="5" stroke="#e67e22" stroke-width="1.5" stroke-dasharray="4,3"/></svg>
            <span>Post-trade deadline</span>
        </div>
    </div>
{/if}

<style>
    .line-legend {
        display: flex;
        gap: 0.75rem;
        font-size: 0.68rem;
        color: #666;
        margin-top: 0.2rem;
        padding-left: 40px;
    }

    .line-legend-item {
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }
</style>
