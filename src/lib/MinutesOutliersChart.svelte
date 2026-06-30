<script>
    import * as d3 from 'd3';

    let { data = [] } = $props();

    const W = 630, H = 200, MT = 10, MR = 8, MB = 20, ML = 42;
    const iw = W - ML - MR;
    const ih = H - MT - MB;

    const MONTH_TICKS = [
        { day: 0,   label: 'Oct' },
        { day: 31,  label: 'Nov' },
        { day: 61,  label: 'Dec' },
        { day: 92,  label: 'Jan' },
        { day: 123, label: 'Feb' },
        { day: 151, label: 'Mar' },
        { day: 181, label: 'Apr' },
        { day: 211, label: 'May' },
    ];

    const SEASON_COLORS = {
        2022: '#5b9ec9',
        2023: '#2471a3',
        2024: '#fdcb6e',
        2025: '#e17055',
        2026: '#c0392b',
    };

    function dayOfSeason(date) {
        const m = date.getMonth();
        const yr = m >= 9 ? date.getFullYear() : date.getFullYear() - 1;
        return Math.floor((date - new Date(yr, 9, 1)) / 86400000);
    }

    function seasonEndYear(date) {
        return date.getMonth() >= 9 ? date.getFullYear() + 1 : date.getFullYear();
    }

    // Convert a day-of-season number back to a "Month Day" label using a fixed reference year
    function dayToLabel(day) {
        const ref = new Date(2023, 9, 1); // Oct 1 as stable reference
        return new Date(ref.getTime() + day * 86400000)
            .toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
    }

    let bySeason = $derived.by(() => {
        if (!data.length) return new Map();
        const map = new Map();
        for (const row of data) {
            const s = seasonEndYear(row.game_date);
            if (s === 2021) continue; // exclude COVID-shortened season
            const day = dayOfSeason(row.game_date);
            if (day > 211) continue; // cut off at May 1
            if (!map.has(s)) map.set(s, []);
            map.get(s).push({ day, val: row.total_min_outliers });
        }
        for (const rows of map.values()) {
            rows.sort((a, b) => a.day - b.day);
            let cum = 0;
            for (const r of rows) { cum += r.val; r.running = cum; }
        }
        return map;
    });

    let seasons = $derived([...bySeason.keys()].sort((a, b) => a - b));

    let maxTotal = $derived(
        seasons.length ? d3.max(seasons, s => bySeason.get(s).at(-1)?.running ?? 0) : 100
    );

    const xScale = d3.scaleLinear().domain([0, 211]).range([0, iw]); // 211 = May 1

    let yScale = $derived(
        d3.scaleLinear().domain([0, maxTotal]).range([ih, 0]).nice()
    );

    let seasonPaths = $derived.by(() => {
        if (!bySeason.size) return [];
        const area = d3.area()
            .x(d => xScale(d.day))
            .y0(ih)
            .y1(d => yScale(d.running))
            .curve(d3.curveMonotoneX);
        const line = d3.line()
            .x(d => xScale(d.day))
            .y(d => yScale(d.running))
            .curve(d3.curveMonotoneX);
        return seasons.map(season => ({
            season,
            color: SEASON_COLORS[season] ?? '#aaa',
            area: area(bySeason.get(season)),
            line: line(bySeason.get(season)),
        }));
    });

    // ── Hover state ──────────────────────────────────────────────────────────

    let hoveredDay = $state(null);
    let tooltipX = $state(0);
    let tooltipY = $state(0);

    const bisect = d3.bisector(d => d.day).right;

    function getCumulativeAtDay(rows, targetDay) {
        if (!rows || !rows.length || targetDay < rows[0].day) return 0;
        const i = bisect(rows, targetDay);
        return rows[Math.max(0, i - 1)].running;
    }

    let hoveredValues = $derived.by(() => {
        if (hoveredDay === null || !bySeason.size) return [];
        return seasons.map(s => ({
            season: s,
            color: SEASON_COLORS[s] ?? '#aaa',
            value: getCumulativeAtDay(bySeason.get(s) ?? [], hoveredDay),
        }));
    });

    function handleMouseMove(event) {
        const rect = event.currentTarget.getBoundingClientRect();
        const mouseX = event.clientX - rect.left - ML;
        if (mouseX < 0 || mouseX > iw) { hoveredDay = null; return; }
        hoveredDay = Math.round(xScale.invert(mouseX));
        tooltipX = event.clientX;
        tooltipY = event.clientY;
    }

    function handleMouseLeave() {
        hoveredDay = null;
    }

    function slabel(s) { return `${s - 1}–${s}`; }
