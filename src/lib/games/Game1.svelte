<script>
    /** @type { import('p5-svelte/types').p5 } TypeScript syntax */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import { cubicOut } from 'svelte/easing';

    const letters = 'abcdefghijklmnopqrstuvwxyz'.toUpperCase().split('');
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 600;
    const POINT_DRAIN_AMOUNT = 50;
    const POINT_DRAIN_DELAY = 2000; // miliseconds

    let level = 0;
    let currentLetter = letters[0];
    let currentRoundPoints = MAX_ROUND_POINTS;

    function nextLevel(params) {
        level++;
    }

    // Point drainer
    const slugCurrentRoundPoints = spring(0, {
        stiffness: 0.2,
        damping: 0.5,
        // duration: 1500,
        // easing: cubicOut,
    });
    const drainPointsInterval = setInterval(() => {
        if (currentRoundPoints - POINT_DRAIN_AMOUNT < MIN_ROUND_POINTS) {
            currentRoundPoints -= currentRoundPoints - MIN_ROUND_POINTS;
            return clearInterval(drainPointsInterval);
        }
        currentRoundPoints -= POINT_DRAIN_AMOUNT;
        console.log(currentRoundPoints);
    }, POINT_DRAIN_DELAY);
    onDestroy(() => {
        clearInterval(drainPointsInterval);
    });

    /** @type { import('p5-svelte/types').Sketch } TypeScript syntax */
    const sketch = p => {
        // Rendering with p5 js
        const COLORS = {
            bar: p.color(84, 212, 37),
            barScore: p.color(0, 0, 0),
            barScoreBG: p.color(255, 255, 255),
            barBG: p.color(255, 255, 255),
        };

        const drawLetter = () => {
            const letterColor = p.color(255, 255, 255, 60);
            p.textSize(300);
            p.textAlign(p.CENTER, p.CENTER);
            p.fill(letterColor);

            p.text(letters[0], 200, 200);
        };

        const drawPointsBar = () => {
            slugCurrentRoundPoints.set(currentRoundPoints);
            const end = (($slugCurrentRoundPoints - MIN_ROUND_POINTS) / (MAX_ROUND_POINTS - MIN_ROUND_POINTS)) * p.width;

            const barWidth = 15;
            p.fill(COLORS.barBG);
            p.rect(0, 0, p.width, barWidth);

            const percent = end / p.width;
            p.fill((1 - percent) * 255, percent * 255, 50);
            p.rect(0, 0, end, barWidth);

            return;
            p.fill(COLORS.barScoreBG);
            p.ellipse(end, 10, 50, 40);

            p.textSize(20);
            p.textAlign(p.CENTER, p.TOP);
            p.fill(COLORS.barScore);

            p.text(currentRoundPoints, end, 0);
        };

        p.setup = () => {
            p.createCanvas(400, 400);
        };

        p.draw = () => {
            p.clear(0, 0, 0, 0);
            drawLetter();
            drawPointsBar();
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
