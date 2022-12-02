<script>
    /**
     * The GUI is responsible for the layout of the game
     * We have a convenient menu to the homescreen and to all the other games at all times
     * The GUI renders each correct game as it's requested,
     * and set of directions as it's requested
     * We also show the current title of the game
     * This helps us create a clean and accessible user interface
     */

    // Import our game files
    import Game1 from './Game1.svelte';
    import Game2 from './Game2.svelte';
    import Game3 from './Game3.svelte';

    export let page;
</script>

<!-- The title of the game -->
<h1 class="mb-12">
    Excercise {page}
</h1>

<!-- Display the menu buttons linking to each game and back home  -->
<div class="flex flex-row w-full mx-0">
    <div class="p-4 border-none border-slate-700">
        <div class="flex flex-col gap-y-4  flex-shrink">
            <button class="bg-sky-800 hover:scale-105 transition-transform" on:click={_ => (page = 0)}>Home</button>
            {#each { length: 3 } as _, i}
                {#if page != i + 1}
                    <button class="bg-sky-800 hover:scale-105 transition-transform" on:click={_ => (page = i + 1)}>
                        Exercise {i + 1}
                    </button>
                {/if}
            {/each}
        </div>
    </div>

    <!-- Render the right game based on what page is selected to show -->
    <div class="bg-sky-900 grid place-items-center relative">
        <div class="col-start-1 row-start-1">
            {#if page === 1}
                <Game1 />
            {:else if page === 2}
                <Game2 />
            {:else}
                <Game3 />
            {/if}
        </div>
    </div>
</div>

<!-- Render the correct directions based on what page is selected to show -->
<div>
    <div class="prose text-left text-white text-xl font-bold border-t pt-8 mt-8 border-slate-600">
        <h2 class="text-white">How to play</h2>
        <ul>
            {#if page === 1}
                <li>Use the mouse to trace the letter displayed on the screen.</li>
                <li>Stay on the line as best as you can for full points</li>
                <li>Complete in a timely manner for full points</li>
            {:else if page === 2}
                <li>Use the mouse to click the targets displayed on the screen.</li>
                <li>Complete in a timely manner before time runs out.</li>
            {:else}
                <li>Use the mouse to trace the path displayed on the screen.</li>
                <li>Stay on the line as best as you can for full points</li>
                <li>Complete in a timely manner for full points</li>
                <li>Difficulty of the path progressively increases!</li>
            {/if}
        </ul>
    </div>
</div>

<style>
</style>
