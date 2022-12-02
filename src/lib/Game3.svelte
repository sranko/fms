<script>
    /**
     * Technical Comments
     *
     * How Path generation works:
     * The goal is to create an engaging training experience fair for todlers
     * We accomplish this by generating our path intelligently
     * Instead of random difficult paths, we came up with an algorithim for clean paths of dyamic difficulty
     * We start from the center of the screen height wise, at x = 0
     * We then generate a random offset within the bounds of the screen based on the difficulty determined by the level
     * This allows us to start from a more or less staight path, and increase in curves
     * We use the p5js curve function to create these curves dynamically from a set of coordinates
     * This creates paths that are more human to trace and help a toddler with patterns and intution while drawing
     *
     * How the points timer bar works:
     * We redraw the bar each frame with a smooth animation
     * This is an innovative combination of points and time left for the level, inspired by quizziz ui
     * We use the visual display of time and points instead of plain text because toddlers are more visual learners,
     * and will understand this representation of points better
     * This reduces the clutter on the screen, allowing them to focus more on the game
     *
     * How path tracing collision works:
     * We paint every path 2 times
     * One time is drawn in a thin stroke weight
     * The other time is drawn in thick stroke weight
     * This allows us to perform nice and forgiving collision for toddlers
     * These are similiar intelligent collision mechanics as used
     * (Helps mantain uniform look and feel throughout our games)
     *
     * We have a large allowable margin of error for the thin line,
     * which is about the radius of the thick path stroke weight in pixels
     *
     * This thin like is called the skeleton, while the thicker path is the flesh.
     * This naming convention is used to simplify variable names via a biological analogy
     *
     * After we draw both of these variants of the path for that level,
     * we scan across the pixels in the canvas,
     * looking for the pixel colors of the skeleton and flesh,
     * and pushing those values to their own arrays respectivly
     *
     * Now, as the user moves the mouse arround,
     * we push the coordinates of the mouse to an array representing the path drawn
     * if the current coordinates are too close to the previous coordinates already drawn,
     * we skip adding that trail point to reduce duplication and improve iteration performance
     *
     * Now we have 3 main arrays:
     * Unique trail points, skeleton points, and flesh points
     * (Refer to brief comments above for naming convention)
     *
     * Next, everytime a new trail point is created, we perform several checks to determine collision context
     * If the trail point is within a radius of the skeleton, we set that skeleton point to be active
     * else if the trail point is *not* within a safety threshold of the flesh, that means we're outside of the path,
     * and set that trail point to an active error
     *
     * This allows us to snap the path cleanly to the path when you're on track,
     * and show you your real wrong path when you're off track
     *
     * Last, on each frame we look over the arrays and render all of the active skeleton points and error trail points
     */

    /** @type { import('p5-svelte/types').p5 } */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount, tick } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import audio from './audioOverlap';

    export let CANVAS_SIZE = 500;
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 600;
    const POINT_DRAIN_AMOUNT = 100;
    const POINT_DRAIN_DELAY = 2000; // miliseconds

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    let state = 0;
    let level = 0;

    let coords = [];
    let currentRoundPoints = MAX_ROUND_POINTS;

    let trailPoints = [{ x: 0, y: 0, size: 0, consumed: false }];
    let skeletonPoints = [{ x: 0, y: 0, marrow: false }];
    let fleshPoints = [{ x: 0, y: 0, consumed: false }];
    let statsTracker = {
        totalBones: 0,
        totalErrors: 0,
        totalScore: 0,
    };

    let nextLevelP5;
    let pause;

    async function nextLevel(params) {
        if (level === 11) {
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

    // Point drainer
    const slugCurrentRoundPointsStore = spring(0, {
        stiffness: 0.1,
        damping: 0.4,
        // duration: 1500,
        // easing: cubicOut,
    });

    // winSFX = new Audio('success.mp3');
    audio.addSFX('win', 'success.mp3', 10);
    audio.addSFX('lose', 'error3.mp3', 10);

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

        // Create the trail points as described above
        const drawTrail = async () => {
            p.noStroke();

            if (!pause && dist({ x: p.mouseX, y: p.mouseY }, trailPoints.at(-1)) > 5) {
                // if (!trailPoints.length || dist({ x: p.mouseX, y: p.mouseY }, trailPoints[0]) > 10) {
                if (p.mouseIsPressed) {
                    // console.log('push' + Math.random().toString().slice(-3, -1));
                    trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 10, consumed: false });
                }
            }

            // trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 40 });

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
                        if (distance > 60) {
                            continue;
                        }

                        // bonesFilled++;
                        skeletonPoints[i].marrow = true;
                        p.fill(green);
                        p.circle(skPoint.x, skPoint.y, 18);
                    } else {
                        if (distance < 20) {
                            // we're looking at a piece of flesh
                            // currentRoundPoints -= 20;

                            fleshed = true;

                            continue;
                        }
                        // tighter around flesh
                        // if (distance < 20) continue;
                    }

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
                        // console.log('decrease');
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
                    } else {
                    }
                }
            }

            return targets;
        };

        // We use the P5 js curve function to draw smooth path between coordinates
        const drawLetter = () => {
            // points is an array of y and x values

            p.noFill();

            p.beginShape();

            p.curveVertex(coords[0], coords[1]);
            for (let i = 0; i < coords.length; i += 2) {
                // p.ellipse(coords[i], coords[i + 1], 10, 10);
                p.curveVertex(coords[i], coords[i + 1]);
            }
            p.curveVertex(coords.at(-2), coords.at(-1));

            p.endShape();

            p.fill(0, 0, 0, 0);
            p.strokeWeight(2);
            p.stroke('#ffffff');
            for (let i = 0; i < coords.length; i += 2) {
                p.ellipse(coords[i], coords[i + 1], 40, 40);
            }
        };

        const drawLetterFlesh = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.stroke(letterColor);
            p.strokeWeight(20);
            drawLetter();
        };

        const drawLetterSkeleton = () => {
            p.fill(193, 50, 50);
            p.stroke(193, 50, 50);
            p.strokeWeight(10);
            drawLetter();
        };

        const drawStartMenu = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.textStyle(p.NORMAL);
            p.textSize(100);

            p.textAlign(p.CENTER, p.CENTER);

            // p.textFont('Lato');
            p.textFont('Work Sans');
            // p.background(200, 200, 200);

            const startMessage = 'PLAY';

            p.text(startMessage, p.width / 2, p.height / 2);
        };

        const drawEndMenu = () => {
            const letterColor = p.color(214, 255, 255, 255);
            p.fill(letterColor);
            p.textStyle(p.BOLD);
            p.textSize(50);

            p.textAlign(p.CENTER, p.CENTER);

            // p.textFont('Lato');
            p.textFont('Work Sans');
            // p.background(200, 200, 200);

            p.text(
                (100 - (statsTracker.totalErrors / statsTracker.totalBones) * 100).toFixed(1) + '% Accuracy',
                p.width / 2,
                p.height / 2
            );

            p.text('Score ' + statsTracker.totalScore, p.width / 2, p.height / 2 - 100);

            // p.text(statsTracker.totalErrors, p.width / 2, p.height / 2 - 100);
        };

        // Points value with a proxy to introduce animation
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

        function getRandomArbitrary(min, max) {
            return Math.random() * (max - min) + min;
        }

        function generatePathPoints() {
            let lasty = p.height / 2;
            const varConst = level < 10 ? 10 - level : 1;
            const pady = 30;
            const out = [];
            for (let i = p.width / 10; i < p.width; i += p.width / 10) {
                const x = i;
                const y = lasty + getRandomArbitrary(pady - lasty, p.height - pady - lasty) / varConst;
                // const y = lasty + getRandomArbitrary(p.height - lasty, -lasty) / varConst;
                lasty = y;
                out.push(x);
                out.push(y);
            }
            return out;
        }

        p.setup = async () => {
            p.createCanvas(CANVAS_SIZE, CANVAS_SIZE);

            await nextLevel();
        };

        nextLevelP5 = () => {
            p.clear(0, 0, 0, 0);

            coords = generatePathPoints();

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
            // p.clear(, 1);
            p.clear(120, 120, 0, 0);

            if (state === 0) {
                drawStartMenu();
            } else if (state === 1) {
                // if (score )
                drawLetterFlesh();

                drawTrail();

                drawPointsBar();
            } else if (state === 2) {
                drawEndMenu();
            }
        };
    };
</script>

<!-- <audio src="win.mp3" preload="auto" bind:this={winSFX} controls>
    <track kind="captions" />
</audio> -->

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
