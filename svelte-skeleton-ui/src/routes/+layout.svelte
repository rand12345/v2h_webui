<script lang="ts">
	import '../app.postcss';
	import { AppShell, AppBar, LightSwitch } from '@skeletonlabs/skeleton';
	import { onMount, onDestroy } from 'svelte';
	import { page } from '$app/stores';
	// Highlight JS
	import hljs from 'highlight.js';
	import 'highlight.js/styles/github-dark.css';
	import { storeHighlightJs } from '@skeletonlabs/skeleton';
	storeHighlightJs.set(hljs);

	// Floating UI for Popups
	import { computePosition, autoUpdate, flip, shift, offset, arrow } from '@floating-ui/dom';
	import { storePopup } from '@skeletonlabs/skeleton';
	storePopup.set({ computePosition, autoUpdate, flip, shift, offset, arrow });
</script>

<!-- App Shell -->

<AppShell>
	<svelte:fragment slot="header">
		<!-- App Bar -->
		<AppBar>
			<svelte:fragment slot="lead">
				<strong class="text-xl uppercase">CHAdeMO V2h</strong>
			</svelte:fragment>
			<svelte:fragment slot="trail">
				{#if $page.url.pathname === '/'}
					<a class="btn btn-sm variant-ghost-surface" href="./debug" rel="noreferrer"> Debug </a>
				{/if}
				{#if $page.url.pathname === '/debug'}
					<a class="btn btn-sm variant-ghost-surface" href="./" rel="noreferrer"> Home </a>
				{/if}
				<LightSwitch />
			</svelte:fragment>
		</AppBar>
	</svelte:fragment>
	<!-- Page Route Content -->
	<slot />
</AppShell>
