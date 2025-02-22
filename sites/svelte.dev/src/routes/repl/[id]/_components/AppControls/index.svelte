<script>
	import { createEventDispatcher, getContext } from 'svelte';
	import { session } from '$app/stores';
	import UserMenu from './UserMenu.svelte';
	import { Icon } from '@sveltejs/site-kit';
	import * as doNotZip from 'do-not-zip';
	import downloadBlob from '../../../_utils/downloadBlob.js';
	import { enter } from '../../../../../utils/events.js';
	import { isMac } from '../../../../../utils/compat.js';

	const dispatch = createEventDispatcher();
	const { login } = getContext('app');

	export let repl;
	export let gist;
	export let name;
	export let zen_mode;
	export let modified_count;

	let saving = false;
	let downloading = false;
	let justSaved = false;
	let justForked = false;

	function wait(ms) {
		return new Promise((f) => setTimeout(f, ms));
	}

	$: canSave = $session.user && gist && gist.owner === $session.user.id;

	function handleKeydown(event) {
		if (event.key === 's' && (isMac ? event.metaKey : event.ctrlKey)) {
			event.preventDefault();
			save();
		}
	}

	async function fork(intentWasSave) {
		saving = true;

		const { components } = repl.toJSON();

		try {
			const r = await fetch(`/repl/create.json`, {
				method: 'POST',
				credentials: 'include',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					name,
					files: components.map((component) => ({
						name: `${component.name}.${component.type}`,
						source: component.source
					}))
				})
			});

			if (r.status < 200 || r.status >= 300) {
				const { error } = await r.json();
				throw new Error(`Received an HTTP ${r.status} response: ${error}`);
			}

			const gist = await r.json();
			dispatch('forked', { gist });

			modified_count = 0;
			repl.markSaved();

			if (intentWasSave) {
				justSaved = true;
				await wait(600);
				justSaved = false;
			} else {
				justForked = true;
				await wait(600);
				justForked = false;
			}
		} catch (err) {
			if (navigator.onLine) {
				alert(err.message);
			} else {
				alert(`It looks like you're offline! Find the internet and try again`);
			}
		}

		saving = false;
	}

	async function save() {
		if (saving) return;

		if (!canSave) {
			fork(true);
			return;
		}

		saving = true;

		try {
			// Send all files back to API
			// ~> Any missing files are considered deleted!
			const { components } = repl.toJSON();

			const r = await fetch(`/repl/${gist.id}.json`, {
				method: 'PUT',
				credentials: 'include',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					name,
					files: components.map((component) => ({
						name: `${component.name}.${component.type}`,
						source: component.source
					}))
				})
			});

			if (r.status < 200 || r.status >= 300) {
				const { error } = await r.json();
				throw new Error(`Received an HTTP ${r.status} response: ${error}`);
			}

			modified_count = 0;
			repl.markSaved();
			justSaved = true;
			await wait(600);
			justSaved = false;
		} catch (err) {
			if (navigator.onLine) {
				alert(err.message);
			} else {
				alert(`It looks like you're offline! Find the internet and try again`);
			}
		}

		saving = false;
	}

	async function download() {
		downloading = true;

		const { components, imports } = repl.toJSON();

		const files = await (await fetch('/svelte-app.json')).json();

		if (imports.length > 0) {
			const idx = files.findIndex(({ path }) => path === 'package.json');
			const pkg = JSON.parse(files[idx].data);
			const { devDependencies } = pkg;
			imports.forEach((mod) => {
				const match = /^(@[^/]+\/)?[^@/]+/.exec(mod);
				devDependencies[match[0]] = 'latest';
			});
			pkg.devDependencies = devDependencies;
			files[idx].data = JSON.stringify(pkg, null, '  ');
		}

		files.push(
			...components.map((component) => ({
				path: `src/${component.name}.${component.type}`,
				data: component.source
			}))
		);
		files.push({
			path: `src/main.js`,
			data: `import App from './App.svelte';

var app = new App({
	target: document.body
});

export default app;`
		});

		downloadBlob(doNotZip.toBlob(files), 'svelte-app.zip');

		downloading = false;
	}
</script>

<svelte:window on:keydown={handleKeydown} />

<div class="app-controls">
	<input bind:value={name} on:focus={e => e.target.select()} use:enter={e => e.target.blur()} />

	<div style="text-align: right; margin-inline-end: .4rem">
		<button class="icon" on:click={() => (zen_mode = !zen_mode)} title="fullscreen editor">
			{#if zen_mode}
				<Icon name="close" />
			{:else}
				<Icon name="maximize" />
			{/if}
		</button>

		<button class="icon" disabled={downloading} on:click={download} title="download zip file">
			<Icon name="download" />
		</button>

		<button
			class="icon"
			disabled={saving || !$session.user}
			on:click={() => fork(false)}
			title="fork"
		>
			{#if justForked}
				<Icon name="check" />
			{:else}
				<Icon name="git-branch" />
			{/if}
		</button>

		<button class="icon" disabled={saving || !$session.user} on:click={save} title="save">
			{#if justSaved}
				<Icon name="check" />
			{:else}
				<Icon name="save" />
				{#if modified_count}
					<div class="badge">{modified_count}</div>
				{/if}
			{/if}
		</button>

		{#if $session.user}
			<UserMenu />
		{:else}
			<button class="icon" on:click|preventDefault={login}>
				<Icon name="log-in" />
				<span>&nbsp;Log in to save</span>
			</button>
		{/if}
	</div>
</div>

<style>
	.app-controls {
		position: absolute;
		inset-block-start: 0;
		inset-inline-start: 0;
		inline-size: 100%;
		block-size: var(--app-controls-h);
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding-inline: var(--side-nav);
		padding-block: 0.6rem;
		background-color: var(--second);
		color: white;
		white-space: nowrap;
		flex: 0;
	}

	.icon {
		position: relative;
		inset-block-start: -0.1rem;
		display: inline-block;
		padding: 0.2em;
		opacity: 0.7;
		transition: opacity 0.3s;
		font-family: var(--font);
		font-size: 1.6rem;
		color: white;
		/* width: 1.6em;
		block-size: 1.6em; */
		line-height: 1;
		margin-inline-start: 0.2em;
	}

	.icon:hover {
		opacity: 1;
	}
	.icon:disabled {
		opacity: 0.3;
	}

	.icon[title^='fullscreen'] {
		display: none;
	}

	input {
		background: transparent;
		border: none;
		color: currentColor;
		font-family: var(--font);
		font-size: 1.6rem;
		opacity: 0.7;
		outline: none;
		flex: 1;
		margin-inline: 0.4em 0.2em;
		padding-block-start: 0.2em;
		border-block-end: 1px solid transparent;
	}

	input:hover {
		border-block-end: 1px solid currentColor;
		opacity: 1;
	}
	input:focus {
		border-block-end: 1px solid currentColor;
		opacity: 1;
	}

	button span {
		display: none;
	}

	.badge {
		background: #ff3e00;
		border-radius: 100%;
		font-size: 10px;
		padding: 0;
		inline-size: 15px;
		block-size: 15px;
		line-height: 15px;
		position: absolute;
		inset-block-start: 10px;
		inset-inline-end: 0px;
	}

	@media (min-width: 600px) {
		.icon[title^='fullscreen'] {
			display: inline;
		}

		button span {
			display: inline-block;
		}
	}
</style>