</script>

<svg width={W} height={H} onmousemove={handleMouseMove} onmouseleave={handleMouseLeave} style="display: block">
    <g transform="translate({ML},{MT})">
        {#each yScale.ticks(5) as tick}
            <line x1={0} y1={yScale(tick)} x2={iw} y2={yScale(tick)} stroke="#ebebeb" stroke-width="1" />
            <text x={-5} y={yScale(tick) + 4} text-anchor="end" font-size="10" fill="#999">{tick}</text>
        {/each}

        {#each MONTH_TICKS as { day, label }}
            <line x1={xScale(day)} y1={ih} x2={xScale(day)} y2={ih + 5} stroke="#ccc" stroke-width="1" />
            <text x={xScale(day)} y={ih + 15} text-anchor="middle" font-size="10" fill="#999">{label}</text>
        {/each}

        <!-- Areas first so lines render on top -->
        {#each seasonPaths as { color, area }}
            <path d={area} fill={color} opacity="0.18" />
        {/each}
        {#each seasonPaths as { color, line }}
            <path d={line} fill="none" stroke={color} stroke-width="1.8" />
        {/each}

        <!-- Vertical hover line -->
        {#if hoveredDay !== null}
            <line
                x1={xScale(hoveredDay)} y1={0}
                x2={xScale(hoveredDay)} y2={ih}
                stroke="#555" stroke-width="1" stroke-dasharray="3,3"
                pointer-events="none"
            />
        {/if}

        <line x1={0} y1={0} x2={0} y2={ih} stroke="#ccc" />
        <line x1={0} y1={ih} x2={iw} y2={ih} stroke="#ccc" />
    </g>
</svg>

{#if hoveredDay !== null}
    <div class="hover-tooltip" style="left: {tooltipX + 14}px; top: {tooltipY}px">
        <strong>{dayToLabel(hoveredDay)}</strong>
        {#each hoveredValues as { season, color, value }}
            <div class="tooltip-row">
                <span class="tooltip-dot" style="background: {color}"></span>
                <span>{slabel(season)}: {value}</span>
            </div>
        {/each}
    </div>
{/if}

<div class="legend">
    {#each seasonPaths as { season, color }}
        <div class="legend-item">
            <svg width="18" height="10">
                <line x1="0" y1="5" x2="18" y2="5" stroke={color} stroke-width="1.8"/>
            </svg>
            <span>{slabel(season)}</span>
        </div>
    {/each}
</div>

<style>
    .legend {
        display: flex;
        gap: 0.75rem;
        font-size: 0.68rem;
        color: #666;
        margin-top: 0.2rem;
        padding-left: 48px;
        flex-wrap: wrap;
    }

    .legend-item {
        display: flex;
        align-items: center;
        gap: 0.3rem;
    }

    .hover-tooltip {
        position: fixed;
        transform: translateY(-50%);
        background: rgba(0, 0, 0, 0.82);
        color: white;
        padding: 0.4rem 0.6rem;
        border-radius: 5px;
        font-size: 0.75rem;
        pointer-events: none;
        display: flex;
        flex-direction: column;
        gap: 3px;
        white-space: nowrap;
        z-index: 10;
    }

    .tooltip-row {
        display: flex;
        align-items: center;
        gap: 5px;
    }

    .tooltip-dot {
        width: 8px;
        height: 8px;
        border-radius: 50%;
        flex-shrink: 0;
    }
</style>
