<script lang="ts">
    import * as d3 from "d3";
    import type { TMovie } from "../types";

    type TGenreSummary = {
        genre: string;
        averageRating: number;
        count: number;
    };

    type TProps = {
        movies: TMovie[];
        selectedGenre: string;
        width?: number;
        height?: number;
    };

    let {
        movies,
        selectedGenre = $bindable(),
        width = 700,
        height = 420,
    }: TProps = $props();

    const margin = { top: 20, right: 20, bottom: 100, left: 55 };
    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    const summaries = $derived.by(() => {
        const genreRatings = new Map<string, number[]>();

        for (const movie of movies) {
            for (const genre of movie.genres) {
                const ratings = genreRatings.get(genre) ?? [];
                ratings.push(movie.average_rating);
                genreRatings.set(genre, ratings);
            }
        }

        return [...genreRatings.entries()]
            .map(([genre, ratings]) => ({
                genre,
                averageRating: d3.mean(ratings) ?? 0,
                count: ratings.length,
            }))
            .sort((a, b) => d3.descending(a.averageRating, b.averageRating));
    });

    const xScale = $derived(
        d3
            .scaleBand<string>()
            .domain(summaries.map((summary) => summary.genre))
            .range([0, innerWidth])
            .padding(0.15),
    );

    const yScale = $derived(
        d3
            .scaleLinear()
            .domain([0, 10])
            .nice()
            .range([innerHeight, 0]),
    );

    function selectGenre(genre: string) {
        selectedGenre = genre;
    }
</script>

<svg {width} {height} aria-label="Average rating by genre">
    <g transform={`translate(${margin.left}, ${margin.top})`}>
        {#each summaries as summary (summary.genre)}
            <g>
                <!-- svelte-ignore a11y_click_events_have_key_events -->
                <!-- svelte-ignore a11y_no_static_element_interactions -->
                <rect
                    x={xScale(summary.genre)}
                    y={yScale(summary.averageRating)}
                    width={xScale.bandwidth()}
                    height={innerHeight - yScale(summary.averageRating)}
                    fill={summary.genre === selectedGenre
                        ? "#d95f02"
                        : "#4c78a8"}
                    opacity={summary.genre === selectedGenre ? 1 : 0.75}
                    onclick={() => selectGenre(summary.genre)}
                >
                    <title>
                        {summary.genre}: {summary.averageRating.toFixed(2)} average rating ({summary.count} movies)
                    </title>
                </rect>
                <text
                    transform={`translate(${(xScale(summary.genre) ?? 0) + xScale.bandwidth() / 2}, ${innerHeight + 15}) rotate(45)`}
                    text-anchor="start"
                    font-size="11"
                >
                    {summary.genre}
                </text>
            </g>
        {/each}

        <line x1="0" x2={innerWidth} y1={innerHeight} y2={innerHeight} stroke="currentColor" />
        <line x1="0" x2="0" y1="0" y2={innerHeight} stroke="currentColor" />
        {#each yScale.ticks(5) as tick}
            <g transform={`translate(0, ${yScale(tick)})`}>
                <line x2={innerWidth} stroke="#ddd" />
                <text x="-8" dy="0.32em" text-anchor="end" font-size="11">
                    {tick}
                </text>
            </g>
        {/each}
        <text transform={`translate(${-42}, ${innerHeight / 2}) rotate(-90)`} text-anchor="middle">
            Average rating
        </text>
    </g>
</svg>
