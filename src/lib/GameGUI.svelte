<script>
    import Game1 from './games/Game1.svelte';

    export let page;
</script>

<h1 class="mb-12">
    Excercise {page}
</h1>

<div class="flex flex-row w-full mx-0">
    <div class="p-4 border-none border-slate-700">
        <div class="flex flex-col gap-y-4  flex-shrink">
            <button on:click={_ => (page = 0)}>Home</button>
            {#each { length: 3 } as _, i}
                {#if page != i + 1}
                    <button disabled={i != 0} on:click={_ => (page = i + 1)}>Exercise {i + 1}</button>
                {/if}
            {/each}
        </div>
    </div>

    <div class="bg-black grid place-items-center relative">
        <div class="col-start-1 row-start-1">
            <Game1 />
        </div>
        <!-- <div class="hud w-full h-full bg-red-500 col-start-1 row-start-1  grid "> -->
        <!-- <button class="mt-auto ml-auto">full</button> -->
        <!-- </div> -->
    </div>
</div>
<div>
    <div class="prose dark:prose-invert text-left border-t pt-8 mt-8 border-slate-600">
        <h2>How to play</h2>
        <ul>
            <li>Use the mouse to trace the letter displayed on the screen.</li>
            <li>Stay on the line as best as you can for full points</li>
            <li>Complete in a timely manner for full points</li>
        </ul>
    </div>
</div>

<style>
</style>
