<script>
	import { untrack, tick, onMount } from "svelte";

	import pokemon from "./pokemon.png";

	const numf = new Intl.NumberFormat("en-US", {
		maximumFractionDigits: 2,
		minimumFractionDigits: 2,
		signDisplay: "exceptZero",
	});

	let splitView = $state(true);
	let keepSymmetric = $state(true);
	let coordinates = $state("polar");
	let sliderStyle = $state("plane");
	let examples = $state("plane");
	let polar = $derived(coordinates === "polar");

	onMount(() => {
		const exs = [];
		const poke = new Image();
		poke.onload = (evt) => {
			const cnv = document.createElement("canvas");
			cnv.width = poke.width;
			cnv.height = poke.height;
			const cnx = cnv.getContext("2d");
			cnx.drawImage(poke, 0, 0);

			const width = poke.width;
			const height = poke.height;

			for (let row = 0; row < height; row += 166) {
				for (let col = 0; col < width; col += 166) {
					const example = {
						mags: Array(16 * 16).fill(0),
						phases: Array(16 * 16).fill(0),
					};
					const imgPixels = cnx.getImageData(col, row, 160, 160);

					for (let y = 0; y < 16; y += 1) {
						for (let x = 0; x < 16; x += 1) {
							const p = x * 4 * 10 + y * 4 * 10 * 10 * 16;
							const ri = p;
							const gi = p + 1;
							const bi = p + 2;
							example.mags[((x + 8) % 16) + ((y + 8) % 16) * 16] =
								(imgPixels.data[ri] +
									imgPixels.data[gi] +
									imgPixels.data[bi]) /
								3 /
								255;
						}
					}

					exs.push(example);

					if (exs.length > 5) {
						break;
					}
				}
				if (exs.length > 5) {
					break;
				}
			}

			examples = exs;
		};
		poke.src = pokemon;
	});

	const w = 16;
	const h = 16;

	let image = $state({
		width: w,
		height: h,
		mags: Array(w * h).fill(0),
		phases: Array(w * h).fill(0),
	});

	let spectrum = $state({
		width: w,
		height: h,
		mags: Array(w * h).fill(0),
		phases: Array(w * h).fill(0),
	});

	let focus = $state({ type: "spatial", x: w / 2, y: h / 2 });

	const rows = $derived(
		Array(image.height)
			.fill(0)
			.map((x, i) => (i + image.height / 2) % image.height),
	);
	const columns = $derived(
		Array(image.width)
			.fill(0)
			.map((x, i) => (i + image.width / 2) % image.width),
	);
	const viewBox = `-0.2 -0.2 ${image.width + 0.4} ${image.height + 0.4}`;

	function loadExample(example) {
		image.mags = example.mags;
		image.phases = example.phases;

		recalculate(spectrum, image, 1);
	}

	function setPolar(subject, x, y, newMag, newPhase, sym = false) {
		subject.mags[rows[y] * subject.width + columns[x]] = newMag;
		subject.phases[rows[y] * subject.width + columns[x]] = newPhase;

		if (
			sym &&
			rows[subject.height - y] != rows[y] &&
			rows[subject.width - x] != rows[x]
		) {
			subject.mags[
				rows[subject.height - y] * subject.width +
					columns[subject.width - x]
			] = newMag;
			subject.phases[
				rows[subject.height - y] * subject.width +
					columns[subject.width - x]
			] = -newPhase;
		}
	}

	function setGlobalPolar(subject, mag, phase) {
		let w = subject.width;
		let h = subject.height;

		for (let y = 0; y <= h; y++) {
			for (let x = 0; x <= w; x++) {
				subject.phases[y * w + x] = phase;
				subject.mags[y * w + x] = mag;
			}
		}
	}

	function setGlobalPhase(subject, value) {
		let w = subject.width;
		let h = subject.height;

		for (let y = 0; y <= h; y++) {
			for (let x = 0; x <= w; x++) {
				subject.phases[y * w + x] = value;
			}
		}
	}

	function setGlobalIm(subject, value) {
		let w = subject.width;
		let h = subject.height;

		for (let y = 0; y <= h; y++) {
			for (let x = 0; x <= w; x++) {
				const mag = subject.mags[y * w + x];
				const pha = subject.phases[y * w + x];

				const re = mag * Math.cos(pha);
				const im = mag * Math.sin(pha);

				const newRe = re;
				const newIm = value;

				const newMag = Math.hypot(newRe, newIm);
				const newPhase = Math.atan2(newIm, newRe);

				subject.mags[y * w + x] = newMag;
				subject.phases[y * w + x] = newPhase;
			}
		}
	}

	function setConjSymmetric(subject) {
		let w = subject.width;
		let h = subject.height;

		const newmags = Array(subject.mags.length).fill(0);
		const newphases = Array(subject.mags.length).fill(0);

		for (let y = 1; y < h; y++) {
			for (let x = 1; x < w; x++) {
				const magA = subject.mags[rows[y] * w + columns[x]];
				const phaA = subject.phases[rows[y] * w + columns[x]];

				const magB =
					subject.mags[
						rows[rows.length - y] * w + columns[columns.length - x]
					];
				const phaB =
					subject.phases[
						rows[rows.length - y] * w + columns[columns.length - x]
					];

				newmags[
					rows[rows.length - y] * w + columns[columns.length - x]
				] = Math.sqrt(magB * magB + magA * magA) / Math.sqrt(2);
				newphases[
					rows[rows.length - y] * w + columns[columns.length - x]
				] = clipPi(
					Math.sign(phaB - phaA) *
						Math.max(Math.abs(phaA), Math.abs(phaB)),
				);
			}
		}

		newphases[rows[h / 2] * w + columns[w / 2]] = 0;

		subject.phases = newphases;
		subject.mags = newmags;
	}

	function complMul(magA, phaseA, magB, phaseB) {
		return {
			mag: magA * magB,
			phase: phaseA + phaseB,
		};
	}

	function complSum(magA, phaseA, magB, phaseB) {
		const reA = magA * Math.cos(phaseA);
		const imA = magA * Math.sin(phaseA);
		const reB = magB * Math.cos(phaseB);
		const imB = magB * Math.sin(phaseB);

		const reSum = reA + reB;
		const imSum = imA + imB;

		return {
			mag: Math.hypot(reSum, imSum),
			phase: Math.atan2(imSum, reSum),
		};
	}

	function fft2into(w, h, magsIn, phasesIn, dir) {
		const accumMag = Array(w * h).fill(0);
		const accumPhase = Array(w * h).fill(0);
		const magsOut = Array(w * h).fill(0);
		const phasesOut = Array(w * h).fill(0);

		const wnorm = dir === 1 ? 1 : 1 / Math.sqrt(w);
		const hnorm = dir === 1 ? 1 : 1 / Math.sqrt(h);

		for (let y = 0; y < h; y++) {
			for (let u = 0; u < w; u++) {
				for (let x = 0; x < w; x++) {
					const prod = complMul(
						magsIn[y * w + x],
						phasesIn[y * w + x],
						1,
						(-2 * dir * Math.PI * x * u) / w,
					);
					const sum = complSum(
						prod.mag * wnorm,
						prod.phase,
						accumMag[y * w + u],
						accumPhase[y * w + u],
					);
					accumMag[y * w + u] = sum.mag;
					accumPhase[y * w + u] = sum.phase;
				}
			}
		}

		for (let x = 0; x < h; x++) {
			for (let v = 0; v < h; v++) {
				magsOut[v * w + x] = 0;
				phasesOut[v * w + x] = 0;
				for (let y = 0; y < h; y++) {
					const prod = complMul(
						accumMag[y * w + x],
						accumPhase[y * w + x],
						1,
						(-2 * dir * Math.PI * y * v) / h,
					);
					const sum = complSum(
						prod.mag * hnorm,
						prod.phase,
						magsOut[v * w + x],
						phasesOut[v * w + x],
					);
					magsOut[v * w + x] = sum.mag;
					phasesOut[v * w + x] = sum.phase;
				}
			}
		}
		return {
			magsOut: magsOut,
			phasesOut: phasesOut,
		};
	}

	function recalculate(to, from, dir = 1) {
		const { magsOut: m, phasesOut: p } = fft2into(
			from.width,
			from.height,
			from.mags,
			from.phases,
			dir,
		);

		to.mags = m;
		to.phases = p;
	}

	function clipPi(angle) {
		while (angle > Math.PI) {
			angle -= Math.PI * 2;
		}
		while (angle < -Math.PI) {
			angle += Math.PI * 2;
		}

		return angle;
	}
