<script lang="ts">
	import type { PageData } from './$types';
	import { VisXYContainer, VisStackedBar, VisAxis } from '@unovis/svelte';
	import { Scale } from '@unovis/ts';
	import { onMount } from 'svelte';
	import { writable, derived } from 'svelte/store';	

	export let data: PageData;

	let loaded = false;
	onMount(async () => {
		loaded = true;
	});

	let loading = false;

	let text = 'Truly intelligent spell chuck';
	$: words = get_words(text);
	let word_index = -1;
	let ed_2 = true;
	let corrs: Correction[] = data.test_corrs.slice(0, 15);
	$: curr_req_id = words.join(' ') + ed_2;

	$: get_corrections(words, word_index, ed_2, curr_req_id)
		.then(([req_id, c]) => {
			if (req_id == curr_req_id && c.length > 0) {
				corrs = c.slice(0, 15);
				loading = false;
			}
		})
		.catch((e) => console.log(e));

	type Correction = {
		term: string;
		prob: number;
		lm_prob: number;
		kbd_prob: number;
	};

	async function get_corrections(
		words: string[],
		word_index: number,
		ed_2: boolean,
		req_id: string
	): Promise<[string, Correction[]]> {		
		await new Promise(r => setTimeout(r, 500));
		if (req_id != curr_req_id) {
			return [req_id, []];
		}
		loading = true;
		const resp = await fetch('http://localhost:3030/corrections', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({
				text: words.slice(0, ((word_index + words.length) % words.length) + 1).join(' '),
				edit_dist: ed_2 ? 2 : 1
			})
		});

		if (resp.ok) {
			console.log(resp);
			let json = await resp.json();			
			return [req_id, json as Correction[]];
		} else {
			return [req_id, []];
		}
	}

	function logit(x: number): number {
		return Math.log(x / (1 - x));
	}

	function kbd_component(c: Correction): number {
		return (c.prob * Math.log(c.lm_prob)) / (Math.log(c.lm_prob) + Math.log(c.kbd_prob));
	}

	function lm_component(c: Correction): number {
		return (c.prob * Math.log(c.kbd_prob)) / (Math.log(c.lm_prob) + Math.log(c.kbd_prob));
	}

	/// Return a list of words in the string
	function get_words(s: string): string[] {
		return s.split(' ');
	}
</script>

<div class="container h-full w-full mx-auto justify-center items-center">
	<div class="grid grid-cols-1 grid-rows-6 h-full w-full">
		<div class="row-span-2 my-auto space-y-10 py-10">
			<div class="w-full mx-auto justify-center items-center">
				<h1 class="unstyled text-6xl font-bold text-center gradient-heading">AISpell</h1>				
				<h2 class="unstyled text-3xl font-italic text-center">Truly intelligent spell check</h2>
			</div>
			<div class="grid grid-cols-1 py-2 px-2 rounded-xl">
				<label class="label space-x-10 grid grid-cols-6">
					<span class="form-label">Consider two-typo completions</span>
					<div class="form-input">
						<input type="checkbox" bind:checked={ed_2} class="checkbox" />
					</div>
				</label>
				<label class="label space-x-10 grid grid-cols-6">
					<span class="form-label">Text</span>
					<div class="form-input">
						<input
							type="input"
							bind:value={text}
							class="input font-mono px-1 variant-form-material text-2xl bg-transparent"
						/>
					</div>
				</label>
			</div>
		</div>
		<div class="row-span-4 justify-center mb-10 font-mono">
			<div class="h-full">
			{#if loaded}				
					<div class="mb-1">
						{#each words as word, i (i)}
							{#if (word_index + words.length) % words.length == i}
								<button class="word-active word">{word}</button>
							{:else}
								<button class="word" on:click={(e) => (word_index = i + 1 != words.length ? i : -1)}
									>{word}</button
								>
							{/if}
						{/each}
					</div>
					<div class="h-full pb-3 {loading ? 'blur-md' : 'blur-none'}">
						<VisXYContainer data={corrs} height="100%" xDomain={[0, 1]}>
							<VisAxis
								type="y"
								data={corrs}
								gridLine={undefined}
								tickValues={corrs.map((_, i) => -i)}
								tickFormat={(i) => corrs[-i].term}
								label="Correction"
								labelMargin={20}
							/>
							<VisAxis
								type="x"
								gridLine={undefined}
								position="top"
								domainLine={true}
								label="Probability"
								tickValues={[0, 0.25, 0.5, 0.75, 1]}
							/>
							<VisStackedBar
								data={corrs}
								x={(corr, i) => -i}
								y={[kbd_component, lm_component]}
								orientation="horizontal"
								maxBarWidth={100}
							/>
						</VisXYContainer>
					</div>				
			{/if}
			</div>
			<div class="table-container font-mono py-10">
				<table class="table table-hover">
					<thead>
						<tr>
							<th>Term</th>
							<th>Prob</th>
							<th>LM Prob</th>
							<th>KBD Prob</th>
						</tr>
					</thead>
					<tbody>
						{#each corrs as corr (corr.term)}
							<tr>
								<td>{corr.term}</td>
								<td>{corr.prob}</td>
								<td>{corr.lm_prob}</td>
								<td>{corr.kbd_prob}</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		</div>
	</div>
</div>

<style lang="postcss">
	.form-input {
		@apply col-span-4 border-none py-1 px-0 bg-transparent align-middle;
	}

	.form-label {
		@apply text-right py-2 text-xl;
		@apply col-span-2;
	}

	:root {
		--vis-font-family: theme('fontFamily.mono');
		--vis-axis-tick-label-color: theme('colors.surface.600');
		--vis-dark-axis-tick-label-color: theme('colors.surface.300');
		--vis-axis-tick-label-font-size: theme('fontSize.lg');		

		--vis-color0: theme('colors.primary.600');
		--vis-color1: theme('colors.secondary.600');

		--vis-dark-color0: theme('colors.primary.400');
		--vis-dark-color1: theme('colors.secondary.400');

		/* --vis-color0: #1846b3;
		--vis-color1: #355e00;
		--vis-color2: #bb8600;
		--vis-color3: #dd2800;
		--vis-color4: #c510d2;
		--vis-color5: #9470fe;

		--vis-dark-color0: #00a0ec;
		--vis-dark-color1: #00bc70;
		--vis-dark-color2: #deca00;
		--vis-dark-color3: #ff7300;
		--vis-dark-color4: #d83390;
		--vis-dark-color5: #7555d3; */
	}

	.word {
		@apply text-surface-400-500-token p-1 mx-1 btn-xl;				
	}

	.word.word-active {
		@apply text-tertiary-400-500-token;
	}

	.gradient-heading {
		@apply bg-clip-text text-transparent box-decoration-clone;
		/* Direction */
		@apply bg-gradient-to-br;
		/* Color Stops */
		@apply from-primary-500 to-secondary-500;
	}
</style>
