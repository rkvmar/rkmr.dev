<script lang="ts">
	import ShellPrompt from '$lib/components/ShellPrompt.svelte';
	import { onMount } from 'svelte';
	let shellPrompt;

	async function runCommand(command: string) {
		await shellPrompt.typeAndExecuteCommand(command);
	}
	onMount(() => {
		runCommand('run');
	});
</script>

<div class="container">
	<ShellPrompt bind:this={shellPrompt} typingDelay={50} commandPause={100} />
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
