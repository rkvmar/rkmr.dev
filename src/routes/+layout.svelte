<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>
{@render children?.()}

<style>
    :root {
        --text: #0f0321;
        --subtext: #0f0321;
        --background: #f8f3fe;
        --primary: #a675ef;
        --secondary: #cdabf6;
        --accent: #edcb57;
        --linearPrimarySecondary: linear-gradient(#a675ef, #cdabf6);
        --linearPrimaryAccent: linear-gradient(#a675ef, #edcb57);
        --linearSecondaryAccent: linear-gradient(#cdabf6, #edcb57);
        --radialPrimarySecondary: radial-gradient(#a675ef, #cdabf6);
        --radialPrimaryAccent: radial-gradient(#a675ef, #edcb57);
        --radialSecondaryAccent: radial-gradient(#cdabf6, #edcb57);

    }
	:global(body) {
		width: 100vw;
		height: 100vh;
		margin: 0;
		padding: 0;
	}
</style>
