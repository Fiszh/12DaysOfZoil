<script lang="ts">
	let { children } = $props();
</script>

<svelte:head>
	<title>12 Days Of Zoil Tracker</title>
	<link rel="icon" href="https://cdn.unii.dev/zoil.png" type="image/png" />

	<meta name="darkreader-lock" />

	<style>
		@import url("https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap");
	</style>
</svelte:head>

{@render children()}
