<script lang="ts">
	import { onMount } from 'svelte';

	type tanpuraSequenceI = { node: number; delayToNext: number }[];

	const nodes = ['A', 'A#', 'B', 'C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#'] as const;

	let nodeIdx = 4;
	let nodeOffset = 5;
	let volumeLevel = 10;

	let isMounted = false;
	let isReady = false;
	let isPlaying = false;

	const TANPURA_AUDIO_URL = '/assets/tanpura-d.wav';

	let audioContext: AudioContext | undefined;
	let tanpuraAudioBuffer: AudioBuffer | undefined;
	let tanpuraLoop: NodeJS.Timer | undefined;
	let tanpuraCurrPos: NodeJS.Timer | undefined;

	onMount(() => {
		isMounted = true;
	});

	async function toneSet() {
		isReady = false;
		if (audioContext?.state !== 'closed') {
			audioContext?.close();
		}
		const ctx = new AudioContext();
		audioContext = ctx;
		tanpuraAudioBuffer = await fetch(TANPURA_AUDIO_URL)
			.then((tanpuraSound) => tanpuraSound.arrayBuffer())
			.then((tanpuraArrayBuffer) => ctx.decodeAudioData(tanpuraArrayBuffer));
		isReady = true;
	}

	$: volumeLevel;
	function playDetuneNode(detune: number) {
		if (!audioContext || audioContext.state==="closed" || !tanpuraAudioBuffer) {
			return;
		}
		const node = audioContext.createBufferSource();
		node.buffer = tanpuraAudioBuffer;
		node.detune.value = 100 * detune;
		const nodeGain = audioContext.createGain();
		nodeGain.gain.value = volumeLevel / 10;
		node.connect(nodeGain);
		nodeGain.connect(audioContext.destination);
		node.start(0);
		return;
	}
	function playTanpuraFromArray(tanpuraSeq: tanpuraSequenceI) {
		playDetuneNode(tanpuraSeq[0].node + nodeIdx - nodeOffset);
		tanpuraCurrPos = setTimeout(() => {
			const latestNode=tanpuraSeq.shift();
			if(latestNode) {
				tanpuraSeq.push(latestNode)
			}
			playTanpuraFromArray(tanpuraSeq);
		}, tanpuraSeq[0].delayToNext);
	}
	function playTanpuraCycle() {
		playTanpuraFromArray([
			{ node: -5, delayToNext: 1000 },
			{ node: 0, delayToNext: 1000 },
			{ node: 0, delayToNext: 1000 },
			{ node: -12, delayToNext: 1500 }
		]);
	}
	function playTanpura() {
		playTanpuraCycle();
	}

	function playPause() {
		isPlaying = !isPlaying;
		if (isPlaying) {
			playTanpura();
		} else {
			clearInterval(tanpuraLoop);
			clearTimeout(tanpuraCurrPos);
		}
	}
	function toneUp() {
		nodeIdx = (nodeIdx + 1) % nodes.length;
		toneSet();
	}
	function toneDown() {
		nodeIdx = (nodes.length + nodeIdx - 1) % nodes.length;
		toneSet();
	}
</script>

<svelte:head>
	<title>Browser tanpura</title>
</svelte:head>
<div class="container">
	{#if audioContext === undefined}
		<div class="loading-layer">
			<button on:click={toneSet} disabled={!isMounted}>Load Tanpura</button>
		</div>
	{/if}
	<div class="content-box">
		<div class="heading">Tanpura</div>
		<div class="scale">
			<button on:click={toneDown}>▼</button>
			<span>{nodes[nodeIdx]}</span>
			<button on:click={toneUp}>▲</button>
		</div>
		<button class="play-pause" on:click={playPause} disabled={!isReady}>
			{#if isPlaying}
				<span class="pause-symbol">| |</span>
			{:else}
				&#9658;
			{/if}
		</button>
	</div>
	<div class="volume-control">
		<div>&#128266;</div>
		<input type="range" min={0} max={10} bind:value={volumeLevel} />
	</div>
</div>

<style lang="scss">
	.container {
		display: flex;
		position: relative;
		max-width: 32em;
		border: 0.1rem solid;
		margin: auto;
		.loading-layer {
			position: absolute;
			display: grid;
			place-items: center;
			width: 100%;
			height: 100%;
			background-color: rgba(0, 0, 0, 0.5);
			button {
				max-width: 12rem;
				padding: 1rem;
				font-size: 1.2rem;
				&:disabled {
					background-color: rgba(200, 200, 200, 1);
				}
			}
		}
		.content-box {
			flex: 1;
			display: flex;
			flex-direction: column;
			gap: 1em;
			padding: 1rem;
			border: 0.1rem solid;
			.heading {
				padding: 1rem;
				margin: -1.1rem;
				border: 0.1rem solid;
				text-align: center;
				font-weight: bold;
				font-size: 1.6rem;
			}
			.scale {
				display: flex;
				align-items: center;
				gap: 0.8rem;
				font-size: 4.8rem;
				button {
					width: 4rem;
					height: 4rem;
				}
				span {
					flex: 1;
					text-align: center;
				}
			}
			.play-pause {
				align-self: center;
				width: 12rem;
				height: 3rem;
				.pause-symbol {
					font-weight: 600;
				}
			}

			button {
				padding: 0;
				font-size: 1.6rem;
				background-color: white;
				border-radius: 0.2rem;
			}
		}
		.volume-control {
			display: flex;
			justify-content: center;
			align-items: center;
			gap: 1rem;
			width: 3rem;
			border: 0.1rem solid;
			font-size: 2rem;
			writing-mode: vertical-lr;
			input {
				width: 100%;
				appearance: auto;
				-webkit-appearance: slider-vertical;
			}
		}
	}
</style>
