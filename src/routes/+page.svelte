<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import Icon, { loadIcons } from '@iconify/svelte';

	type tanpuraSequenceI = { node: number; delayToNext: number }[];

	const TANPURA_AUDIO_URL = '/assets/tanpura-d.wav';
	const nodes = ['A', 'A#', 'B', 'C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#'] as const;
	const nodeOffset = 5;

	let nodeIdx = 4,
		isMute = false,
		volumeLevel = 100,
		fineTuneLevel = 0;

	let isMounted = false;
	let isReady = false;
	let isPlaying = false;

	let audioContext: AudioContext | undefined;
	let tanpuraAudioBuffer: AudioBuffer | undefined;
	let tanpuraPlayingNodes: AudioBufferSourceNode[] = [];
	let tanpuraNextPos: NodeJS.Timer | undefined;

	let playPauseButton: HTMLButtonElement | undefined;
	let toneUpButton: HTMLButtonElement | undefined;
	let toneDownButton: HTMLButtonElement | undefined;
	let loadTanpuraButton: HTMLButtonElement | undefined;

	onMount(() => {
		document.addEventListener('keypress', handleKeyboardShortcut);
		loadIcons(['material-symbols:pause-outline', 'ic:round-volume-off', 'ic:round-volume-up']);
		if (localStorage) {
			let availableConfigJSON = localStorage.getItem('config-default');
			if (availableConfigJSON) {
				const availableConfig = JSON.parse(availableConfigJSON);
				nodeIdx = availableConfig.nodeIdx ?? 4;
				isMute = availableConfig.isMute ?? false;
				volumeLevel = availableConfig.volumeLevel ?? 100;
				fineTuneLevel = availableConfig.fineTuneLevel ?? 0;
			}
		}
		isMounted = true;
	});
	onDestroy(() => {
		document.removeEventListener('keypress', handleKeyboardShortcut);
		clearTimeout(tanpuraNextPos);
	});

	$: nodeIdx, isMute, volumeLevel, fineTuneLevel, setConfig();
	function setConfig() {
		if (!isMounted) {
			return;
		}
		localStorage.setItem(
			'config-default',
			JSON.stringify({ nodeIdx, isMute, volumeLevel, fineTuneLevel })
		);
	}

	const keyboardShortcuts = [
		{ keyArray: ['Space'], description: 'Play / Pause' },
		{ keyArray: ['Shift', 'j'], description: 'Tone down' },
		{ keyArray: ['Shift', 'k'], description: 'Tone up' },
		{ keyArray: [], description: 'Fine tuning' },
		{ keyArray: ['Shift', ';'], description: 'Fine tone reset' },
		{ keyArray: ['Shift', 'h'], description: 'Fine tone down' },
		{ keyArray: ['Shift', 'l'], description: 'Fine tone up' },
		{ keyArray: [], description: 'Volume' },
		{ keyArray: ['Shift', ','], description: 'Volume down' },
		{ keyArray: ['Shift', '.'], description: 'Volume up' },
		{ keyArray: ['Shift', '/'], description: 'Volume mute' }
	];
	function handleKeyboardShortcut(event: KeyboardEvent) {
		if (event.key == ' ' && document.activeElement?.tagName == 'BUTTON') {
			(document.activeElement as HTMLButtonElement).blur();
		}
		if (isReady && event.key == ' ') {
			event.preventDefault();
			playPauseButton?.click();
		} else {
			if (event.key == 'J') {
				toneDownButton?.click();
			} else if (event.key == 'K') {
				toneUpButton?.click();
			} else if (event.key == ':') {
				resetFineTune();
			} else if (event.key == 'H') {
				fineTuneLevel = Math.max(-100, fineTuneLevel - 1);
			} else if (event.key == 'L') {
				fineTuneLevel = Math.min(100, fineTuneLevel + 1);
			} else if (event.key == '<') {
				volumeLevel = Math.max(0, volumeLevel - 5);
			} else if (event.key == '>') {
				volumeLevel = Math.min(100, volumeLevel + 5);
			} else if (event.key == '?') {
				toggleMute();
			} else if (event.key == ' ') {
				event.preventDefault();
				loadTanpuraButton?.click();
			}
		}
	}

	function toneSet() {
		isReady = false;
		tanpuraPlayingNodes.forEach((node) => node.stop());
		if (audioContext === undefined) {
			audioContext = new AudioContext();
		}
		if (!tanpuraAudioBuffer) {
			fetch(TANPURA_AUDIO_URL)
				.then((tanpuraNode) => tanpuraNode.arrayBuffer())
				.then((tanpuraArrayBuffer) => new AudioContext().decodeAudioData(tanpuraArrayBuffer))
				.then((tanpuraAudio) => {
					tanpuraAudioBuffer = tanpuraAudio;
					isReady = true;
				});
		} else {
			isReady = true;
		}
	}
	function playDetuneNode(detune: number) {
		if (!audioContext || audioContext.state === 'closed' || !tanpuraAudioBuffer) {
			return;
		}
		const node = audioContext.createBufferSource();
		node.buffer = tanpuraAudioBuffer;
		const detuneCents = 100 * detune + fineTuneLevel;
		node.playbackRate.value = Math.pow(2, detuneCents / 1200);
		const nodeGain = audioContext.createGain();
		nodeGain.gain.value = isMute ? 0 : volumeLevel / 100;
		node.connect(nodeGain);
		nodeGain.connect(audioContext.destination);
		node.start(0);
		node.addEventListener('ended', function () {
			if (tanpuraPlayingNodes.length > 0) tanpuraPlayingNodes.shift();
		});
		return node;
	}
	function playTanpuraFromArray(tanpuraSeq: tanpuraSequenceI) {
		let currNode = playDetuneNode(tanpuraSeq[0].node + nodeIdx - nodeOffset);
		if (currNode) {
			tanpuraPlayingNodes.push(currNode);
		}
		tanpuraNextPos = setTimeout(() => {
			const latestNode = tanpuraSeq.shift();
			if (latestNode) {
				tanpuraSeq.push(latestNode);
			}
			playTanpuraFromArray(tanpuraSeq);
		}, tanpuraSeq[0].delayToNext);
	}
	function playTanpura() {
		playTanpuraFromArray([
			{ node: -5, delayToNext: 1000 },
			{ node: 0, delayToNext: 1000 },
			{ node: 0, delayToNext: 1000 },
			{ node: -12, delayToNext: 1500 }
		]);
	}
	async function playPause() {
		isPlaying = !isPlaying;
		if (isPlaying) {
			await audioContext?.resume();
			playTanpura();
		} else {
			audioContext?.suspend();
			tanpuraPlayingNodes.forEach((node) => node.stop());
			clearTimeout(tanpuraNextPos);
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
	function resetFineTune() {
		fineTuneLevel = 0;
	}
	function toggleMute() {
		isMute = !isMute;
	}
</script>

<svelte:head>
	<title>Browser tanpura</title>
	<meta
		name="description"
		content="A browser (javascript) based open source Tanpura application."
	/>
</svelte:head>
<div class="container">
	{#if audioContext === undefined}
		<div class="loading-layer">
			<button
				aria-label="Load tanpura"
				on:click={toneSet}
				disabled={!isMounted}
				bind:this={loadTanpuraButton}
			>
				Load Tanpura
			</button>
		</div>
	{/if}
	<div class="fine-tune-control">
		<button aria-label="Reset fine tuning" on:click={resetFineTune}>
			<Icon icon="material-symbols:discover-tune" />
		</button>
		<input type="range" min={-100} max={100} bind:value={fineTuneLevel} aria-label="fine-tune" />
		<span>
			{#if fineTuneLevel >= 0}+{:else}-{/if}{(
				100 *
				(Math.pow(2, Math.abs(fineTuneLevel) / 100) - 1)
			).toFixed(1)}%
		</span>
	</div>
	<div class="content-box">
		<div class="heading">Tanpura</div>
		<div class="scale">
			<button aria-label="Tone down by one node" on:click={toneDown} bind:this={toneDownButton}>
				<Icon icon="material-symbols:arrow-drop-down" />
			</button>
			<span>{nodes[nodeIdx]}</span>
			<button aria-label="Tone up by one node" on:click={toneUp} bind:this={toneUpButton}>
				<Icon icon="material-symbols:arrow-drop-up" />
			</button>
		</div>
		<button
			class="play-pause"
			aria-label={isPlaying ? 'Pause' : 'Play'}
			on:click={playPause}
			disabled={!isReady}
			bind:this={playPauseButton}
		>
			{#if isPlaying}
				<Icon icon="material-symbols:pause-outline" />
			{:else}
				<Icon icon="material-symbols:play-arrow" />
			{/if}
		</button>
	</div>
	<div class="volume-control">
		<button aria-label={isMute ? 'Unmute' : 'Mute'} on:click={toggleMute}>
			{#if isMute}
				<Icon icon="ic:round-volume-off" />
			{:else}
				<Icon icon="ic:round-volume-up" />
			{/if}
		</button>
		<input type="range" min={0} max={100} bind:value={volumeLevel} aria-label="volume" />
		<span>{volumeLevel}%</span>
	</div>
</div>
<br />
<h2>Keyboard shortcuts:</h2>
<table class="keyboard-shortcuts">
	<thead>
		<tr> <th>Key shortcut</th> <th>Action</th> </tr>
	</thead>
	<tbody>
		{#each keyboardShortcuts as shortcutDetails}
			<tr>
				{#if shortcutDetails.keyArray.length == 0}
					<td colspan="2"> <b> {shortcutDetails.description} </b> </td>
				{:else}
					<td>
						{#each shortcutDetails.keyArray as key}
							<div class="key">
								{#if key == 'Shift'}
									<Icon icon="material-symbols:shift-lock" />
								{:else if key == 'Space'}
									<Icon icon="material-symbols:space-bar" />
								{/if}
								{key}
							</div>
						{/each}
					</td>
					<td>{shortcutDetails.description}</td>
				{/if}
			</tr>
		{/each}
	</tbody>
</table>

<style lang="scss">
	.container {
		display: flex;
		position: relative;
		max-width: 24em;
		min-height: 20rem;
		margin: auto;
		.loading-layer {
			position: absolute;
			display: grid;
			place-items: center;
			width: 100%;
			height: 100%;
			background-color: rgba(0, 0, 0, 0.5);
			button {
				max-width: 12em;
				padding: 1em;
				font-size: 0.8em;
				&:disabled {
					background-color: rgba(200, 200, 200, 1);
				}
			}
		}
		.content-box {
			flex: 1;
			display: flex;
			justify-content: space-between;
			flex-direction: column;
			gap: 1em;
			padding: 1rem;
			border-bottom: 0.1rem solid;
			.heading {
				padding: 1em;
				margin: -1rem -1rem 0rem -1rem;
				border-width: 0.1rem 0;
				border-style: solid;
				text-align: center;
				font-weight: bold;
			}
			.scale {
				display: flex;
				align-items: center;
				gap: 0.4em;
				font-size: 2em;
				span {
					font-size: 1.4em;
					flex: 1;
					text-align: center;
				}
			}
			.play-pause {
				font-size: 1.6em;
			}
		}
		.volume-control,
		.fine-tune-control {
			display: flex;
			justify-content: center;
			align-items: center;
			gap: 1em;
			width: clamp(1.6em, 16%, 3em);
			padding: 0.6em 0;
			border: 0.1rem solid;
			writing-mode: vertical-lr;
			input {
				width: 100%;
				appearance: auto;
				-webkit-appearance: slider-vertical;
			}
			span {
				font-size: 0.8em;
				writing-mode: horizontal-tb;
			}
		}
		button {
			display: flex;
			justify-content: center;
			padding: 0.2rem;
			border-radius: 0.2em;
			background-color: white;
			font-size: inherit;
		}
	}
	.keyboard-shortcuts {
		td {
			padding: 0.2em;
		}
		.key {
			display: inline-block;
			padding: 0.2em;
			border: 0.1rem solid;
			border-radius: 0.2em;
			cursor: pointer;
			&:not(:last-child) {
				margin-right: 0.2em;
			}
		}
	}
	@media (max-width: 500px) {
		.container {
			font-size: 1.2em;
		}
	}
	@media (min-width: 501px) {
		.container {
			font-size: 1.6em;
		}
	}
</style>
