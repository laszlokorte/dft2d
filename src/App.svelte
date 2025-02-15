<script>
	import {untrack, tick} from 'svelte'

	let splitView = $state(true)
	let coordinates = $state("polar")
	let polar = $derived(coordinates === "polar")

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

	let focus = $state({type:'spatial', x:w/2,y:h/2})


	const rows = $derived(Array(image.height).fill(0).map((x,i) => (i + image.height/2)%image.height))
	const columns = $derived(Array(image.width).fill(0).map((x,i) => (i + image.width/2)%image.width))
	const viewBox = `-0.2 -0.2 ${image.width+0.4} ${image.height+0.4}`

	function setPhase(subject, value) {
		let w = subject.width
		let h = subject.height

		for (let y=0;y<h;y++) {
			for (let x=0;x<w;x++) {
				subject.phases[y*w+x] = value
			}
		}
	}

	function setIm(subject, value) {
		let w = subject.width
		let h = subject.height

		for (let y=0;y<h;y++) {
			for (let x=0;x<w;x++) {
				const mag = subject.mags[y*w+x]
				const pha = subject.phases[y*w+x]

				const re = mag * Math.cos(pha)
				const im = mag * Math.sin(pha)

				const newRe = re;
				const newIm = value;


				const newMag = Math.hypot(newRe, newIm)
				const newPhase = Math.atan2(newIm, newRe)


				subject.mags[y*w+x] = newMag
				subject.phases[y*w+x] = newPhase

			}
		}
	}

	function setConjSymmetric(subject) {
		let w = subject.width
		let h = subject.height

		const newmags = Array(subject.mags.length).fill(0)
		const newphases = Array(subject.mags.length).fill(0)

		for (let y=1;y<h;y++) {
			for (let x=1;x<w;x++) {
				const magA = subject.mags[rows[y]*w+columns[x]]
				const phaA = subject.phases[rows[y]*w+columns[x]]


				const magB = subject.mags[rows[rows.length-y]*w+columns[columns.length-x]]
				const phaB = subject.phases[rows[rows.length-y]*w+columns[columns.length-x]]

				newmags[rows[rows.length-y]*w+columns[columns.length-x]] = Math.sqrt(magB*magB+magA*magA)/Math.sqrt(2)
				newphases[rows[rows.length-y]*w+columns[columns.length-x]] = clipPi(Math.sign(phaB-phaA) * Math.max(Math.abs(phaA),Math.abs(phaB)))

			}
		}

		newphases[rows[h/2]*w+columns[w/2]] = 0

		subject.phases = newphases
		subject.mags = newmags
	}

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


<h1>2D Discrete Fourier Transform</h1>

<fieldset style="width: max-content; padding: 1em 1em; margin: auto">
	<legend>
		Options
	</legend>

	<div>
		<label><input type="radio" bind:group={coordinates} value={"polar"}> Polar</label>
	<label><input type="radio" bind:group={coordinates} value={"cartesian"}> Cartesian</label>
	</div>


	<div>
		<label><input type="checkbox" bind:checked={splitView}> Split {polar?"Magnitude/Phase":"Real/Imaginary"} </label>
	</div>
</fieldset>

<div style="display: grid; grid-template-columns: 1fr 1fr; justify-content: center; justify-items: stretch; max-width: 70em; margin: auto;">
<div class="domain">
<h2>Spatial Domain</h2>

<fieldset>
	<legend>Adjustments</legend>

	<div>
		<button type="button" onclick={e => {setPhase(image, 0); recalculate(spectrum,  image, 1)}}>Set Phase to 0</button>
		<button type="button" onclick={e => {setIm(image, 0); recalculate(spectrum,  image, 1)}}>Set Im to 0</button>
		<button type="button" onclick={e => {setConjSymmetric(image); recalculate(spectrum,  image, 1)}}>Make Conj. Symmetric</button>
	</div>
</fieldset>


{@render showImage(image, focus.type=="spatial"?focus:null, ({x,y}) => focus = ({type:"spatial", x,y}))}


{@render slider(image, spectrum, focus.type=="spatial"?focus:null, 1)}

