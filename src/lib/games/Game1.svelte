<script>
    import P5 from 'p5-svelte';

  export const letters = "abcdefghijklmnopqrstuvwxyz".split()

     const sketch = (p5) => {
    let x = 0;
    let y = 0;
    let size = 15;
    let threshold = 0;

    p5.setup = () => {
      p5.createCanvas(400, 400);
    };

    p5.draw = () => {
      p5.stroke(1);
      threshold = p5.random(0.75);

      if (threshold < 0.1) p5.line(x, y, x + size, y + size);
      else if (0.505 > threshold > 0.5) p5.line(x, y, x, y + size);
      else p5.line(x, y + size, x + size, y);

      x = x + size;
      if (x > p5.width) {
        x = 0;
        y = y + size;
      }
    };
  };

</script>

  <div class=" w-full h-full border-slate-500 border">
        <P5 {sketch} />
  </div>