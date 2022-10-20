<script>
    let page = 'home';
    import Game from './lib/Game.svelte';
</script>

{#if page == 'home'}
    <main class="flex flex-col items-center mx-auto w-full">
        <h1>Homepage</h1>

        <div class="flex flex-col gap-y-4 mt-12 w-full">
            <button on:click={_ => (page = '1')}>Exercise 1</button>

            <button on:click={_ => (page = '2')}>Exercise 2</button>

            <button on:click={_ => (page = '3')}>Exercise 3</button>
        </div>
    </main>
{:else}
    <Game {page} />
{/if}

<style>
    button {
        @apply border-2 text-2xl border-slate-500 p-5;
    }
</style>
