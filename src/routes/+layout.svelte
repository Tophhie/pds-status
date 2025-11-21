<script lang="ts">
	import favicon from '$lib/assets/favicon.png';
	import '../app.css';
	let { children } = $props();
</script>

<svelte:head>
	<title>Tophhie Social Server Status</title>
	<link rel="icon" href={favicon} />
	<meta name="theme-color" content="#100235" >
	<meta name="apple-mobile-web-app-capable" content="yes">
	<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
</svelte:head>

{@render children()}
