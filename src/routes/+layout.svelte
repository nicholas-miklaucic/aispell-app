<script lang='ts'>
	// The ordering of these imports is critical to your app working properly
	import '../theme.postcss';
	// If you have source.organizeImports set to true in VSCode, then it will auto change this ordering
	import '@skeletonlabs/skeleton/styles/all.css';
	// Most of your app wide CSS should be put in this file
	import '../app.postcss';
	import { AppShell, AppBar, LightSwitch } from '@skeletonlabs/skeleton';

	function onClick() {
		if (document.body.classList.contains('theme-dark')) {
			document.body.classList.remove('theme-dark');			
		} else {			
			document.body.classList.add('theme-dark');
		}
	};
</script>

<!-- App Shell -->
<AppShell>
	<svelte:fragment slot="header">
		<!-- App Bar -->
		<AppBar>
			<svelte:fragment slot="lead">
				<!-- <strong class="text-xl">AISpell</strong> -->
				<span></span>
			</svelte:fragment>
			<svelte:fragment slot="trail">
				<LightSwitch on:click={onClick}/>
			</svelte:fragment>
		</AppBar>
	</svelte:fragment>
	<!-- Page Route Content -->
	<slot />
</AppShell>
