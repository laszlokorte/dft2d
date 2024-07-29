<script>
	import {untrack, tick} from 'svelte'

	let splitView = $state(false)
	let polar = $state(true)

	const w = 16
	const h = 16

	let image = $state({
		width: w,
		height: h,
		mags: Array(w*h).fill(0),
		phases: Array(w*h).fill(0),
	})

	let spectrum = $state({
		width: w,
		height: h,
		mags: Array(w*h).fill(0),
		phases: Array(w*h).fill(0),
	})

	let spatialFocus = $state({x:0,y:0})
	let frequencyFocus = $state({x:0,y:0})


	const rows = $derived(Array(image.height).fill(0).map((x,i) => i))
	const columns = $derived(Array(image.width).fill(0).map((x,i) => i))
	const viewBox = `-0.2 -0.2 ${image.width+0.4} ${image.height+0.4}`


	function complMul(magA, phaseA, magB, phaseB) {
		return {
			mag: magA*magB,
			phase: phaseA+phaseB
		}
	}

	function complSum(magA, phaseA, magB, phaseB) {
		const reA = magA*Math.cos(phaseA)
		const imA = magA*Math.sin(phaseA)
		const reB = magB*Math.cos(phaseB)
		const imB = magB*Math.sin(phaseB)

		const reSum = reA+reB
		const imSum = imA+imB

		return {
			mag: Math.hypot(reSum, imSum),
			phase: Math.atan2(imSum, reSum),
		}
	}


	function fft2into(w ,h, magsIn, phasesIn, dir) {
		const accumMag = Array(w*h).fill(0)
		const accumPhase = Array(w*h).fill(0)
		const magsOut = Array(w*h).fill(0)
		const phasesOut = Array(w*h).fill(0)

		const wnorm = dir === 1 ? 1 : 1/Math.sqrt(w)
		const hnorm = dir === 1 ? 1 : 1/Math.sqrt(h)

		for (let y=0;y<h;y++) {
			for (let u=0;u<w;u++) {
				for (let x=0;x<w;x++) {
					const prod = complMul(magsIn[y*w+x], phasesIn[y*w+x], 1, -2*dir*Math.PI*x*(u)/w)
					const sum = complSum(prod.mag * wnorm, prod.phase, accumMag[y*w+u], accumPhase[y*w+u])
					accumMag[y*w+u] = sum.mag
					accumPhase[y*w+u] = sum.phase
				}
			}
		}

		for (let x=0;x<h;x++) {
			for (let v=0;v<h;v++) {
				magsOut[v*w+x] = 0
				phasesOut[v*w+x] = 0
				for (let y=0;y<h;y++) {
					const prod = complMul(accumMag[y*w+x], accumPhase[y*w+x], 1, -2*dir*Math.PI*y*(v)/h)
					const sum = complSum(prod.mag * hnorm, prod.phase, magsOut[v*w+x], phasesOut[v*w+x])
					magsOut[v*w+x] = sum.mag
					phasesOut[v*w+x] = sum.phase
				}
			}
		}
		return {
			magsOut: magsOut, phasesOut: phasesOut
		}
	}

	function recalculate(to, from, dir=1) {
		const {magsOut: m,phasesOut: p} = fft2into(from.width, from.height, from.mags, from.phases, dir)

		to.mags = m
		to.phases = p

	}



	function clipPi(angle) {
		while(angle > Math.PI) {
			angle-=Math.PI*2
		}
		while(angle < -Math.PI) {
			angle+=Math.PI*2
		}

		return angle
	}
</script>


<h1>2D DFT</h1>

<fieldset>
	<legend>
		Options
	</legend>

	<div>
		<label><input type="radio" bind:group={polar} value={true}> Polar</label>
	<label><input type="radio" bind:group={polar} value={false}> Cartesian</label>
	</div>


	<div>
		<label><input type="checkbox" bind:checked={splitView}> Split {polar?"Magnitude/Phase":"Real/Imaginary"} </label>
	</div>
</fieldset>

