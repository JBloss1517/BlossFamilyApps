<script>
	export const prerender = true;
	import { SvelteUIProvider } from '@svelteuidev/core';

	import Header from './Header.svelte';
	import './styles.css';
</script>

<div class="app">
	<Header />

	<SvelteUIProvider themeObserver="light">
		<slot />
	</SvelteUIProvider>
</div>

<style>
	.app {
		display: flex;
		flex-direction: column;
		min-height: 100vh;
	}
</style>