</script>

<h1>2D Discrete Fourier Transform</h1>

<fieldset class="options">
	<legend> Options </legend>

	<div class="radio-list">
		<span class="radio-list-head">Coordinate System</span>
		<label class="radio-field"
			><input
				type="radio"
				bind:group={coordinates}
				value={"polar"}
				class="radio-control"
			/>
			<span class="radio-label">Polar</span></label
		>
		<label class="radio-field"
			><input
				type="radio"
				bind:group={coordinates}
				value={"cartesian"}
				class="radio-control"
			/>
			<span class="radio-label">Cartesian</span></label
		>

		<label class="checkbox-field"
			><input
				type="checkbox"
				bind:checked={splitView}
				class="checkbox-control"
			/>
			<span class="checkbox-label"
				>Split {polar ? "Magnitude/Phase" : "Real/Imaginary"}</span
			>
		</label>
	</div>

	<div>
		<select
			onchange={(evt) => {
				if (evt.currentTarget.value) {
					loadExample(examples[evt.currentTarget.value]);
				}
			}}
		>
			<option value="">Load Example</option>
			{#each examples as ex, i (i)}
				<option value={i}>Example #{i}</option>
			{/each}
		</select>
	</div>
</fieldset>

<div class="domain-container">
	<div class="domain">
		<h2 class="domain-name">Spatial Domain</h2>

		<fieldset>
			<legend>Global Adjustments</legend>

			<div class="button-row">
				<button
					type="button"
					onclick={(e) => {
						setGlobalPolar(image, 0, 0);
						recalculate(spectrum, image, 1);
					}}>Set All 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setGlobalPhase(image, 0);
						recalculate(spectrum, image, 1);
					}}>Set Phase to 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setGlobalIm(image, 0);
						recalculate(spectrum, image, 1);
					}}>Set Im to 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setConjSymmetric(image);
						recalculate(spectrum, image, 1);
					}}>Make Conj. Symmetric</button
				>
			</div>
		</fieldset>

		{@render showImage(
			image,
			focus.type == "spatial" ? focus : null,
			({ x, y }) => (focus = { type: "spatial", x, y }),
		)}

		{@render slider(
			image,
			spectrum,
			focus.type == "spatial" ? focus : null,
			1,
			() => {
				focus = {
					type: "spatial",
					x: spectrum.width - focus.x,
					y: spectrum.height - focus.y,
				};
			},
		)}
	</div>

	<div class="domain">
		<h2 class="domain-name">Frequency Domain</h2>

		<fieldset>
			<legend>Global Adjustments</legend>

			<div class="button-row">
				<button
					type="button"
					onclick={(e) => {
						setGlobalPolar(spectrum, 0, 0);
						recalculate(image, spectrum, -1);
					}}>Set All 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setGlobalPhase(spectrum, 0);
						recalculate(image, spectrum, -1);
					}}>Set Phase to 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setGlobalIm(spectrum, 0);
						recalculate(image, spectrum, -1);
					}}>Set Im to 0</button
				>
				<button
					type="button"
					onclick={(e) => {
						setConjSymmetric(spectrum);
						recalculate(image, spectrum, -1);
					}}>Make Conj. Symmetric</button
				>
			</div>
		</fieldset>

		{@render showImage(
			spectrum,
			focus.type == "frequency" ? focus : null,
			({ x, y }) => (focus = { type: "frequency", x, y }),
		)}

		{@render slider(
			spectrum,
			image,
			focus.type == "frequency" ? focus : null,
			-1,
			() => {
				focus = {
					type: "frequency",
					x: spectrum.width - focus.x,
					y: spectrum.height - focus.y,
				};
			},
		)}
	</div>
