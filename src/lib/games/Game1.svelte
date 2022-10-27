<script>
    import P5 from 'p5-svelte';

    const letters = 'abcdefghijklmnopqrstuvwxyz'.toUpperCase().split('');

    /**
    //  * @type {(Sketch)=>any}
     */
    const sketch = p5 => {
        let x = 0;
        let y = 0;
        let size = 15;
        let threshold = 0;

        p5.setup = () => {
            p5.createCanvas(400, 400);
        };

        p5.draw = () => {
            p5.textSize(80);
            p5.textAlign(p5.CENTER);
            p5.text(letters[0], 200, 200);
            // p5.fill(0, 102, 153);
            // p5.stroke(1);
            // threshold = p5.random(0.75);

            // if (threshold < 0.1) p5.line(x, y, x + size, y + size);
            // else if (0.505 > threshold && threshold > 0.5) p5.line(x, y, x, y + size);
            // else p5.line(x, y + size, x + size, y);

            // x = x + size;
            // if (x > p5.width) {
            //     x = 0;
            //     y = y + size;
            // }
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
