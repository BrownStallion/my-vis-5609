<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";
  // define the props of the Q1 component
  type Props = {
    movies: TMovie[];
    width?: number;
    height?: number;
  };
  let { movies, width = 900, height = 520 }: Props = $props();

  type RankValue = { year: number; rank?: number; count?: number };
  type Series = { genre: string; values: RankValue[] };

  let selectedGenre: string = $state("");

  // processing the data; $derived is used to create a reactive variable that updates whenever the dependent variables change
  // one column per (year, genre); "NA"/empty genres and invalid years are skipped
  // getUTCFullYear() is used because new Date("1946") is parsed as UTC midnight
  const genreYears = $derived(
    movies
      .filter((movie) => !isNaN(movie.year.getTime()))
      .flatMap((movie) =>
        movie.genres
          .filter((genre: string) => genre !== "" && genre !== "NA")
          .map((genre: string) => ({
            year: movie.year.getUTCFullYear(),
            genre,
          }))
      )
  );

  const yearGenreCounts = $derived(
    d3.rollup(
      genreYears,
      (v) => v.length, // number of movies with given genre key
      (d) => d.year, // year data
      (d) => d.genre // genre data
    )
  );

  // total number of movies for a genre (used only to break ties)
  const genreTotals = $derived(
    d3.rollup(
      genreYears,
      (v) => v.length,
      (d) => d.genre
    )
  );

  const genres = $derived(
    Array.from(genreTotals.keys()).sort((a, b) => a.localeCompare(b))
  );

  // every year between the first and last release year
  const years = $derived.by(() => {
    const keys = Array.from(yearGenreCounts.keys());
    if (keys.length === 0) return [] as number[];
    const first = Math.min(...keys);
    const last = Math.max(...keys);
    return Array.from({ length: last - first + 1 }, (_, i) => first + i);
  });

  // for each year, map each genre based on rank and count; rank 1 means most movies released that year
  // for ties, higher all-time total first, then alphabetical, so ranks are unique
  const ranksByYear = $derived.by(() => {
    const res = new Map<number, Map<string, { rank: number; count: number }>>();
    yearGenreCounts.forEach((genreCounts, year) => {
      const sorted = Array.from(genreCounts, ([genre, count]) => ({
        genre,
        count,
      })).sort(
        (a, b) =>
          b.count - a.count ||
          (genreTotals.get(b.genre) ?? 0) - (genreTotals.get(a.genre) ?? 0) ||
          a.genre.localeCompare(b.genre)
      );
      res.set(
        year,
        new Map(
          sorted.map((d, i) => [d.genre, { rank: i + 1, count: d.count }])
        )
      );
    });
    return res;
  });

  // one line per genre; rank is undefined in years where the genre has no movies
  const series: Series[] = $derived(
    genres.map((genre) => ({
      genre,
      values: years.map((year) => {
        const r = ranksByYear.get(year)?.get(genre);
        return { year, rank: r?.rank, count: r?.count };
      }),
    }))
  );

  const maxRank = $derived(
    Math.max(1, ...Array.from(ranksByYear.values(), (m) => m.size))
  );
  const ranks = $derived(Array.from({ length: maxRank }, (_, i) => i + 1));

  // drawing the bump chart

  const margin = {
    top: 15,
    bottom: 40,
    left: 55,
    right: 150, // extra room for the legend
  };

  let usableArea = $derived({
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  });

  const xScale = $derived(
    // tip: use d3.scaleBand() to create a band scale for the x-axis
    d3
      .scaleLinear()
      .range([usableArea.left, usableArea.right])
      .domain([years[0] ?? 0, years[years.length - 1] ?? 1])
  );

  const yScale = $derived(
    // tip: use d3.scaleLinear() to create a linear scale for the y-axis
    d3
      .scaleLinear()
      .range([usableArea.top, usableArea.bottom])
      .domain([1, maxRank])
  );

  // one unique color per genre; n + 1 samples are taken and the last is dropped
  // (t = 0 and t = 1 are the same color)
  const palette = $derived(
    genres.length > 0
      ? d3
          .quantize((t) => d3.interpolateSinebow(t), genres.length + 1)
          .slice(0, genres.length)
      : []
  );
  const colorScale = $derived(d3.scaleOrdinal<string, string>(genres, palette));

  // segments end wherever a genre has no movies in a year (rank is undefined)
  const lineGenerator = $derived(
    d3
      .line<RankValue>()
      .defined((d) => d.rank !== undefined)
      .x((d) => xScale(d.year))
      .y((d) => yScale(d.rank!))
  );

  let xAxis: any = $state(),
    yAxis: any = $state();

  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale)
      .tickFormat(d3.format("d")));
    
    // tip:
    // similar to the x-axis, create a y-axis using d3.axisLeft() and bind it to the yAxis variable

    d3.select(yAxis)
      .call(d3.axisLeft(yScale)
      .tickValues(ranks)
      .tickFormat(d3.format("d"))
    );
  }

  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  Ranking of Genres by Movies Released per Year {years[0]} - {years[years.length - 1]} - Bump Chart
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <g class="grid">
      {#each ranks as r}
        <line
          x1={usableArea.left}
          x2={usableArea.right}
          y1={yScale(r)}
          y2={yScale(r)}
          stroke="#ddd"
          stroke-dasharray="3,3"
        />
      {/each}
    </g>

    <g class="lines">
      {#each series as s (s.genre)}
        <g
          class="series"
          opacity={selectedGenre && selectedGenre !== s.genre ? 0.15 : 1}
          onmouseover={() => {
            selectedGenre = s.genre;
          }}
          onmouseout={() => {
            selectedGenre = "";
          }}
        >
          <path
            d={lineGenerator(s.values) ?? ""}
            fill="none"
            stroke={colorScale(s.genre)}
            stroke-width="2.5"
          />
          {#each s.values.filter((v) => v.rank !== undefined) as v}
            <circle
              cx={xScale(v.year)}
              cy={yScale(v.rank!)}
              r="3.5"
              fill={colorScale(s.genre)}
            >
              <title>{s.genre}, {v.year}: {v.count} movies (rank {v.rank})</title>
            </circle>
          {/each}
        </g>
      {/each}
    </g>

    <g class="legend" transform="translate({usableArea.right + 20}, {usableArea.top})">
      <text font-size="12" font-weight="bold" y="-3">Genre</text>
      {#each genres as genre, i}
        <g
          transform="translate(0, {10 + i * 17})"
          opacity={selectedGenre && selectedGenre !== genre ? 0.25 : 1}
          onmouseover={() => {
            selectedGenre = genre;
          }}
          onmouseout={() => {
            selectedGenre = "";
          }}
        >
          <line
            x1="0"
            x2="18"
            y1="0"
            y2="0"
            stroke={colorScale(genre)}
            stroke-width="3"
          />
          <circle cx="9" cy="0" r="3.5" fill={colorScale(genre)} />
          <text x="24" y="4" font-size="12">{genre}</text>
        </g>
      {/each}
    </g>

    <text
      transform="rotate(-90)"
      x={-(usableArea.top + usableArea.bottom) / 2}
      y="14"
      font-size="12"
      text-anchor="middle"
    >
      Rank (1 = Most Movies Released)
    </text>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis}></g>
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis}></g>
  </svg>
{/if}

<style>
  .series {
    transition: opacity 0.1s ease; /* Smooth transition for hover dimming */
  }
</style>