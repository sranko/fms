<script>
    let page = 0;
    import Game from './lib/GameGUI.svelte';
</script>

{#if page == 0}
    <main class="flex flex-col items-center mx-auto w-full">
        <h1>Homepage</h1>

        <div class="flex flex-col gap-y-4 mt-12 w-full">
            {#each { length: 3 } as _, i}
                <button class="bg-sky-800 hover:scale-105 transition-transform" on:click={_ => (page = i + 1)}>
                    Exercise {i + 1}
                </button>
            {/each}
        </div>
    </main>
{:else}
    <Game bind:page />
{/if}

<style>
    button {
    }
</style>
