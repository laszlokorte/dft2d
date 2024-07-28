<script>
	import {untrack} from 'svelte'

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
	let frequencyFocus = $state({u:0,v:0})


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


	function fft2into(w ,h, magsIn, phasesIn) {
		const accumMag = Array(w*h).fill(0)
		const accumPhase = Array(w*h).fill(0)
		const magsOut = Array(w*h).fill(0)
		const phasesOut = Array(w*h).fill(0)

		for (let y=0;y<h;y++) {
			for (let u=0;u<w;u++) {
				for (let x=0;x<w;x++) {
					const prod = complMul(magsIn[y*w+x], phasesIn[y*w+x], 1, -2*Math.PI*x*(u)/w)
					const sum = complSum(prod.mag, prod.phase, accumMag[y*w+u], accumPhase[y*w+u])
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
					const prod = complMul(accumMag[y*w+x], accumPhase[y*w+x], 1, -2*Math.PI*y*(v)/h)
					const sum = complSum(prod.mag, prod.phase, magsOut[v*w+x], phasesOut[v*w+x])
					magsOut[v*w+x] = sum.mag
					phasesOut[v*w+x] = sum.phase
				}
			}
		}
		return {
			magsOut: magsOut, phasesOut: phasesOut
		}
	}

	$effect(() => {
		const newImage = image
		const sum = newImage.phases.reduce((x,y) => x+y, 0) + newImage.mags.reduce((x,y) => x+y, 0)

		untrack(() => {
			const {magsOut: m,phasesOut: p} = fft2into(newImage.width, newImage.height, newImage.mags, newImage.phases)

			spectrum.mags = m
			spectrum.phases = p
		})

	})
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
<div>
<h2>Spacial Domain</h2>
<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl(0,0%,{100*image.mags[y*image.width+x]}%)" stroke="gray" stroke-width="0.1"
			onclick={() => spatialFocus = ({x,y})}
			></rect>
		{/each}
	{/each}

	{#if spatialFocus}
	<rect pointer-events="none" {...spatialFocus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>

<svg {viewBox}>
	{#each rows as y (y)}
		{#each columns as x (x)}
			<rect tabindex="-1" role="button" {x} {y} width="1" height="1" fill="hsl({180+clipPi(image.phases[y*image.width+x])/Math.PI*180},100%,45%)" stroke="gray" stroke-width="0.1"
			onclick={() => spatialFocus = ({x,y})}></rect>
		{/each}
	{/each}

	{#if spatialFocus}
	<rect pointer-events="none" {...spatialFocus} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>



{#if spatialFocus}
<div>
	<label>
		<span>Magnitude:</span>
	<input type="range" min="0" max="1" step="0.01" bind:value={image.mags[spatialFocus.y*image.width+spatialFocus.x]} /></label>
	<label>
		<span>Phase:</span>
	<input type="range" min="-3.14" max="3.14" bind:value={image.phases[spatialFocus.y*image.width+spatialFocus.x]} step="0.01" /> </label>
</div>
{/if}

</div>

<div>
<h2>Frequency Domain</h2>
<svg {viewBox}>
	{#each rows as u (u)}
		{#each columns as v (v)}
			<rect tabindex="-1" role="button" x={v} y={u} width="1" height="1" fill="hsl(0,0%,{100*spectrum.mags[u*image.width+v]}%)" stroke="gray" stroke-width="0.1"
			onclick={() => frequencyFocus = ({v,u})}
			></rect>
		{/each}
	{/each}

	{#if frequencyFocus}
	<rect pointer-events="none" x={frequencyFocus.v} y={frequencyFocus.u} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>

<svg {viewBox}>
	{#each rows as u (u)}
		{#each columns as v (v)}
			<rect tabindex="-1" role="button" x={v} y={u} width="1" height="1" fill="hsl({180+clipPi(spectrum.phases[u*spectrum.width+v])/Math.PI*180},100%,45%)" stroke="gray" stroke-width="0.1"
			onclick={() => frequencyFocus = ({v,u})}></rect>
		{/each}
	{/each}

	{#if frequencyFocus}
	<rect pointer-events="none" x={frequencyFocus.v} y={frequencyFocus.u} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>


{#if frequencyFocus}
<div>
	<label>
		<span>Magnitude:</span>
	<input type="range" min="0" max="1" step="0.01" bind:value={spectrum.mags[frequencyFocus.u*spectrum.width+frequencyFocus.v]} /></label>
	<label>
		<span>Phase:</span>
	<input type="range" min="-3.14" max="3.14" bind:value={spectrum.phases[frequencyFocus.u*spectrum.width+frequencyFocus.v]} step="0.01" /> </label>
</div>
{/if}
</div>


<style>
	svg {
		max-width: 25em;
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
</style>