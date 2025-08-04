<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>
{@render children?.()}

<style>
	:global(body) {
		width: 100vh;
		height: 100vw;
		margin: 0;
	}
</style>
