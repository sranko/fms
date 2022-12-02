<script>
    /**
     * Technical Comments
     * We took a timeless game concept and improved it for toddlers developing FMS
     * Instead of targets appearing randomly everywhere on the screen,
     * we limit the next target to be within a radius of the last target
     * this helps a toddler trust their motor skill response and train the coordination better,
     * because they are able to focus on predicitng where the next target would be
     *
     * We generate the new coordinates by using right triangle laws
     * and some triganometry concepts
     * We generate a random angle, then solve for the triangle with a specific hypothenuse
     * Then, we add this new triangle to the last targets coordinates to get the new coordinates
     * Then we spawn the new circle there, instead of simply choosing random confusing coordinates anywhere on the canvas
     *
     * We use animations in order to help toddlers follow along better with what's happening,
     * by mimicking how objects behave in the real world
     * Instead of the targets appearing and dissapearing, we add attractive animations
     * This means toddlers see what is going on and have a visual cue on what to do
     *
     * we have a bar in the top representing the time remaining, that changes color as time runs out
     * this is a good visual representation
     */

    /** @type { import('p5-svelte/types').p5 } */
    import P5 from 'p5-svelte';
    import { onDestroy, onMount, tick } from 'svelte';

    import { tweened, spring } from 'svelte/motion';
    import audio from './audioOverlap';

    export let CANVAS_SIZE = 500;
    const MAX_ROUND_POINTS = 1000;
    const MIN_ROUND_POINTS = 0;
    const POINT_DRAIN_AMOUNT = 50;
    const POINT_DRAIN_DELAY = 1000; // miliseconds

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    let state = 0;
    let level = 0;
    let currentRoundPoints = MAX_ROUND_POINTS;

    // let targetPoints = [{ x: 200, y: 200, size: 80 }];
    let trans = false;
    let show = false;
    let targetA = { x: 150, y: 150, size: 80 };
    let targetB = { x: 300, y: 300, size: 80 };
    let trailPoints = [{ x: 0, y: 0, size: 0, consumed: false }];
    let statsTracker = {
        totalBones: 0,
        totalErrors: 0,
        totalScore: 0,
    };

    let nextLevelP5;

    async function nextLevel(params) {
        if (level >= 20) {
            state = 2;
            return;
        }

        statsTracker.totalScore += currentRoundPoints > 0 ? currentRoundPoints : 0;

        currentRoundPoints -= currentRoundPoints - MIN_ROUND_POINTS;
        currentRoundPoints = MAX_ROUND_POINTS + POINT_DRAIN_AMOUNT;

        level++;
        trailPoints = [{ x: 0, y: 0, size: 0, consumed: false }];

        console.log('Next level');

        await tick();
        nextLevelP5();
    }

    // shrinker
    const slugShrinkerStore = spring(0, {
        stiffness: 0.1,
        damping: 0.2,
        // duration: 1500,
        // easing: cubicOut,
    });
    // shrinker
    const slugShrinkerStoreB = spring(0, {
        stiffness: 0.05,
        damping: 0.4,
        // duration: 1500,
        // easing: cubicOut,
    });

    // Point drainer
    const slugCurrentRoundPointsStore = spring(0, {
        stiffness: 0.1,
        damping: 0.4,
        // duration: 1500,
        // easing: cubicOut,
    });

    audio.addSFX('win', 'success.mp3', 10);
    audio.addSFX('lose', 'error3.mp3', 10);

    const drainPointsInterval = setInterval(async () => {
        if (state !== 1) return;
        if (currentRoundPoints - POINT_DRAIN_AMOUNT < MIN_ROUND_POINTS) {
            audio.playSFX('lose');
            state = 2;
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

        // Solve a triangle in order to relocate a point smartly
        const relocate = () => {
            // const hypothenuse = getRandomArbitrary(100, 150);
            const hypo = 120;
            const newAngle = getRandomArbitrary(10, 35);
            const x = Math.sin(newAngle) * hypo;
            const y = Math.cos(newAngle) * hypo;

            const newX = targetA.x + x;
            const newY = targetA.y + y;

            if (newX < hypo || newX > p.width - hypo || newY < hypo || newY > p.height - hypo) relocate();
            else {
                targetA.x = newX;
                targetA.y = newY;
            }
        };

        function getRandomArbitrary(min, max) {
            return Math.random() * (max - min) + min;
        }

        const drawTrail = async () => {
            p.noStroke();

            if (dist({ x: p.mouseX, y: p.mouseY }, trailPoints.at(-1)) > 5) {
                // if (!trailPoints.length || dist({ x: p.mouseX, y: p.mouseY }, trailPoints[0]) > 10) {
                if (p.mouseIsPressed) {
                    if (dist({ x: p.mouseX, y: p.mouseY }, { x: targetA.x, y: targetA.y }) < 50) {
                        const oldX = targetA.x;
                        const oldY = targetA.y;

                        // targetA.size = 0;
                        // slugShrinkerStore.set(0);
                        // await sleep(200);
                        statsTracker.totalBones++;
                        trans = true;

                        audio.playSFX('win');

                        slugShrinkerStore.set(0);
                        targetA.size = 0;
                        relocate();

                        targetB.x = oldX;
                        targetB.y = oldY;
                        slugShrinkerStoreB.set(80);
                        targetB.size = 80;
                        show = true;
                        await sleep(200);
                        trans = false;
                        slugShrinkerStoreB.set(0);
                        slugShrinkerStore.set(80);

                        await sleep(300);
                        show = false;
                        // targetB.x = targetA.x;
                        // targetB.y = targetA.y;
                    }
                    // console.log('push' + Math.random().toString().slice(-3, -1));
                    // trailPoints.push({ x: p.mouseX, y: p.mouseY, size: 10, consumed: false });
                }
            }
        };

        // Functions to paint the required shapes like the targets and menus

        const drawTarget = (x, y, size) => {
            p.strokeWeight(4);
            p.stroke('#ffffff');
            p.fill('#a2b2ff');
            p.circle(x, y, size);
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

            p.text('Score ' + statsTracker.totalBones, p.width / 2, p.height / 2 - 100);

            // p.text(statsTracker.totalErrors, p.width / 2, p.height / 2 - 100);
        };

        const drawPointsBar = () => {
            slugCurrentRoundPointsStore.set(currentRoundPoints);
            const end = (($slugCurrentRoundPointsStore - MIN_ROUND_POINTS) / (MAX_ROUND_POINTS - MIN_ROUND_POINTS)) * p.width;

            const barWidth = 15;
            p.fill(COLORS.barBG);
            p.rect(0, 0, p.width, barWidth);

            const percent = end / p.width;
            p.fill(100, 255, 50);
            p.rect(0, 0, end, barWidth);
        };

        const drawTargets = () => {
            drawTarget(targetA.x, targetA.y, trans ? targetA.size : $slugShrinkerStore);
            if (show) drawTarget(targetB.x, targetB.y, trans ? targetB.size : $slugShrinkerStoreB);
        };

        p.setup = async () => {
            p.createCanvas(CANVAS_SIZE, CANVAS_SIZE);

            await nextLevel();
        };

        nextLevelP5 = () => {
            p.clear(0, 0, 0, 0);
            slugShrinkerStore.set(80);
            slugShrinkerStoreB.set(0);
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
                drawTargets();

                drawTrail();

                drawPointsBar();

                const letterColor = p.color(214, 255, 255, 255);
                p.fill(letterColor);
                p.textStyle(p.NORMAL);
                p.textSize(100);

                p.textAlign(p.CENTER, p.CENTER);

                p.textFont('Work Sans');

                const startMessage = statsTracker.totalBones;

                p.text(startMessage, p.width - 50, 80);
            } else if (state === 2) {
                drawEndMenu();
            }
        };
    };
</script>

<div class=" w-full h-full border-slate-500 border">
    <P5 {sketch} />
</div>
