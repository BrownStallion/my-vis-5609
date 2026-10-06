<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte";
  import Q1 from "$lib/Q1.svelte";
  import Q2 from "$lib/Q2.svelte";

  // Reactive variable for storing the data
  let movies: TMovie[] = [];

  // Function to load the CSV
  async function loadCsv() {
    try {
      const csvUrl = "./summer_movies.csv";
      movies = await d3.csv(csvUrl, (row) => {
        // TIP: in row, all values are strings, so we need to use a row conversion function here to format them
        return {
          ...row, // spread syntax to copy all properties from row
          num_votes: Number(row.num_votes),
          year: new Date(row.year),
          average_rating: Number(row.average_rating),
          runtime_minutes: Number(row.runtime_minutes),
          genres: row.genres.split(",")
          // please also format the values for other non-string attributes. You can check the attributes in the CSV file
        };
      });

      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  // Call the loader when the component mounts
  onMount(loadCsv);
</script>

<h1>Summer Movies</h1>

<p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>
<Bar {movies} />

<h2>Question 1</h2>
<Q1 {movies} />
<p>
    The top 3 genres Drama, Comedy, and Romance. These remained the top 3
    for most of history, but Comedy surpassed Drama 8 times since 1946, and
    horror surpassed Romance for the #3 spot twice in the 2010s and 2020s.
</p>

<h2>Question 2</h2>
<Q2 {movies} />

<p>
    Romance and Drama have the biggest co-occurence rate with 122 overlapping
    movies. Comedy and Drama have the second most co-occurences with 108 movies. 
    Romance and Comedy have 73 co-occurences, and Family/Comedy represents 33 movies.
</p>