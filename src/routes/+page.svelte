<script>
	import { onDestroy } from 'svelte';

	const MODEL_URL = '/tm-my-image-model/';

	let model;
	let webcam;
	let tmImage;
	let maxPredictions = 0;
	let predictions = [];
	let webcamContainer;
	let isStarting = false;
	let isRunning = false;
	let animationFrameId;

	async function ensureLibrariesLoaded() {
		if (tmImage) return;
		await import('@tensorflow/tfjs');
		const tmImageModule = await import('@teachablemachine/image');
		tmImage = tmImageModule.default ?? tmImageModule;
	}

	async function init() {
		if (isStarting || isRunning) return;

		isStarting = true;
		try {
			await ensureLibrariesLoaded();
			const modelURL = `${MODEL_URL}model.json`;
			const metadataURL = `${MODEL_URL}metadata.json`;

			model = await tmImage.load(modelURL, metadataURL);
			maxPredictions = model.getTotalClasses();
			predictions = Array.from({ length: maxPredictions }, () => ({ className: '', probability: 0 }));

			const flip = true;
			webcam = new tmImage.Webcam(200, 200, flip);
			await webcam.setup();
			await webcam.play();

			if (webcamContainer) {
				webcamContainer.innerHTML = '';
				webcamContainer.appendChild(webcam.canvas);
			}

			isRunning = true;
			loop();
		} catch (error) {
			console.error(error);
		} finally {
			isStarting = false;
		}
	}

	async function predict() {
		if (!model || !webcam) return;

		const result = await model.predict(webcam.canvas);
		predictions = result.map((item) => ({
			className: item.className,
			probability: item.probability
		}));
	}

	async function loop() {
		if (!isRunning || !webcam) return;
		webcam.update();
		await predict();
		animationFrameId = window.requestAnimationFrame(loop);
	}

	function stop() {
		isRunning = false;
		if (animationFrameId) {
			window.cancelAnimationFrame(animationFrameId);
			animationFrameId = undefined;
		}
		if (webcam) {
			webcam.stop();
		}
	}

	onDestroy(() => {
		stop();
	});
</script>

<div>Teachable Machine Image Model</div>
<button type="button" onclick={init} disabled={isStarting || isRunning}>
	{#if isStarting}
		Starting...
	{:else if isRunning}
		Running
	{:else}
		Start
	{/if}
</button>

<div bind:this={webcamContainer}></div>

<div>
	{#each predictions as prediction}
		<div>{prediction.className}: {prediction.probability.toFixed(2)}</div>
	{/each}
</div>
