<script>
    import P5 from 'p5-svelte';
    // import Game1 from "./src/lib/Game1";
  
    export let page;

    let canvas;
    let width = 55;
    let height = 55;

    const sketch = p5 => {
        p5.setup = () => {
            p5.createCanvas(400, 400);
        };

        p5.draw = () => {
            p5.ellipse(p5.width / 2, p5.height / 2, width, height);
        };
    };
</script>

  <h1 class="mb-12">
    Excercise {page}
  </h1>

<div class="flex flex-row w-full mx-0">
    <div class="p-4 border-none border-slate-700">
        <div class="flex flex-col gap-y-4  flex-shrink">
          <button on:click={_ => (page = 0)}>Home</button>
          {#each {length:3} as _, i}
            {#if page!=i+1}
            <button on:click={_ => (page = (i+1))}>Exercise {i+1}</button>
          {/if}
          {/each}
        </div>
    </div>

  <div class="w-96 h-96 bg-slate-500 grid place-items-center" >
    Game {page} Placeholder
  </div>

<!--   <div>
    <div>
        <label>
            Width
            <input type="range" bind:value={width} min="100" max="1000" step="0.01" />
            {width}
        </label>

        <label class="block">
            Height
            <input type="range" bind:value={height} min="100" max="1000" step="0.01" />
            {height}
        </label>

        <P5 {sketch} />
    </div>
  </div> -->
  
</div>
<div>
  <div class="prose dark:prose-invert text-left border-t pt-8 mt-8 border-slate-600">
    <h2>
      How to play
    </h2>
    <ul>

      <li>
              Use the mouse to trace the letter displayed on the screen.
      </li>
      <li>
Stay on the line as best as you can for full points
        
      </li>
      <li>
Complete in a timely manner for full points
        
      </li>
      
    </ul>
  </div>
</div>

<style>
  
</style>