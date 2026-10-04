<script>
    let maxClick = $state(4);
    let cnt = $state(4); // tip: https://svelte.dev/docs/svelte/$state
    
  
    function onClick() {
      // tip: Since DOM (i.e., the webpage content) will automatically update based on values, [<p id="info">Remaining Number of Clicks: {cnt}</p>]
      // we only need to change the cnt number here.\
      if (cnt > 0) {
        cnt -= 1;
      }
    }
  </script>
  
  <h1>Ayman's VIS Site</h1>
  <img
    width="200px"
    src="https://media.tenor.com/RRGAOm6_neAAAAAe/milk-cat-cat.png"
    alt="Clueless cat covers himself in milk"
  />
  <div>
    You can click up to
    <select
        bind:value={maxClick} 
        onchange={() => (cnt = maxClick)}>
      {#each [2, 4, 6] as optionNum}
        <option value={optionNum}>
          {optionNum}
        </option>
      {/each}
    </select>
    times
  </div>
  <button onclick={onClick}> Click Me </button>

  <!-- tip: use {#if...} template syntax here (https://svelte.dev/docs/svelte/if) so that the content below will automatically update when cnt value changes -->
    <!-- `the content below should only show when cnt >0` -->
    {#if cnt > 0}
      <p id="info">Remaining Number of Clicks: {cnt}</p>
    {:else}
    <!-- The content below should only shown when cnt =0 -->
      <p>No more clicks allowed</p>
    {/if}
  
  
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
    }
    button {
      background-color: #44aa66;
      /* background-color: blue; */
      color: white;
      font-size: xx-large;
      padding: 10px 20px;
      border: none;
      cursor: pointer;
      border-radius: 5px;
    }
  </style>