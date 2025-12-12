<script lang="ts">
	import ShellPrompt from '$lib/components/ShellPrompt.svelte';
	import { onMount } from 'svelte';
	import type { PageData } from './$types';

	export let data: PageData;
	let shellPrompt;
	let mounted = false;

	async function runCommand(command: string) {
		await shellPrompt.typeAndExecuteCommand(command);
	}

	// Run the command after both component is mounted and shellPrompt is ready
	$: if (mounted && shellPrompt) {
		runCommand('run');
	}

	onMount(() => {
		mounted = true;
	});
</script>

<div class="container">
	<ShellPrompt bind:this={shellPrompt} typingDelay={50} commandPause={100} initialPath={data.slug ? `~/${data.slug}` : '~/'} />
</div>

<style>
	@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:ital,wght@0,100..800;1,100..800&display=swap');
	.container {
		background-color: #181825;
		width: 100vw;
		height: 100vh;
		display: flex;
		justify-content: left;
	}
</style>