</div>

{#snippet slider(img, other, focus, dir, flipFocus)}
	{#if focus}
		{@const re =
			img.mags[rows[focus.y] * img.width + columns[focus.x]] *
			Math.cos(img.phases[rows[focus.y] * img.width + columns[focus.x]])}
		{@const im =
			img.mags[rows[focus.y] * img.width + columns[focus.x]] *
			Math.sin(img.phases[rows[focus.y] * img.width + columns[focus.x]])}
		<fieldset>
			<legend
				>Modify Selected Pixel
				<div class="radio-list">
					<label class="radio-field"
						><input
							type="radio"
							bind:group={sliderStyle}
							value={"range"}
							class="radio-control"
						/>
						<span class="radio-label">Slider</span></label
					>
					<label class="radio-field"
						><input
							type="radio"
							bind:group={sliderStyle}
							value={"plane"}
							class="radio-control"
						/>
						<span class="radio-label">Plane</span></label
					>
				</div>
			</legend>

			<div class="radio-list">
				<label class="radio-field"
					><input
						type="checkbox"
						bind:checked={keepSymmetric}
						class="radio-control"
					/>
					<span class="radio-label">Keep Symmetric</span></label
				>
			</div>
			{#if sliderStyle == "range"}
				<div
					class="number-slider"
					style:--current-value={img.mags[
						rows[focus.y] * img.width + columns[focus.x]
					]}
				>
					<label for="magnitude-{dir}"
						>Magnitude: ({numf.format(
							img.mags[
								rows[focus.y] * img.width + columns[focus.x]
							],
						)})</label
					>
					<button
						type="button"
						onclick={(e) => {
							const el = e.currentTarget.nextElementSibling;
							el.value = 0;
							el?.dispatchEvent(
								new Event("input", { bubbles: true }),
							);
						}}>set to 0</button
					>

					<input
						id="magnitude-{dir}"
						class="mag-slider"
						type="range"
						min="0"
						max="1"
						step="0.01"
						bind:value={img.mags[
							rows[focus.y] * img.width + columns[focus.x]
						]}
						oninput={(e) => {
							if (keepSymmetric) {
								img.mags[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] =
									img.mags[
										rows[focus.y] * img.width +
											columns[focus.x]
									];
							}
							recalculate(other, img, dir);
						}}
					/>
				</div>
				<div
					class="number-slider"
					style:--current-value={img.phases[
						rows[focus.y] * img.width + columns[focus.x]
					]}
				>
					<label for="phase-{dir}"
						>Phase: ({numf.format(
							img.phases[
								rows[focus.y] * img.width + columns[focus.x]
							],
						)})</label
					>

					<button
						type="button"
						onclick={(e) => {
							const el = e.currentTarget.nextElementSibling;
							el.value = 0;
							el?.dispatchEvent(
								new Event("input", { bubbles: true }),
							);
						}}>set to 0</button
					>

					<input
						id="phase-{dir}"
						class="phase-slider"
						type="range"
						min="-3.14"
						max="3.14"
						bind:value={img.phases[
							rows[focus.y] * img.width + columns[focus.x]
						]}
						step="0.01"
						oninput={(e) => {
							if (keepSymmetric) {
								img.phases[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] =
									-img.phases[
										rows[focus.y] * img.width +
											columns[focus.x]
									];
							}
							recalculate(other, img, dir);
						}}
					/>
				</div>

				<hr />

				<div class="number-slider" style:--current-value={re}>
					<label for="real-{dir}">Real: ({numf.format(re)})</label>

					<button
						type="button"
						onclick={(e) => {
							const el = e.currentTarget.nextElementSibling;
							el.valueAsNumber = 0;
							el?.dispatchEvent(
								new Event("input", { bubbles: true }),
							);
						}}>set to 0</button
					>

					<input
						id="real-{dir}"
						type="range"
						class="real-slider"
						min="-1"
						max="1"
						step="0.01"
						value={re}
						oninput={(e) => {
							const newMag = Math.hypot(
								e.currentTarget.valueAsNumber,
								im,
							);
							const newPhase = Math.atan2(
								im,
								e.currentTarget.valueAsNumber,
							);

							img.mags[
								rows[focus.y] * img.width + columns[focus.x]
							] = newMag;
							img.phases[
								rows[focus.y] * img.width + columns[focus.x]
							] = newPhase;

							if (keepSymmetric) {
								img.mags[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] = newMag;
								img.phases[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] = -newPhase;
							}

							recalculate(other, img, dir);
						}}
					/>
				</div>
				<div class="number-slider" style:--current-value={im}>
					<label for="imaginary-{dir}"
						>Imaginary: ({numf.format(im)})</label
					>

					<button
						type="button"
						onclick={(e) => {
							const el = e.currentTarget.nextElementSibling;
							el.value = 0;
							el?.dispatchEvent(
								new Event("input", { bubbles: true }),
							);
						}}>set to 0</button
					>

					<input
						id="imaginary-{dir}"
						class="imag-slider"
						type="range"
						min="-1"
						max="1"
						value={im}
						step="0.01"
						oninput={(e) => {
							const newMag = Math.hypot(
								e.currentTarget.valueAsNumber,
								re,
							);
							const newPhase = Math.atan2(
								e.currentTarget.valueAsNumber,
								re,
							);

							img.mags[
								rows[focus.y] * img.width + columns[focus.x]
							] = newMag;
							img.phases[
								rows[focus.y] * img.width + columns[focus.x]
							] = newPhase;

							if (keepSymmetric) {
								img.mags[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] = newMag;
								img.phases[
									rows[img.height - focus.y] * img.width +
										columns[img.width - focus.x]
								] = -newPhase;
							}

							recalculate(other, img, dir);
						}}
					/>
				</div>
			{:else if sliderStyle == "plane"}
				<svg
					viewBox="-105 -105 210 210"
					class="polar-control"
					style:--value-phase={img.phases[
						rows[focus.y] * img.width + columns[focus.x]
					]}
				>
					<line
						class="polar-control-axis"
						x1="-100"
						x2="100"
						y1="0"
						y2="0"
						stroke="black"
						stroke-width="1"
					/>
					<line
						class="polar-control-axis"
						x1="0"
						x2="0"
						y1="-100"
						y2="100"
						stroke="black"
						stroke-width="1"
					/>
					<circle
						cx={0}
						cy={0}
						r="100"
						fill="none"
						class="polar-control-unit"
						stroke="#eee"
						stroke-width="3"
						stroke-dasharray="5 5"
					/>

					<path class="polar-control-tip" d="M103 0 l -5 -3 v 6 z " />
					<path
						class="polar-control-tip"
						d="M 0 -103 l  -3 5 h 6 z "
					/>

					<text
						text-anchor="end"
						x="95"
						y="-5"
						class="polar-control-axis-label">Real</text
					>
					<text
						text-anchor="start"
						x="5"
						y="-90"
						class="polar-control-axis-label">Imaginary</text
					>

					<line
						class="polar-control-marker-magnitude"
						x1="0"
						y1="0"
						x2={100 * re}
						y2={-100 * im}
						stroke="#aaa"
						stroke-width="1"
						stroke-dasharray="1 1"
					/>

					<line
						class="polar-control-marker-real"
						x1="0"
						y1={-100 * im}
						x2={100 * re}
						y2={-100 * im}
						stroke="#aaa"
						stroke-width="1"
						stroke-dasharray="1 1"
					/>

					<line
						class="polar-control-marker-imag"
						x1={100 * re}
						y1="0"
						x2={100 * re}
						y2={-100 * im}
						stroke="#aaa"
						stroke-width="1"
						stroke-dasharray="1 1"
					/>

					<text
						class:hidden={img.mags[
							rows[focus.y] * img.width + columns[focus.x]
						] < 0.25}
						text-anchor="middle"
						x={50 * re}
						y={-50 * im}
						class="polar-control-axis-value"
						>{numf.format(
							img.mags[
								rows[focus.y] * img.width + columns[focus.x]
							],
						)}</text
					>

					<text
						class:hidden={Math.abs(im) < 0.2}
						text-anchor={re > 0 ? "start" : "end"}
						x={100 * re + Math.sign(re) * 3}
						y={-50 * im}
						class="polar-control-axis-value">{numf.format(im)}</text
					>

					<text
						class:hidden={Math.abs(re) < 0.2}
						text-anchor="middle"
						x={50 * re}
						y={-100 * im + 3 - Math.sign(im) * 6}
						class="polar-control-axis-value">{numf.format(re)}</text
					>

					{#if keepSymmetric}
						<line
							class="polar-control-marker-magnitude"
							x1="0"
							y1="0"
							x2={100 * re}
							y2={-100 * im}
							stroke="#aaa"
							stroke-width="1"
							stroke-dasharray="1 1"
						/>

						<circle
							cx={100 * re}
							cy={100 * im}
							r="4"
							fill="DodgerBlue"
							cursor="move"
							stroke="white"
							stroke-width="3"
							class="polar-control-head-mirror"
							onclick={(e) => {
								flipFocus();
							}}
						/>
					{/if}

					<circle
						cx={100 * re}
						cy={-100 * im}
						r="5"
						fill="DodgerBlue"
						cursor="move"
						stroke="white"
						stroke-width="3"
						class="polar-control-head"
						onpointerdown={(evt) => {
							if (evt.isPrimary) {
								const svg = evt.currentTarget.ownerSVGElement;
								const pt = svg.createSVGPoint();

								pt.x = evt.currentTarget.getAttribute("cx");
								pt.y = evt.currentTarget.getAttribute("cy");
								const svgGlobal = pt.matrixTransform(
									svg.getScreenCTM(),
								);

								evt.currentTarget.setAttribute(
									"data-offset-x",
									evt.clientX - svgGlobal.x,
								);
								evt.currentTarget.setAttribute(
									"data-offset-y",
									evt.clientY - svgGlobal.y,
								);
								evt.currentTarget.setPointerCapture(
									evt.pointerId,
								);
							}
						}}
						onpointermove={(evt) => {
							if (
								evt.isPrimary &&
								evt.currentTarget.hasPointerCapture(
									evt.pointerId,
								)
							) {
								const offsetX = parseFloat(
									evt.currentTarget.getAttribute(
										"data-offset-x",
									) || "0",
								);
								const offsetY = parseFloat(
									evt.currentTarget.getAttribute(
										"data-offset-y",
									) || "0",
								);
								const svg = evt.currentTarget.ownerSVGElement;
								const pt = svg.createSVGPoint();

								pt.x = evt.clientX - offsetX;
								pt.y = evt.clientY - offsetY;
								const svgGlobal = pt.matrixTransform(
									svg.getScreenCTM().inverse(),
								);

								const newRe = svgGlobal.x / 100;
								const newIm = -svgGlobal.y / 100;

								const newMag = Math.hypot(newRe, newIm);
								const newPhase = Math.atan2(newIm, newRe);

								setPolar(
									img,
									focus.x,
									focus.y,
									newMag,
									newPhase,
									keepSymmetric,
								);
								recalculate(other, img, dir);
							}
						}}
					/>
				</svg>
			{/if}
		</fieldset>
	{/if}
{/snippet}

{#snippet showImage(img, focus, onclick)}
	{@const max = Math.max(0.1, ...image.mags) || 1}
	{#if polar}
		{#if splitView}
			<div class="domain-split">
				<figure class="plot">
					<svg class="plot-graphic" {viewBox}>
						<rect
							x="0"
							y="0"
							width={img.width}
							height={img.height}
							fill="black"
						/>
						{#each rows as y, yi (yi)}
							{#each columns as x, xi (xi)}
								<rect
									tabindex="-1"
									role="button"
									x={xi}
									y={yi}
									width="1"
									height="1"
									fill="hsl(0,0%,{100 *
										img.mags[y * img.width + x]}%)"
									stroke="gray"
									stroke-width="0.1"
									onclick={(e) => onclick({ x: xi, y: yi })}
									onmouseenter={(e) =>
										e.buttons === 1 &&
										onclick({ x: xi, y: yi })}
								></rect>
							{/each}
						{/each}

						<circle
							pointer-events="none"
							fill="none"
							r="0.2"
							stroke="cyan"
							stroke-width="0.1"
							cx={columns[0] + 0.5}
							cy={rows[0] + 0.5}
						/>

						{#if focus}
							<rect
								pointer-events="none"
								x={focus.x}
								y={focus.y}
								width="1"
								height="1"
								stroke="magenta"
								fill="none"
								class="focus"
								rx="0.2"
								ry="0.2"
							/>
						{/if}
					</svg>

					<figcaption class="plot-caption">Magnitude</figcaption>
				</figure>

				<figure class="plot">
					<svg class="plot-graphic" {viewBox}>
						<rect
							x="0"
							y="0"
							width={img.width}
							height={img.height}
							fill="black"
						/>
						{#each rows as y, yi (yi)}
							{#each columns as x, xi (xi)}
								<rect
									tabindex="-1"
									role="button"
									x={xi}
									y={yi}
									width="1"
									height="1"
									fill="hsla({180 +
										(clipPi(img.phases[y * img.width + x]) /
											Math.PI) *
											180},100%,45%)"
									stroke="gray"
									stroke-width="0.1"
									onclick={(e) => onclick({ x: xi, y: yi })}
									onmouseenter={(e) =>
										e.buttons === 1 &&
										onclick({ x: xi, y: yi })}
								></rect>
							{/each}
						{/each}

						<circle
							pointer-events="none"
							fill="none"
							r="0.2"
							stroke="cyan"
							stroke-width="0.1"
							cx={columns[0] + 0.5}
							cy={rows[0] + 0.5}
						/>

						{#if focus}
							<rect
								pointer-events="none"
								x={focus.x}
								y={focus.y}
								width="1"
								height="1"
								stroke="magenta"
								fill="none"
								class="focus"
								rx="0.2"
								ry="0.2"
							/>
						{/if}
					</svg>
					<figcaption class="plot-caption">Phase</figcaption>
				</figure>
			</div>
		{:else}
			<figure class="plot">
				<svg class="plot-graphic" {viewBox}>
					<rect
						x="0"
						y="0"
						width={img.width}
						height={img.height}
						fill="black"
					/>
					{#each rows as y, yi (yi)}
						{#each columns as x, xi (xi)}
							<path
								fill="hsl(0,0%,{100 *
									img.mags[y * img.width + x]}%)"
								stroke="none"
								d="m{xi} {yi} h 1 v 1 h-0.4 l -0.6,-0.6 z"
								pointer-events="none"
							></path>
							<path
								fill="hsla({180 +
									(clipPi(img.phases[y * img.width + x]) /
										Math.PI) *
										180},100%,45%, {Math.abs(
									Math.sign(img.mags[y * img.width + x]),
								)})"
								stroke="none"
								d="m{xi} {yi} m0,0.4 v 0.6 h 0.6 z"
								pointer-events="none"
							></path>

							<rect
								tabindex="-1"
								role="button"
								x={xi}
								y={yi}
								width="1"
								height="1"
								stroke="gray"
								stroke-width="0.1"
								fill="none"
								onclick={(e) => onclick({ x: xi, y: yi })}
								onmouseenter={(e) =>
									e.buttons === 1 &&
									onclick({ x: xi, y: yi })}
								pointer-events="all"
							></rect>
						{/each}
					{/each}

					<circle
						pointer-events="none"
						fill="none"
						r="0.2"
						stroke="cyan"
						stroke-width="0.1"
						cx={columns[0] + 0.5}
						cy={rows[0] + 0.5}
					/>

					{#if focus}
						<rect
							pointer-events="none"
							x={focus.x}
							y={focus.y}
							width="1"
							height="1"
							stroke="magenta"
							fill="none"
							class="focus"
							rx="0.2"
							ry="0.2"
						/>
					{/if}
				</svg>

				<figcaption class="plot-caption">Magnitude/Phase</figcaption>
			</figure>
		{/if}
	{:else if splitView}
		<div class="domain-split">
			<figure class="plot">
				<svg class="plot-graphic" {viewBox}>
					<rect
						x="0"
						y="0"
						width={img.width}
						height={img.height}
						fill="black"
					/>
					{#each rows as y, yi (yi)}
						{#each columns as x, xi (xi)}
							{@const re =
								img.mags[y * img.width + x] *
								Math.cos(img.phases[y * img.width + x])}
							<rect
								tabindex="-1"
								role="button"
								x={xi}
								y={yi}
								width="1"
								height="1"
								fill="hsl(0,0%,{50 + 50 * re}%)"
								stroke="gray"
								stroke-width="0.1"
								onclick={(e) => onclick({ x: xi, y: yi })}
								onmouseenter={(e) =>
									e.buttons === 1 &&
									onclick({ x: xi, y: yi })}
							></rect>
						{/each}
					{/each}

					<circle
						pointer-events="none"
						fill="none"
						r="0.2"
						stroke="cyan"
						stroke-width="0.1"
						cx={columns[0] + 0.5}
						cy={rows[0] + 0.5}
					/>

					{#if focus}
						<rect
							pointer-events="none"
							x={focus.x}
							y={focus.y}
							width="1"
							height="1"
							stroke="magenta"
							fill="none"
							class="focus"
							rx="0.2"
							ry="0.2"
						/>
					{/if}
				</svg>
				<figcaption class="plot-caption">Real</figcaption>
			</figure>

			<figure class="plot">
				<svg class="plot-graphic" {viewBox}>
					<rect
						x="0"
						y="0"
						width={img.width}
						height={img.height}
						fill="black"
					/>
					{#each rows as y, yi (yi)}
						{#each columns as x, xi (xi)}
							{@const im =
								img.mags[y * img.width + x] *
								Math.sin(img.phases[y * img.width + x])}
							<rect
								tabindex="-1"
								role="button"
								x={xi}
								y={yi}
								width="1"
								height="1"
								fill="hsl(0,0%,{50 + 50 * im}%)"
								stroke="gray"
								stroke-width="0.1"
								onclick={(e) => onclick({ x: xi, y: yi })}
								onmouseenter={(e) =>
									e.buttons === 1 &&
									onclick({ x: xi, y: yi })}
							></rect>
						{/each}
					{/each}

					<circle
						pointer-events="none"
						fill="none"
						r="0.2"
						stroke="cyan"
						stroke-width="0.1"
						cx={columns[0] + 0.5}
						cy={rows[0] + 0.5}
					/>

					{#if focus}
						<rect
							pointer-events="none"
							x={focus.x}
							y={focus.y}
							width="1"
							height="1"
							stroke="magenta"
							fill="none"
							class="focus"
							rx="0.2"
							ry="0.2"
						/>
					{/if}
				</svg>
				<figcaption class="plot-caption">Imaginary</figcaption>
			</figure>
		</div>
	{:else}
		<figure class="plot">
			<svg class="plot-graphic" {viewBox}>
				<rect
					x="0"
					y="0"
					width={img.width}
					height={img.height}
					fill="black"
				/>
				{#each rows as y, yi (yi)}
					{#each columns as x, xi (xi)}
						{@const re =
							img.mags[y * img.width + x] *
							Math.cos(img.phases[y * img.width + x])}

						{@const im =
							img.mags[y * img.width + x] *
							Math.sin(img.phases[y * img.width + x])}
						<path
							fill="hsl(0,0%,{50 + 50 * re}%)"
							stroke="none"
							d="m{xi} {yi} h 1 v 1 h-1 z"
							pointer-events="none"
						></path>
						<path
							fill="hsl(0,0%,{50 + 50 * im}%)"
							stroke="none"
							d="m{xi} {yi} v 1 h 1 z"
							pointer-events="none"
						></path>

						<rect
							tabindex="-1"
							role="button"
							x={xi}
							y={yi}
							width="1"
							height="1"
							stroke="gray"
							stroke-width="0.1"
							fill="none"
							onclick={(e) => onclick({ x: xi, y: yi })}
							onmouseenter={(e) =>
								e.buttons === 1 && onclick({ x: xi, y: yi })}
							pointer-events="all"
						></rect>
					{/each}
				{/each}

				<circle
					pointer-events="none"
					fill="none"
					r="0.2"
					stroke="cyan"
					stroke-width="0.1"
					cx={columns[0] + 0.5}
					cy={rows[0] + 0.5}
				/>

				{#if focus}
					<rect
						pointer-events="none"
						x={focus.x}
						y={focus.y}
						width="1"
						height="1"
						stroke="magenta"
						fill="none"
						class="focus"
						rx="0.2"
						ry="0.2"
					/>
				{/if}
			</svg>
			<figcaption class="plot-caption">Real/Imaginary</figcaption>
		</figure>
	{/if}
{/snippet}

<style>
	* {
		font-family: monospace;
	}

	svg {
		display: block;
		width: 100%;
		border: none;
	}

	rect {
		stroke-width: 1px;
		stroke-linejoin: bevel;
		stroke-linecap: butt;
		vector-effect: non-scaling-stroke;
	}

	.focus {
		stroke-width: 2px;
		vector-effect: non-scaling-stroke;
	}

	.number-slider {
		display: grid;
		grid-template-columns: 1fr auto;
		gap: 1ex 0.5ex;
		padding: 1ex 0;
		align-items: baseline;
	}

	.number-slider input {
		grid-column: 1 / -1;
	}

	.phase-slider {
		accent-color: hsl(calc(180 + 57 * var(--current-value, 0)), 100%, 50%);
	}
	.real-slider {
		accent-color: hsl(0, 0%, calc(50% + 50% * var(--current-value, 0)));
	}
	.imag-slider {
		accent-color: hsl(0, 0%, calc(50% + 50% * var(--current-value, 0)));
	}

	button {
		border: none;
		background: #111;
		color: #fff;
		cursor: pointer;
		display: inline-block;
		padding: 0.5ex 1ex;
	}

	button:hover {
		background: #252525;
	}

	button:active {
		background: #000;
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
		gap: 1em 0.5em;
	}

	.domain-split {
		display: flex;
		gap: 2px;
	}

	.options {
		width: max-content;
		padding: 1em 1em;
		margin: auto;
		min-width: 70em;
		box-sizing: border-box;
	}

	.domain-container {
		display: grid;
		grid-template-columns: 1fr 1fr;
		justify-content: center;
		justify-items: stretch;
		max-width: 70em;
		margin: auto;
		box-sizing: border-box;
		margin-top: 1em;
	}

	fieldset {
		align-self: stretch;
		border: 3px solid #ccc;
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
		padding: 4px 7px;
		display: flex;
		align-items: center;
		gap: 2ex;
	}

	input[type="range"] {
		width: 100%;
	}

	.button-row {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5ex;
	}

	.plot {
		margin: 0;
		padding: 0;
		display: grid;
		grid-template-rows: 1fr auto;
		gap: 0;
		background: #000;
		width: 100%;
	}

	.plot-graphic {
		margin: 0;
	}

	.plot-caption {
		background: #000;
		color: #fff;
		text-align: center;
		padding: 0.5ex 0 1ex;
	}

	.domain-name {
		font-size: 2em;
		margin: 0;
	}

	.radio-list {
		display: flex;
		align-items: baseline;
	}

	.radio-list-head {
		display: block;
		align-self: center;
		padding: 1ex;
	}

	.radio-list-head::after {
		content: ": ";
	}

	.radio-field {
		padding: 1ex;
		display: flex;
		align-items: center;
		gap: 1ex;
	}

	.radio-control {
		padding: 0;
		margin: 0;
	}

	.radio-label {
	}

	.checkbox-list {
		display: flex;
		align-items: baseline;
	}

	.checkbox-field {
		padding: 1ex;
		display: flex;
		align-items: center;
		gap: 1ex;
	}

	.checkbox-control {
		padding: 0;
		margin: 0;
	}

	.checkbox-label {
	}
	.polar-control {
		width: 100%;
	}
	.polar-control-axis {
		vector-effect: non-scaling-stroke;
	}
	.polar-control-axis-label {
		font-size: 0.5em;
	}
	.polar-control-head {
		vector-effect: non-scaling-stroke;
		fill: hsl(calc(180 + 57 * var(--value-phase, 0)), 100%, 50%);
		cursor: move;
		stroke: white;
		stroke-width: 2;
	}

	.polar-control-head-mirror {
		vector-effect: non-scaling-stroke;
		stroke: hsl(calc(180 - 57 * var(--value-phase, 0)), 100%, 50%);
		cursor: pointer;
		fill: hsl(calc(180 - 57 * var(--value-phase, 0)), 100%, 80%);
		stroke-width: 3;
		stroke-dasharray: 3 4;
	}

	.polar-control-unit {
		fill: none;
		vector-effect: non-scaling-stroke;
	}

	.polar-control-marker-magnitude,
	.polar-control-marker-real,
	.polar-control-marker-imag {
		vector-effect: non-scaling-stroke;
		stroke-dasharray: 2 3;
	}

	.polar-control-axis-value.hidden {
		display: none;
	}

	.polar-control-axis-value {
		font-size: 0.5em;
	}
</style>