<div style="display: grid; grid-template-columns: 1fr 1fr; justify-content: center; justify-items: stretch; max-width: 60em; margin: auto;">
<div class="domain">
<h2>Spacial Domain</h2>




{@render showImage(image, spatialFocus, ({x,y}) => spatialFocus = ({x,y}))}


{@render slider(image, spectrum, spatialFocus, 1)}

</div>

<div class="domain">
<h2>Frequency Domain</h2>

{@render showImage(spectrum, frequencyFocus, ({x,y}) => frequencyFocus = ({x,y}))}



{@render slider(spectrum, image, frequencyFocus, -1)}

</div>
</div>


{#snippet slider(img, other, focus, dir)}
{#if spatialFocus}
<fieldset>
	<legend>Modify</legend>

	{#if polar}
	<label>
		<span>Magnitude:</span>
	<input type="range" min="0" max="1" step="0.01" bind:value={img.mags[focus.y*img.width+focus.x]} oninput={e => recalculate(other, img, dir)} /></label>
	<label>
		<span>Phase:</span>
	<input type="range" min="-3.14" max="3.14" bind:value={img.phases[focus.y*img.width+focus.x]} step="0.01"  oninput={e => recalculate(other, img, dir)}/> </label>
	{/if}
</fieldset>
{/if}
{/snippet}

{#snippet showImage(img, focus, onclick)}
{@const max = Math.max(0.1, ...image.mags) || 1}
{#if polar}
{#if splitView}
<div style="display: flex;">
	<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl(0,0%,{100*img.mags[y*img.width+x]}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x,y})}
			></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>

<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl({180+clipPi(img.phases[y*img.width+x])/Math.PI*180},100%,45%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x,y})}></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>
</div>
{:else}

<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			<path fill="hsl(0,0%,{100*img.mags[y*img.width+x]}%)" stroke="none"
			d="m{x} {y} h 1 v 1 h-0.4 l -0.6,-0.6 z"pointer-events="none"
			></path>
			<path fill="hsl({180+clipPi(img.phases[y*img.width+x])/Math.PI*180},100%,45%)" stroke="none"
			d="m{x} {y} m0,0.4 v 0.6 h 0.6 z"pointer-events="none"
			></path>

			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" stroke="gray" stroke-width="0.1" fill="none"
			onclick={e => onclick({x,y})} pointer-events="all"
			></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2"  />
	{/if}
</svg>
{/if}
{:else}
{#if splitView}
<div style="display: flex;">
	<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			{@const re = img.mags[y*img.width+x] * Math.cos(img.phases[y*img.width+x])}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl(0,0%,{50+50*re}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x,y})}
			></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>

<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			{@const im = img.mags[y*img.width+x] * Math.sin(img.phases[y*img.width+x])}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl(0,0%,{50+50*im}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x,y})}></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>
</div>
{:else}

<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			{@const re = img.mags[y*img.width+x] * Math.cos(img.phases[y*img.width+x])}

			{@const im = img.mags[y*img.width+x] * Math.sin(img.phases[y*img.width+x])}
			<path fill="hsl(0,0%,{50+50*re}%)" stroke="none"
			d="m{x} {y} h 1 v 1 z"pointer-events="none"
			></path>
			<path fill="hsl(0,0%,{50+50*im}%)" stroke="none"
			d="m{x} {y} v 1 h 1 z"pointer-events="none"
			></path>

			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" stroke="gray" stroke-width="0.1" fill="none"
			onclick={e => onclick({x,y})} pointer-events="all"
			></rect>
		{/each}
	{/each}

	{#if focus}
	<rect pointer-events="none" {...focus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2"  />
	{/if}
</svg>
{/if}
{/if}
{/snippet}

<style>
	svg {
		width: 100%;
	}

	rect {
		stroke-width: 2px;
		vector-effect: non-scaling-stroke;
	}

	.focus {
		stroke-width: 4px;
	}

	label span {
		display: block;
	}

	h2 {
		text-align: center;
	}

	.domain {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	fieldset {
		align-self: stretch;
	}
</style>