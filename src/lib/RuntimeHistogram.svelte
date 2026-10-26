<script lang="ts">
    import * as d3 from "d3";
    import type { TMovie } from "../types";

    type TProps = {
        movies: TMovie[];
        selectedGenre: string;
        width?: number;
        height?: number;
        binCount?: number;
    };

    type THistogramBin = {
        x0: number;
        x1: number;
        count: number;
    };

    let {
        movies,
        selectedGenre,
        width = 700,
        height = 380,
        binCount = 10,
    }: TProps = $props();

    const margin = { top: 20, right: 20, bottom: 55, left: 55 };
    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    const runtimeValues = $derived(
        movies
            .map((movie) => movie.runtime_minutes)
            .filter((runtime) => Number.isFinite(runtime)),
    );
    const selectedRuntimeValues = $derived(
        movies
            .filter((movie) => movie.genres.includes(selectedGenre))
            .map((movie) => movie.runtime_minutes)
            .filter((runtime) => Number.isFinite(runtime)),
    );

    const domain = $derived.by((): [number, number] => {
        if (runtimeValues.length === 0) {
            return [0, 1];
        }

        const minimum = d3.min(runtimeValues) ?? 0;
        const maximum = d3.max(runtimeValues) ?? 1;
        return minimum === maximum ? [minimum, maximum + 1] : [minimum, maximum];
    });

    const bins = $derived.by((): THistogramBin[] => {
        const [minimum, maximum] = domain;
        const step = (maximum - minimum) / binCount;

        return Array.from({ length: binCount }, (_, index) => {
            const x0 = minimum + index * step;
            const x1 = index === binCount - 1 ? maximum : x0 + step;
            const count = runtimeValues.filter((runtime) =>
                index === binCount - 1
                    ? runtime >= x0 && runtime <= x1
                    : runtime >= x0 && runtime < x1,
            ).length;
            return { x0, x1, count };
        });
    });

    const selectedCounts = $derived(
        bins.map((bin) =>
            selectedRuntimeValues.filter((runtime) =>
                bin === bins[bins.length - 1]
                    ? runtime >= bin.x0 && runtime <= bin.x1
                    : runtime >= bin.x0 && runtime < bin.x1,
            ).length,
        ),
    );

    const xScale = $derived(
        d3.scaleLinear().domain(domain).range([0, innerWidth]),
    );
    const yScale = $derived(
        d3
            .scaleLinear()
            .domain([0, d3.max(bins, (bin) => bin.count) ?? 1])
            .nice()
            .range([innerHeight, 0]),
    );

    function binWidth(bin: THistogramBin) {
        return Math.max(0, xScale(bin.x1 ?? domain[1]) - xScale(bin.x0 ?? domain[0]) - 1);
    }
</script>

<svg {width} {height} aria-label="Runtime distribution histogram">
    <g transform={`translate(${margin.left}, ${margin.top})`}>
        {#each bins as bin, index (bin.x0)}
            <rect
                x={xScale(bin.x0 ?? domain[0])}
                y={yScale(bin.count)}
                width={binWidth(bin)}
                height={innerHeight - yScale(bin.count)}
                fill="#bdbdbd"
                opacity="0.8"
            >
                    <title>{bin.count} movies</title>
            </rect>
            <rect
                x={xScale(bin.x0 ?? domain[0])}
                y={yScale(selectedCounts[index])}
                width={binWidth(bin)}
                height={innerHeight - yScale(selectedCounts[index])}
                fill="#e45756"
                opacity="0.8"
            >
                <title>{selectedCounts[index]} {selectedGenre} movies</title>
            </rect>
        {/each}

        <line x1="0" x2={innerWidth} y1={innerHeight} y2={innerHeight} stroke="currentColor" />
        <line x1="0" x2="0" y1="0" y2={innerHeight} stroke="currentColor" />
        {#each xScale.ticks(6) as tick}
            <g transform={`translate(${xScale(tick)}, ${innerHeight})`}>
                <line y2="5" stroke="currentColor" />
                <text y="18" text-anchor="middle" font-size="11">{tick}</text>
            </g>
        {/each}
        {#each yScale.ticks(5) as tick}
            <g transform={`translate(0, ${yScale(tick)})`}>
                <line x2={innerWidth} stroke="#ddd" />
                <text x="-8" dy="0.32em" text-anchor="end" font-size="11">{tick}</text>
            </g>
        {/each}
        <text x={innerWidth / 2} y={innerHeight + 43} text-anchor="middle">
            Runtime (minutes)
        </text>
        <text transform={`translate(${-42}, ${innerHeight / 2}) rotate(-90)`} text-anchor="middle">
            Number of movies
        </text>
    </g>
</svg>