</div>

<div class="domain">
<h2>Frequency Domain</h2>

<fieldset>
	<legend>Adjustments</legend>

	<div>
		<button type="button" onclick={e => {setPhase(spectrum, 0);recalculate(image, spectrum, -1)}}>Set Phase to 0</button>
		<button type="button" onclick={e => {setIm(spectrum, 0);recalculate(image, spectrum, -1)}}>Set Im to 0</button>
		<button type="button" onclick={e => {setConjSymmetric(spectrum);recalculate(image, spectrum, -1)}}>Make Conj. Symmetric</button>
	</div>
</fieldset>

{@render showImage(spectrum, focus.type=="frequency"?focus:null, ({x,y}) => focus = ({type:"frequency",x,y}))}



{@render slider(spectrum, image, focus.type=="frequency"?focus:null, -1)}

</div>
</div>


{#snippet slider(img, other, focus, dir)}
{#if focus}

{@const re = img.mags[rows[focus.y]*img.width+columns[focus.x]] * Math.cos(img.phases[rows[focus.y]*img.width+columns[focus.x]])}
{@const im = img.mags[rows[focus.y]*img.width+columns[focus.x]] * Math.sin(img.phases[rows[focus.y]*img.width+columns[focus.x]])}
<fieldset>
	<legend>Modify Selected Pixel</legend>

	<label>
		<span>Magnitude: ({img.mags[rows[focus.y]*img.width+columns[focus.x]]})</span>
	<input type="range" min="0" max="1" step="0.01" bind:value={img.mags[rows[focus.y]*img.width+columns[focus.x]]} oninput={e => recalculate(other, img, dir)} /></label>
	<label>
		<span>Phase: ({img.phases[rows[focus.y]*img.width+columns[focus.x]]})</span>
	<input type="range" min="-3.14" max="3.14" bind:value={img.phases[rows[focus.y]*img.width+columns[focus.x]]} step="0.01"  oninput={e => recalculate(other, img, dir)}/> </label>

	<hr>

	<label>
		<span>Real:</span>
	<input type="range" min="-1" max="1" step="0.01" value={re} oninput={e => {
		const newMag = Math.hypot(e.currentTarget.valueAsNumber, im)
		const newPhase = Math.atan2(im, e.currentTarget.valueAsNumber)

		img.mags[rows[focus.y]*img.width+columns[focus.x]] = newMag
		img.phases[rows[focus.y]*img.width+columns[focus.x]] = newPhase

		recalculate(other, img, dir)

	}} /></label>
	<label>
		<span>Imaginary:</span>
	<input type="range" min="-1" max="1" value={im} step="0.01"  oninput={e => {
		const newMag = Math.hypot(e.currentTarget.valueAsNumber, re)
		const newPhase = Math.atan2(e.currentTarget.valueAsNumber, re)

		img.mags[rows[focus.y]*img.width+columns[focus.x]] = newMag
		img.phases[rows[focus.y]*img.width+columns[focus.x]] = newPhase

		recalculate(other, img, dir)
	}}/> </label>
</fieldset>
{/if}
{/snippet}

{#snippet showImage(img, focus, onclick)}
{@const max = Math.max(0.1, ...image.mags) || 1}
{#if polar}
{#if splitView}
<div class="domain-split">
	<svg {viewBox}>
		<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y, yi (yi)}
		{#each columns as x, xi (xi)}
			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" fill="hsl(0,0%,{100*img.mags[y*img.width+x]}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})}
			></rect>
		{/each}
	{/each}

		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>

<svg {viewBox}>
	<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y,yi (yi)}
		{#each columns as x,xi (xi)}
			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" fill="hsla({180+clipPi(img.phases[y*img.width+x])/Math.PI*180},100%,45%, {Math.abs(Math.sign(img.mags[y*img.width+x]))})" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})}></rect>
		{/each}
	{/each}

		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>
</div>
{:else}

<svg {viewBox}>
	<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y, yi (yi)}
		{#each columns as x, xi (xi)}
			<path fill="hsl(0,0%,{100*img.mags[y*img.width+x]}%)" stroke="none"
			d="m{xi} {yi} h 1 v 1 h-0.4 l -0.6,-0.6 z"pointer-events="none"
			></path>
			<path fill="hsla({180+clipPi(img.phases[y*img.width+x])/Math.PI*180},100%,45%, {Math.abs(Math.sign(img.mags[y*img.width+x]))})" stroke="none"
			d="m{xi} {yi} m0,0.4 v 0.6 h 0.6 z"pointer-events="none"
			></path>

			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" stroke="gray" stroke-width="0.1" fill="none"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})} pointer-events="all"
			></rect>
		{/each}
	{/each}

		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2"  />
	{/if}
</svg>
{/if}
{:else}
{#if splitView}
<div class="domain-split">
	<svg {viewBox}>
		<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y,yi (yi)}
		{#each columns as x,xi (xi)}
			{@const re = img.mags[y*img.width+x] * Math.cos(img.phases[y*img.width+x])}
			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" fill="hsl(0,0%,{50+50*re}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})}
			></rect>
		{/each}
	{/each}


		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}


</svg>

<svg {viewBox}>
	<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y,yi (yi)}
		{#each columns as x,xi (xi)}
			{@const im = img.mags[y*img.width+x] * Math.sin(img.phases[y*img.width+x])}
			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" fill="hsl(0,0%,{50+50*im}%)" stroke="gray" stroke-width="0.1"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})}></rect>
		{/each}
	{/each}

		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2" />
	{/if}
</svg>
</div>
{:else}

<svg {viewBox}>
	<rect x="0" y="0" width={img.width} height={img.height} fill="black" />
	{#each rows as y,yi (yi)}
		{#each columns as x,xi (xi)}
			{@const re = img.mags[y*img.width+x] * Math.cos(img.phases[y*img.width+x])}

			{@const im = img.mags[y*img.width+x] * Math.sin(img.phases[y*img.width+x])}
			<path fill="hsl(0,0%,{50+50*re}%)" stroke="none"
			d="m{xi} {yi} h 1 v 1 h-1 z"pointer-events="none"
			></path>
			<path fill="hsl(0,0%,{50+50*im}%)" stroke="none"
			d="m{xi} {yi} v 1 h 1 z"pointer-events="none"
			></path>

			<rect tabindex="-1" role="button" x={xi} y={yi} width="1" height="1" stroke="gray" stroke-width="0.1" fill="none"
			onclick={e => onclick({x:xi,y:yi})} onmouseenter={e => e.buttons===1 && onclick({x:xi,y:yi})} pointer-events="all"
			></rect>
		{/each}
	{/each}

		<circle pointer-events="none" fill="none" r="0.2" stroke="cyan" stroke-width="0.1" cx={columns[0]+0.5} cy={rows[0]+0.5}/>


	{#if focus}
	<rect pointer-events="none" x={focus.x} y={focus.y} width="1" height="1"  stroke="magenta" fill="none" class="focus" rx="0.2" ry="0.2"  />
	{/if}
</svg>
{/if}
{/if}
{/snippet}

<style>
	* {
		font-family: monospace;
	}

	svg {
		width: 100%;
	}

	rect {
		stroke-width: 1px;
		stroke-linejoin: none;
		stroke-linecap: none;
		vector-effect: non-scaling-stroke;
	}

	.focus {
		stroke-width: 2px;
		vector-effect: non-scaling-stroke;
	}

	label span {
		display: block;
	}

	h1 {
		text-align: center;
	}
	h2 {
		text-align: center;
	}

	.domain {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 0.3em;
		gap: 0.5em;
	}

	.domain-split {
		display: flex;
		gap: 2px;
	}

	fieldset {
		align-self: stretch;
		border: 1px solid #444;
	}

	hr {
		border: none;
		border-bottom: 1px solid #444;
		margin: 1em 0 1em;
		height: 0;
		padding: 0;
	}

	legend {
		background: #444;
		color: #fff;
		padding: 2px 6px;
		display: inline-blockl;
	}

	input[type=range] {
		width: 100%;
	}
</style>