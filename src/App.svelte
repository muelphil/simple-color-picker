<script lang="ts">

    const colorNames = {
        'rgb' : ['Red', 'Green', 'Blue'],
        'cmy' : ['Cyan', 'Magenta', 'Yellow']
    }

    const width = document.body.clientWidth //700;
    const height = document.body.clientHeight //500;
    const circleRadius = 0.2 * height;

    $: intensities = [255, 255, 255]
    let canvas: HTMLCanvasElement;
    let ctx;
    $: currentlyAnimating = false;

    const direction: Array<Array<number>> = [
        [Math.cos(5 / 6 * Math.PI), -Math.sin(5 / 6 * Math.PI)],
        [Math.cos(9 / 6 * Math.PI), -Math.sin(9 / 6 * Math.PI)],
        [Math.cos(Math.PI / 6), -Math.sin(Math.PI / 6)]
    ]
    const center = [width / 2,
        height / 2
        // circleRadius + (height-2*circleRadius) * Math.abs(direction[0][1]) / Math.abs(direction[0][1]-direction[1][1])
    ];
    console.log('percent=', Math.abs(direction[1][1]) / Math.abs(direction[0][1] - direction[1][1]))
    console.log('percent=', direction)

    function calculateColors(intensity1, intensity2, intensity3, mode: 'rgb' | 'cmy') {
        if (mode === 'rgb') {
            return [`rgb(${intensity1},0,0)`, `rgb(0,${intensity2},0)`, `rgb(0,0,${intensity3})`]
        } else {
            return [`rgb(${255 - intensity1},255,255)`, `rgb(255,${255 - intensity2},255)`, `rgb(255,255,${255 - intensity3})`]
        }
    }

    let mode: 'rgb' | 'cmy' = 'cmy';
    let colors = calculateColors(255, 255, 255, mode);
    const distanceFromCenterDisjoint = [2, 0.666 + (2 - 0.666) / (-Math.sin(3 / 2 * Math.PI) / Math.sin(5 / 6 * Math.PI)), 2];
    const distanceFromCenterOverlap = [0.666, 0.666, 0.666];
    let currentOverlapState: 'overlap' | 'disjoint' = 'overlap';
    let distanceFromCenter = [...(currentOverlapState === 'disjoint' ? distanceFromCenterDisjoint : distanceFromCenterOverlap)];

    function easeInOutQuad(x: number): number {
        return x < 0.5 ? 2 * x * x : 1 - Math.pow(-2 * x + 2, 2) / 2;
    }

    function animateDistanceFromCenter(totalDurationInMs = 3000) {
        currentlyAnimating = true;
        console.log('animateDistanceFromCenter started!');
        const stepsPerColor = 60;
        let currentColor = 0;
        let currentSteps = 0;
        const from = currentOverlapState == 'disjoint' ? distanceFromCenterDisjoint : distanceFromCenterOverlap;
        const to = currentOverlapState == 'disjoint' ? distanceFromCenterOverlap : distanceFromCenterDisjoint;
        console.log('disjoint=', distanceFromCenterDisjoint, 'overlap=', distanceFromCenterOverlap)
        console.log('from=', from, 'to=', to)
        // console.log('overlap state=',currentOverlapState)
        currentOverlapState = currentOverlapState == 'disjoint' ? 'overlap' : 'disjoint'
        const colorAnimationOrder = [1, 0, 2]
        // console.log('overlap state=',currentOverlapState)

        const handler = setInterval(() => {
            // console.log('doing stuff!', to[currentColor] - from[currentColor])
            const animatedCol = colorAnimationOrder[currentColor]
            distanceFromCenter[animatedCol] = from[animatedCol] + easeInOutQuad(currentSteps / stepsPerColor) * (to[animatedCol] - from[animatedCol]);
            // console.log(distanceFromCenter);
            draw();
            currentSteps++;
            if (currentSteps === stepsPerColor) {
                if (currentColor == 2) {
                    clearInterval(handler);
                    currentlyAnimating = false;
                    console.log('animateDistanceFromCenter finished!');
                } else {
                    currentSteps = 0;
                    currentColor++;
                }
            }
        }, totalDurationInMs / stepsPerColor / 3)
    }

    function draw() {
        // console.log(colors)
        if (ctx) {
            ctx.globalCompositeOperation = 'source-over';//"lighter"; // multiply
            ctx.fillStyle = mode === 'rgb' ? 'black' : 'white';
            ctx.fillRect(0, 0, width, height);
            ctx.globalCompositeOperation = mode === 'rgb' ? 'screen' : 'multiply';

            for (let color of [0, 1, 2]) {
                ctx.beginPath();
                ctx.fillStyle = colors[color];
                ctx.arc(
                    center[0] + direction[color][0] * circleRadius * distanceFromCenter[color],
                    center[1] + direction[color][1] * circleRadius * distanceFromCenter[color],
                    circleRadius, 0, 2 * Math.PI)
                ctx.fill();
            }
            // ctx.fillStyle = "rgb(255,0,0)";
            // ctx.fillRect(center[0], center[1], 10, 10);
        }
    }

    function updateColors() {
        colors = calculateColors(intensities[0], intensities[1], intensities[2], mode)
        draw();
    }

    function swapModes() {
        // currentOverlapState = 'disjoint'
        // distanceFromCenter = [...distanceFromCenterDisjoint];
        mode = mode === 'rgb' ? 'cmy' : 'rgb';
        intensities = [255, 255, 255]
        updateColors();
    }

    window.onload = () => {
        canvas = document.getElementById("tutorial")! as HTMLCanvasElement;
        ctx = canvas.getContext("2d")!;
        draw();
    }
