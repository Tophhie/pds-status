<script lang="ts">
	import favicon from '$lib/assets/favicon.png';
	import '../app.css';
	let { children } = $props();
</script>

<svelte:head>
	<title>Tophhie Social Server Status</title>
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children()}
