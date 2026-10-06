<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";
  // define the props of the Q2 component
  type Props = {
    movies: TMovie[];
    width?: number;
    height?: number;
  };
  let { movies, width = 600, height = 420 }: Props = $props();

  // the genre chosen in the dropdown
  let primaryGenre: string = $state("Comedy");
  // the genre currently hovered
  let selectedGenre: string = $state("");

  // processing the data; "NA"/empty genres are skipped
  const genres = $derived(
    Array.from(
      new Set(
        movies.flatMap((movie) =>
          movie.genres.filter((genre: string) => genre !== "" && genre !== "NA")
        )
      )
    ).sort((a, b) => a.localeCompare(b))
  );

  // number of movies tagged with the primary genre
  const primaryTotal = $derived(
    movies.filter((movie) => movie.genres.includes(primaryGenre)).length
  );

  // genre is number of movies that have both that genre and the primary genre
  // every other genre starts at 0 so the x-axis always shows all genres
  function getOverlapNums(movies: TMovie[], primaryGenre: string, genres: string[]) {
    let res: { [genre: string]: number } = {};
    genres
      .filter((genre) => genre !== primaryGenre)
      .forEach((genre) => (res[genre] = 0));
    movies
      .filter((movie) => movie.genres.includes(primaryGenre))
      .forEach((movie) => {
        movie.genres.forEach((genre: string) => {
          if (res[genre] !== undefined) res[genre] += 1;
        });
      });
    return res;
  }

  const overlapNums = $derived(getOverlapNums(movies, primaryGenre, genres));

  // sorted from most to least overlap; ties are alphabetical
  const overlapEntries = $derived(
    Object.entries(overlapNums).sort(
      (a, b) => b[1] - a[1] || a[0].localeCompare(b[0])
    )
  );

  // drawing the bar chart

  const margin = {
    top: 15,
    bottom: 70,
    left: 50,
    right: 10,
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
      .scaleBand()
      .range([usableArea.left, usableArea.right])
      .domain(overlapEntries.map(([genre]) => genre))
      .padding(0.15)
  );

  const yScale = $derived(
    // tip: use d3.scaleLinear() to create a linear scale for the y-axis
    d3
      .scaleLinear()
      .range([usableArea.bottom, usableArea.top])
      .domain([0, d3.max(Object.values(overlapNums)) ?? 0])
  );

  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: any = $state(),
    yAxis: any = $state();

  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale))
      .selectAll("text")
      .attr("transform", "rotate(45)")
      .style("text-anchor", "start");
    
    // tip:
    // similar to the x-axis, create a y-axis using d3.axisLeft() and bind it to the yAxis variable

    d3.select(yAxis)
        .call(d3.axisLeft(yScale));
  }

  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  Genres Overlapping with {primaryGenre} ({primaryTotal} {primaryGenre} movies)
</h3>

{#if movies.length > 0}
  <label>
    Primary genre:
    <select bind:value={primaryGenre}>
      {#each genres as genre}
        <option value={genre}>{genre}</option>
      {/each}
    </select>
  </label>

  <svg {width} {height}>
    <g class="bars">
      {#each overlapEntries as [genre, cnt] (genre)}
        <g>
          <rect
            width={xBarwidth}
            height={yScale(0) - yScale(cnt)}
            x={xScale(genre)}
            y={yScale(cnt)}
            fill="#449900"
            class="bar"
            opacity={selectedGenre && selectedGenre !== genre ? 0.25 : 1}
            onmouseover={() => {
              selectedGenre = genre;
            }}
            onmouseout={() => {
              selectedGenre = "";
            }}
          />

          <text
            x={xScale(genre)! + xBarwidth / 2}
            y={yScale(cnt) - 5}
            font-size="12"
            text-anchor="middle"
          >
          <!-- tip: the text below should change with the hover on interaction -->
            {selectedGenre === genre ? `${genre}: ${cnt}` : cnt}
          </text>
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
      Number of movies also tagged {primaryGenre}
    </text>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis}></g>
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis}></g>
  </svg>
{/if}

<style>
  .bar {
    transition:
      x 0.1s ease,
      y 0.1s ease,
      height 0.1s ease,
      width 0.1s ease; /* Smooth transition for height */
  }
</style>