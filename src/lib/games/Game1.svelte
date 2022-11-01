<script>
    /** @type { import('p5-svelte/types').p5 } TypeScript syntax */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount, tick } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import { cubicOut } from 'svelte/easing';

    export let CANVAS_SIZE = 400;
    const LETTERS = 'bcdefhiklnoprstuvwxyz'.toUpperCase().split('');
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 600;
    const POINT_DRAIN_AMOUNT = 100;
    const POINT_DRAIN_DELAY = 1500; // miliseconds

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    let level = 0;
    $: currentLetter = LETTERS[level];
    let currentRoundPoints = MAX_ROUND_POINTS;

    let trailPoints = [{ x: 0, y: 0, size: 0 }];

    function nextLevel(params) {
        level++;
        trailPoints = [{ x: 0, y: 0, size: 0 }];
    }

    // Point drainer
    const slugCurrentRoundPoints = spring(0, {
        stiffness: 0.1,
        damping: 0.5,
        // duration: 1500,
        // easing: cubicOut,
    });
    const drainPointsInterval = setInterval(async () => {
        if (currentRoundPoints - POINT_DRAIN_AMOUNT < MIN_ROUND_POINTS) {
            currentRoundPoints -= currentRoundPoints - MIN_ROUND_POINTS;
            currentRoundPoints = MAX_ROUND_POINTS + POINT_DRAIN_AMOUNT;
            nextLevel();
            await sleep(POINT_DRAIN_DELAY);
            // return clearInterval(drainPointsInterval);
        }
        currentRoundPoints -= POINT_DRAIN_AMOUNT;
    }, POINT_DRAIN_DELAY);
    onDestroy(() => {
        clearInterval(drainPointsInterval);
    });

    // Distance func
    const dist = (p1, p2) => Math.sqrt((p1.x - p2.x) ** 2 + (p1.y - p2.y) ** 2);

    /** @type { import('p5-svelte/types').Sketch } TypeScript syntax */
    const sketch = p => {
        // Rendering with p5 js
        const COLORS = {
            bar: p.color(84, 212, 37),
            barScore: p.color(0, 0, 0),
            barScoreBG: p.color(255, 255, 255),
            barBG: p.color(255, 255, 255),
        };

        const drawTrail = () => {
            p.noStroke();

            if (dist({ x: p.mouseX, y: p.mouseY }, trailPoints.at(-1)) > 5) {
                // if (!trailPoints.length || dist({ x: p.mouseX, y: p.mouseY }, trailPoints[0]) > 10) {
                if (p.mouseIsPressed) {
                    console.log('push' + Math.random().toString().slice(-3, -1));
                    trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 10 });
                }
            }

            // trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 40 });

            // render points
            for (let i = 0; i < trailPoints.length; i++) {
                const point = trailPoints[i];

                // p.circle(point.x, point.y, point.size);

                // if (point.size <= 0 && trailPoints.length > 0) trailPoints.shift();
                // if (point.size > 0) point.size -= 2;
                // if (i === 0) return;

                // const dec = (30 - point.size) * 0.1;

                // if (point.size - dec <= 5) {
                //     trailPoints.shift();
                //     continue;
                // }

                // if (i !== 0) point.size -= dec;
            }
        };

        const scanLetter = () => {
            const targets = [];

            // canvas has a width
            // has more pixels than canvas units

            // const block = 0.05 * p.width;
            const block = 5;
            p.loadPixels();
            const d = p.pixelDensity();
            p.noStroke();

            for (let x = 0; x < p.width; x += block) {
                for (let y = 0; y < p.height; y += block) {
                    const i = 4 * d * (y * d * p.width + x);

                    // if (x % 10 == 0 && y % 10 == 0) {
                    if (p.pixels[i] === 193) {
                        // targets.push({ x, y });
                        let red = p.color(200, 100, 100, 150);
                        let green = p.color(100, 200, 200, 150);

                        if (nearest(x, y, 25)) {
                            p.fill(green);
                            p.circle(x, y, 5);
                        } else {
                            p.fill(red);
                        }
                    } else if (p.pixels[i] === 214) {
                        // let red = p.color(200, 100, 100, 50);
                        // let green = p.color(100, 200, 100, 50);
                        // if (nearest(x, y)) {
                        //     p.fill(green);
                        // } else {
                        //     p.fill(red);
                        // }
                        // p.circle(x, y, 10);
                    } else {
                        if (p.pixels[i] !== 0) {
                            continue;
                        }
                        // console.log(p.pixels[i]);
                        // continue;
                        let red = p.color(100, 200, 200, 10);
                        let green = p.color(200, 200, 100, 80);

                        if (nearest(x, y, 4)) {
                            p.fill(green);
                            p.circle(x, y, 20);
                        } else {
                            p.fill(red);
                        }
                    }
                }
            }

            // let fullCanvas = 1 * (p.width * d) * (p.height * d);

            return;
        };

        const nearest = (x, y, radius = 20) => {
            for (let i = 0; i < trailPoints.length; i++) {
                const point = trailPoints[i];
                if (dist({ x, y }, point) < radius) {
                    return trailPoints[i];
                }
            }
        };

        const drawLetter = () => {
            const letterColor = p.color(214, 255, 255, 100);
            const size = 400;
            p.textAlign(p.CENTER, p.CENTER);
            p.textSize(size);
            // p.textFont('Lato');
            p.textFont('Work Sans');

            p.fill(letterColor);
            p.textStyle(p.BOLD);

            p.text(currentLetter, 200, 200);

            p.fill(193, 50, 50);
            p.textSize(size * 0.85);
            p.textStyle(p.NORMAL);

            p.text(currentLetter, 200, 200);
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

        let fontRegular, fontItalic, fontBold;
        p.preload = async () => {
            // fontRegular = p.loadFont('assets/Regular.otf');
            // fontItalic = p.loadFont('assets/Italic.ttf');
            // fontBold = p.loadFont('assets/Bold.ttf');
        };

        p.setup = async () => {
            p.createCanvas(CANVAS_SIZE, CANVAS_SIZE);
            p.clear(0, 0, 0, 0);
            drawLetter();
            // setInterval(nextLevel, 1000);
            // scanLetter();
        };

        p.draw = () => {
            p.clear(0, 0, 0, 0);
            drawLetter();
            scanLetter();
            drawPointsBar();
            drawTrail();
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
