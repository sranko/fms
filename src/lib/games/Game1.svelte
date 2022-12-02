<script>
    /** @type { import('p5-svelte/types').p5 } */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount, tick } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import audio from '../audioOverlap';

    // setup constants
    export let CANVAS_SIZE = 500;
    const LETTERS = 'bcdefhiklnoprstuvxyz'.toUpperCase().split('');
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 600;
    const POINT_DRAIN_AMOUNT = 100;
    const POINT_DRAIN_DELAY = 2000; // miliseconds

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    let state = 0;
    let level = 0;
    // let level = LETTERS.length - 2;
    $: currentLetter = LETTERS[level];
    let currentRoundPoints = MAX_ROUND_POINTS;

    // setup data storage
    let trailPoints = [{ x: 0, y: 0, size: 0, consumed: false }];
    let skeletonPoints = [{ x: 0, y: 0, marrow: false }];
    let fleshPoints = [{ x: 0, y: 0, consumed: false }];
    let statsTracker = {
        totalBones: 0,
        totalErrors: 0,
        totalScore: 0,
    };

    // setup control variables
    let nextLevelP5;
    let pause = false;

    // resets states, increments points and signals to move on to the next round
    async function nextLevel(params) {
        if (level === LETTERS.length - 1) {
            state = 2;
            return;
        }

        pause = true;

        statsTracker.totalScore += currentRoundPoints > 0 ? currentRoundPoints : 0;

        currentRoundPoints -= currentRoundPoints - MIN_ROUND_POINTS;
        currentRoundPoints = MAX_ROUND_POINTS + POINT_DRAIN_AMOUNT;

        level++;
        trailPoints = [{ x: 0, y: 0, size: 0, consumed: false }];
        skeletonPoints = [{ x: 0, y: 0, marrow: false }];
        fleshPoints = [{ x: 0, y: 0, consumed: false }];

        console.log('Next level');

        await tick();
        await nextLevelP5();
        await sleep(500);
        pause = false;
    }

    // Point drainer smooth animation proxy
    const slugCurrentRoundPointsStore = spring(0, {
        stiffness: 0.1,
        damping: 0.4,
        // duration: 1500,
        // easing: cubicOut,
    });

    // load the sound effects
    audio.addSFX('win', '/success.mp3', 10);
    audio.addSFX('lose', '/error3.mp3', 10);

    // runs every n miliseconds to decrement time and points
    const drainPointsInterval = setInterval(async () => {
        if (state !== 1) return;
        if (currentRoundPoints - POINT_DRAIN_AMOUNT < MIN_ROUND_POINTS) {
            audio.playSFX('lose');
            await nextLevel();
            await sleep(POINT_DRAIN_DELAY);
            // return clearInterval(drainPointsInterval);
        }
        currentRoundPoints -= POINT_DRAIN_AMOUNT;
    }, POINT_DRAIN_DELAY);

    // when game changes
    onDestroy(() => {
        clearInterval(drainPointsInterval);
    });

    // Distance func
    const dist = (p1, p2) => Math.sqrt((p1.x - p2.x) ** 2 + (p1.y - p2.y) ** 2);

    // Rendering methods
    /** @type { import('p5-svelte/types').Sketch } TypeScript syntax */
    const sketch = p => {
        // Rendering with p5 js
        const COLORS = {
            bar: p.color(84, 212, 37),
            barScore: p.color(0, 0, 0),
            barScoreBG: p.color(255, 255, 255),
            barBG: p.color(255, 255, 255),
        };

        // only runs after significant movement to avoid drawing new trail in the same spot
        const drawTrail = async () => {
            p.noStroke();

            // if playing and the mouse has traveled more than 5 pixels
            if (!pause && dist({ x: p.mouseX, y: p.mouseY }, trailPoints.at(-1)) > 5) {
                if (p.mouseIsPressed) {
                    trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 10, consumed: false });
                }
            }

            // store colors
            const red = p.color(200, 100, 100, 200);
            const green = p.color(40, 200, 120, 40);

            let voidFilled = 0;

            // render points
            const len = skeletonPoints.length + fleshPoints.length;
            for (let i = 0; i < trailPoints.length; i++) {
                const point = trailPoints[i];

                let error = false;
                let fleshed = false;

                for (let i = 0; i < len; i++) {
                    // loop over each skeleton point
                    const isBone = i < skeletonPoints.length;
                    const fleshIndex = i - skeletonPoints.length;
                    const skPoint = isBone ? skeletonPoints[i] : fleshPoints[fleshIndex];
                    // if (!skPoint) continue;
                    const distance = dist(point, skPoint);

                    if (isBone) {
                        // ok so its a bone
                        if (distance > 50) {
                            continue;
                        }

                        // act on the detected skeleton point
                        skeletonPoints[i].marrow = true;
                        p.fill(green);
                        p.circle(skPoint.x, skPoint.y, 30);
                    } else {
                        if (distance < 20) {
                            // we're looking at a piece of flesh

                            fleshed = true;

                            continue;
                        }
                        // tighter around flesh
                    }

                    // if playing and not yet detected then show as error
                    if (!pause && !fleshed && i === len - 1) {
                        voidFilled++;
                        p.fill(red);
                        p.circle(point.x, point.y, 40);
                        error = true;
                    }
                }

                if (error) {
                    if (!trailPoints[i].consumed) {
                        // if (i >= skeletonPoints.length && !fleshPoints[fleshIndex].consumed) {
                        console.log('decrease');
                        currentRoundPoints -= 10;
                        statsTracker.totalErrors++;
                        // currentRoundPoints -= 20;
                        trailPoints[i].consumed = true;
                    }
                }
            }

            let bonesFilled = 0;
            for (let i = 0; i < skeletonPoints.length; i++) {
                if (skeletonPoints[i].marrow) bonesFilled++;
            }

            if (bonesFilled >= skeletonPoints.length * 0.98) {
                console.log(bonesFilled, skeletonPoints.length);
                console.log(bonesFilled, voidFilled);
                console.log('COMPLETED');

                statsTracker.totalBones += bonesFilled;

                // winSFX.play();() - Ca
                audio.playSFX('win');
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
                    } else if (p.pixels[i] === 214) {
                        fleshPoints.push({ x, y, consumed: false });
                    }
                }
            }

            return targets;
        };

        const drawLetter = (sizeValue = 1) => {
            const size = 500;
            p.textSize(size * sizeValue);
            p.textAlign(p.CENTER, p.CENTER);
            p.textFont('Work Sans');
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

        const drawStartMenu = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.textStyle(p.NORMAL);
            p.textSize(100);
            p.textAlign(p.CENTER, p.CENTER);
            p.textFont('Work Sans');
            const startMessage = 'PLAY';
            p.text(startMessage, p.width / 2, p.height / 2);
        };

        const drawEndMenu = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.textStyle(p.BOLD);
            p.textSize(50);
            p.textAlign(p.CENTER, p.CENTER);
            p.textFont('Work Sans');
            p.text(
                (100 - (statsTracker.totalErrors / statsTracker.totalBones) * 100).toFixed(1) + '% Accuracy',
                p.width / 2,
                p.height / 2
            );
            p.text('Score ' + statsTracker.totalScore, p.width / 2, p.height / 2 - 100);
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
        };

        p.setup = async () => {
            p.createCanvas(CANVAS_SIZE, CANVAS_SIZE);
            await nextLevel();
        };

        nextLevelP5 = () => {
            p.clear(0, 0, 0, 0);

            drawLetterFlesh();
            drawLetterSkeleton();

            scanLetter();
        };

        p.mouseClicked = () => {
            if (state === 0) state = 1;
            if (state === 2) {
                // level = 0;
                // state = 1;
            }
        };

        p.draw = () => {
            p.clear(0, 0, 0, 0);

            if (state === 0) {
                drawStartMenu();
            } else if (state === 1) {
                drawLetterFlesh();
                p.fill('#5398AC30');
                p.textStyle(p.NORMAL);
                drawLetter(0.85);

                drawTrail();

                drawPointsBar();
            } else if (state === 2) {
                drawEndMenu();
            }
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