</script>

<canvas id="tutorial" width="{width}" height="{height}" style="position:absolute; top:0; left:0;"></canvas>
<main style="position:absolute; top:30%; left:0;z-index: 10; color: {mode == 'rgb' ? 'white' : 'black'}">


    {#each {length:3} as _, i}
    <div>
        <span style="width:100px; display:inline-block;font-weight: bold">{colorNames[mode][i]}</span>
        <input type="range" min="0" max="255" bind:value={intensities[i]} on:input={updateColors}>
        <span style="width:100px; display:inline-block"> {intensities[i]}</span>
    </div>
    {/each}
    <div style="height: 200px; border:1px solid gray; background-color: {mode === 'rgb' ? `rgb(${intensities.join(',')})` : `rgb(${intensities.map(e => 255-e).join(',')})`}; margin: 16px 0"></div>
    <div>
        <button disabled="{currentlyAnimating}" on:click={() => animateDistanceFromCenter()}>{currentOverlapState === 'disjoint' ? 'join colors' : 'separate colors'}</button>
        <button on:click={() => swapModes()}>change to {mode == 'rgb' ? 'CMY' : 'RGB'}</button>
    </div>
</main>

<style>


    h2 {
        margin-top: 16px !important;
    }

    h1 {
        margin: 10px 0;
    }

    .bwimage {
        display: grid;
        grid-template-columns: repeat(var(--image-width, 8), var(--pixel-size, 40px));
        grid-template-rows: repeat(var(--image-height, 8), var(--pixel-size, 40px));
    }

    .bwimage.spaced {
        min-height: calc(var(--pixel-size, 40px) * 12);
        min-width: calc(var(--pixel-size, 40px) * 12);
    }

    .bwimage div {
        /*width:30px;*/
        /*height:20px;*/
        border: calc(var(--pixel-size, 40px) / 20) solid #c5c5c5;
        cursor: pointer;
        user-select: none;
        color: lightgray;
        text-align: end;
        font-size: small;
    }

    .bwimage div:hover {
        border-color: #646cff;
    }

    .updownbuttons {
        display: flex;
        align-items: center;
        flex-direction: column;
    }

    .updownbuttons button {
        flex: 1 1 auto;
        width: 100%;
    }

    .saved-images {

    }
</style>
