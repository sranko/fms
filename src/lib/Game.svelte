<script>
    import P5 from 'p5-svelte';
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

<div class="flex flex-row w-full ">
    <div class="p-4 outline">
        <div class="flex flex-col gap-y-4  flex-shrink">
            <button on:click={_ => (page = '1')}>Exercise 1</button>

            <button on:click={_ => (page = '2')}>Exercise 2</button>

            <button on:click={_ => (page = '3')}>Exercise 3</button>
        </div>
    </div>

    <div>
        <label>
            Width
            <input type="range" bind:value={width} min="100" max="1000" step="0.01" />
            {width}
        </label>

        <label>
            Height
            <input type="range" bind:value={height} min="100" max="1000" step="0.01" />
            {height}
        </label>

        <P5 {sketch} />
    </div>
</div>
