<script>
    /** @type { import('p5-svelte/types').p5 } */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount, tick } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import { cubicOut } from 'svelte/easing';

    export let CANVAS_SIZE = 500;
    const LETTERS = 'bcdefhiklnoprstuvxyz'.toUpperCase().split('');
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 600;
    const POINT_DRAIN_AMOUNT = 100;
    const POINT_DRAIN_DELAY = 2000; // miliseconds

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    let level = 0;
    $: currentLetter = LETTERS[level];
    let currentRoundPoints = MAX_ROUND_POINTS;

    let trailPoints = [{ x: 0, y: 0, size: 0 }];
    let skeletonPoints = [{ x: 0, y: 0, marrow: false }];
    let fleshPoints = [{ x: 0, y: 0 }];

    let nextLevelP5;

    async function nextLevel(params) {
        currentRoundPoints -= currentRoundPoints - MIN_ROUND_POINTS;
        currentRoundPoints = MAX_ROUND_POINTS + POINT_DRAIN_AMOUNT;

        level++;
        trailPoints = [{ x: 0, y: 0, size: 0 }];
        skeletonPoints = [{ x: 0, y: 0, marrow: false }];
        fleshPoints = [{ x: 0, y: 0 }];

        console.log('Next level');

        await tick();
        nextLevelP5();
    }

    // Point drainer
    const slugCurrentRoundPointsStore = spring(0, {
        stiffness: 0.1,
        damping: 0.5,
        // duration: 1500,
        // easing: cubicOut,
    });

    onMount(async () => {
        const drainPointsInterval = setInterval(async () => {
            if (currentRoundPoints - POINT_DRAIN_AMOUNT < MIN_ROUND_POINTS) {
                await nextLevel();
                await sleep(POINT_DRAIN_DELAY);
                // return clearInterval(drainPointsInterval);
            }
            currentRoundPoints -= POINT_DRAIN_AMOUNT;
        }, POINT_DRAIN_DELAY);

        return () => {
            clearInterval(drainPointsInterval);
        };
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

        const drawTrail = async () => {
            p.noStroke();

            if (dist({ x: p.mouseX, y: p.mouseY }, trailPoints.at(-1)) > 5) {
                // if (!trailPoints.length || dist({ x: p.mouseX, y: p.mouseY }, trailPoints[0]) > 10) {
                if (p.mouseIsPressed) {
                    // console.log('push' + Math.random().toString().slice(-3, -1));
                    trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 10 });
                }
            }

            // trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 40 });

            const red = p.color(200, 100, 100, 120);
            const green = p.color(100, 200, 200, 40);

            let voidFilled = 0;

            // render points
            for (let i = 0; i < trailPoints.length; i++) {
                const point = trailPoints[i];

                let fleshed = false;
                const len = skeletonPoints.length + fleshPoints.length;
                for (let i = 0; i < len; i++) {
                    // loop over each skeleton point
                    const isBone = i < skeletonPoints.length;
                    const skPoint = isBone ? skeletonPoints[i] : fleshPoints[i - skeletonPoints.length];
                    // if (!skPoint) continue;
                    const distance = dist(point, skPoint);

                    if (isBone) {
                        // ok so its a bone
                        if (distance > 50) {
                            continue;
                        }

                        // bonesFilled++;
                        skeletonPoints[i].marrow = true;
                        p.fill(green);
                        p.circle(skPoint.x, skPoint.y, 30);
                    } else {
                        if (distance < 20) {
                            fleshed = true;
                            // we're looking at a piece of flesh
                            continue;
                        }
                        // tighter around flesh
                        // if (distance < 20) continue;
                    }

                    if (!fleshed && i === len - 1) {
                        voidFilled++;
                        p.fill(red);
                        p.circle(point.x, point.y, 40);
                    }
                }

                // p.fill(100, 200, 100);
                // p.circle(point.x, point.y, 20);

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

            let bonesFilled = 0;
            for (let i = 0; i < skeletonPoints.length; i++) {
                if (skeletonPoints[i].marrow) bonesFilled++;
            }

            if (bonesFilled >= skeletonPoints.length * 0.98) {
                console.log(bonesFilled, skeletonPoints.length);
                console.log(bonesFilled, voidFilled);
                console.log('COMPLETED');
                await nextLevel();
            }
        };

        // return a list of the coordinates for points of the letter skeleton
        const scanLetter = () => {
            const targets = [];

            // console.log('holu');

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

                    if (p.pixels[i] === 193) {
                        skeletonPoints.push({ x, y, marrow: false });

                        // if (nearest(x, y, 30)) {
                        //     p.fill(green);
                        //     p.circle(x, y, 5);
                        // } else {
                        //     p.fill(red);
                        // }
                    } else if (p.pixels[i] === 214) {
                        fleshPoints.push({ x, y, bone: false });
                    } else {
                        // if (p.pixels[i] !== 0) {
                        //     continue;
                        // }
                        // if (nearest(x, y, 4)) {
                        //     p.fill(green);
                        //     p.circle(x, y, 20);
                        // } else {
                        //     p.fill(red);
                        // }
                    }
                }
            }

            return targets;
        };

        const nearest = (x, y, radius = 20) => {
            for (let i = 0; i < trailPoints.length; i++) {
                const point = trailPoints[i];
                if (dist({ x, y }, point) < radius) {
                    return trailPoints[i];
                }
            }
        };

        const drawLetter = (sizeValue = 1) => {
            const size = 500;
            p.textSize(size * sizeValue);
            p.textAlign(p.CENTER, p.CENTER);

            // p.textFont('Lato');
            p.textFont('Work Sans');

            p.text(currentLetter, p.width / 2, p.height / 2);

            p.text(currentLetter, p.width / 2, p.height / 2);
        };

        const drawLetterFlesh = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.textStyle(p.BOLD);
            drawLetter();
        };

        const drawLetterSkeleton = () => {
            p.fill(193, 50, 50);
            p.textStyle(p.NORMAL);
            drawLetter(0.85);
        };

        const drawPointsBar = () => {
            slugCurrentRoundPointsStore.set(currentRoundPoints);
            const end = (($slugCurrentRoundPointsStore - MIN_ROUND_POINTS) / (MAX_ROUND_POINTS - MIN_ROUND_POINTS)) * p.width;

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

            await nextLevel();

            // setInterval(nextLevel, 1000);
            // scanLetter();
        };

        nextLevelP5 = () => {
            p.clear(0, 0, 0, 0);

            drawLetterFlesh();
            drawLetterSkeleton();

            scanLetter();
        };

        p.draw = () => {
            p.clear(0, 0, 0, 0);

            // if (score )
            drawLetterFlesh();
            p.fill('#5398AC30');
            p.textStyle(p.NORMAL);
            drawLetter(0.85);
            // drawLetterSkeleton();

            // let green = p.color(100, 200, 200, 150);
            // for (const po of skeletonPoints) {
            //     if (!po.bone) continue;
            //     p.fill(green);
            //     p.circle(po.x, po.y, 5);
            // }

            drawTrail();

            drawPointsBar();
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
