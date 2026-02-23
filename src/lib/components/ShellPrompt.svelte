<script lang="ts">
	import { onMount } from 'svelte';

	export let user = 'rkmr';
	export let host = 'rkmr.dev';
	export let path = '~/';
	export let initialPath = '';
	let prompt = '';
	let output: Array<{
		type: 'command' | 'result';
		content: string;
		user?: string;
		host?: string;
		path?: string;
	}> = [];
	let currentPath = '~/';

	export let typingDelay = 50;
	export let commandPause = 100;

	let terminalElement: HTMLElement;

	const directories = {
		'~/': ['about', 'projects'],
		'~/about': [],
		'~/projects': []
	};

	const commands = {
		help: 'Available commands: help, clear, cd, ls, run, open',
		clear: '',
		ls: '',
		cd: '',
		run: '',
		open: '',
		fimsh: 'he? he romtater??'
	};

	const runOutputs = {
		'~/': '         ░██                                           ░██                       \n         ░██                                           ░██                       \n░██░████ ░██    ░██░█████████████  ░██░████      ░████████  ░███████  ░██    ░██ \n░███     ░██   ░██ ░██   ░██   ░██ ░███         ░██    ░██ ░██    ░██ ░██    ░██ \n░██      ░███████  ░██   ░██   ░██ ░██          ░██    ░██ ░█████████  ░██  ░██  \n░██      ░██   ░██ ░██   ░██   ░██ ░██          ░██   ░███ ░██          ░██░██   \n░██      ░██    ░██░██   ░██   ░██ ░██      ░██  ░█████░██  ░███████     ░███    \n\n I make stupid stuff on the interwebs\n[link:cmd:cd ~/about && clear && run]About[/link] | [link:cmd:cd ~/projects && clear && run]Projects[/link] | [link:cmd:open https://github.com/rkvmar]Github[/link]',
		'~/about':
			"           ░██                                 ░██    \n           ░██                                 ░██    \n ░██████   ░████████   ░███████  ░██    ░██ ░████████ \n      ░██  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n ░███████  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n░██   ░██  ░███   ░██ ░██    ░██ ░██   ░███    ░██    \n ░█████░██ ░██░█████   ░███████   ░█████░██     ░████\n\nHai!\nI make things on the interwebs.\nyou can find some of my projects in [link:cmd:cd ./projects && clear && run]Projects[/link]\n[link:cmd:cd && clear && run]Home[/link]",
		'~/projects':
			'                                 ░██                          ░██               \n                                                              ░██               \n░████████  ░██░████  ░███████    ░██  ░███████   ░███████  ░████████  ░███████  \n░██    ░██ ░███     ░██    ░██   ░██ ░██    ░██ ░██    ░██    ░██    ░██        \n░██    ░██ ░██      ░██    ░██   ░██ ░█████████ ░██           ░██     ░███████  \n░███   ░██ ░██      ░██    ░██   ░██ ░██        ░██    ░██    ░██           ░██ \n░██░█████  ░██       ░███████    ░██  ░███████   ░███████      ░████  ░███████  \n░██                              ░██                                            \n░██                            ░███\n\n[link:cmd:open https://headways.rkmr.dev]Headways[/link] - filterable UI for [link:cmd:open https://pantographapp.com]Pantograph[/link]\n[link:cmd:open kanji.rkmr.dev]Kanji[/link] - Kanji training website based on "Kanji Look + Learn" workbook\n[link:cmd:open vehicles.rkmr.dev]Vehicles[/link] - Historical public transit tracker\n[link:cmd:open https://github.com/rkvmar/CelesteSplitsGenerator]CelesteSplitsGenerator[/link] - Speedrun splits generator for Celeste\nBayAreaGame - Unofficial App for Jet Lag: The Game (Link coming soon)\nCalico - lightweight tui calendar app (WIP)\n\n[link:cmd:cd && clear && run]Home[/link]'
	};

	function parseLinks(text: string) {
		const linkRegex = /\[link:(url|cmd):([^\]]+)\]([^\[]+)\[\/link\]/g;
		const parts = [];
		let lastIndex = 0;
		let match;

		while ((match = linkRegex.exec(text)) !== null) {
			if (match.index > lastIndex) {
				parts.push({
					type: 'text',
					content: text.slice(lastIndex, match.index)
				});
			}

			parts.push({
				type: 'link',
				linkType: match[1], // 'url' or 'cmd'
				target: match[2],
				content: match[3]
			});

			lastIndex = match.index + match[0].length;
		}

		if (lastIndex < text.length) {
			parts.push({
				type: 'text',
				content: text.slice(lastIndex)
			});
		}

		return parts.length > 0 ? parts : [{ type: 'text', content: text }];
	}

	async function handleLinkClick(linkType: string, target: string) {
		if (linkType === 'url') {
			window.open(target, '_blank');
		} else if (linkType === 'cmd') {
			if (target.includes('&&')) {
				const commands = target.split('&&').map((cmd) => cmd.trim());
				for (const cmd of commands) {
					await typeAndExecuteCommand(cmd);
					await new Promise((resolve) => setTimeout(resolve, commandPause));
				}
			} else {
				await typeAndExecuteCommand(target);
			}
		}
	}

	export async function typeAndExecuteCommand(command: string) {
		clearPrompt();
		await typeText(command, typingDelay);
		await new Promise((resolve) => setTimeout(resolve, commandPause));
		executeCommandDirectly(command);
	}

	export async function typeText(text: string, delay: number = typingDelay) {
		for (const char of text) {
			prompt += char;
			await new Promise((resolve) => setTimeout(resolve, delay));
		}
	}

	export function clearPrompt() {
		prompt = '';
	}

	export function clearOutput() {
		output = [];
	}

	export function executeCommandDirectly(command: string) {
		executeCommand(command);
		prompt = '';
		scrollToBottom();
	}

	function scrollToBottom() {
		if (terminalElement) {
			setTimeout(() => {
				terminalElement.scrollTop = terminalElement.scrollHeight;
			}, 10);
		}
	}

	function executeCommand(command: string) {
		const trimmedCommand = command.trim();

		if (trimmedCommand === '') return;

		output = [
			...output,
			{ type: 'command', content: trimmedCommand, user, host, path: currentPath }
		];

		if (trimmedCommand === 'clear') {
			output = [];
			return;
		}

		const [cmd, ...args] = trimmedCommand.split(' ');

		if (cmd === 'cd') {
			handleCd(args[0] || '~/');
		} else if (cmd === 'ls') {
			handleLs();
		} else if (cmd === 'run') {
			handleRun();
		} else if (cmd === 'open') {
			if (args[0]) {
				handleOpen(args[0]);
			} else {
				output = [...output, { type: 'result', content: 'open: missing URL argument' }];
			}
		} else if (commands[trimmedCommand]) {
			output = [...output, { type: 'result', content: commands[trimmedCommand] }];
		} else {
			output = [
				...output,
				{ type: 'result', content: `zsh: command not found: ${trimmedCommand}` }
			];
		}
	}

	function handleCd(targetPath: string) {
		if (targetPath === '~' || targetPath === '~/') {
			currentPath = '~/';
			path = currentPath;
			updateUrl('/');
		} else if (targetPath === '..') {
			if (currentPath !== '~/') {
				currentPath = '~/';
				path = currentPath;
				updateUrl('/');
			}
		} else if (targetPath.startsWith('~/')) {
			if (directories[targetPath]) {
				currentPath = targetPath;
				path = currentPath;
				updateUrl('/' + targetPath.replace('~/', ''));
			} else {
				output = [
					...output,
					{ type: 'result', content: `cd: no such file or directory: ${targetPath}` }
				];
			}
		} else {
			const fullPath = currentPath === '~/' ? `~/${targetPath}` : `${currentPath}/${targetPath}`;
			if (directories[fullPath]) {
				currentPath = fullPath;
				path = currentPath;
				updateUrl('/' + fullPath.replace('~/', ''));
			} else {
				output = [
					...output,
					{ type: 'result', content: `cd: no such file or directory: ${targetPath}` }
				];
			}
		}
	}

	function handleLs() {
		const items = directories[currentPath] || [];
		if (items.length > 0) {
			output = [...output, { type: 'result', content: items.join('  ') }];
		}
	}

	function handleRun() {
		const runOutput = runOutputs[currentPath];
		if (runOutput) {
			output = [...output, { type: 'result', content: runOutput }];
		} else {
			output = [...output, { type: 'result', content: 'Nothing to run in this directory.' }];
		}
	}
	function handleOpen(url: string) {
		if (!url.startsWith('http://') && !url.startsWith('https://')) {
			url = 'https://' + url;
		}
		window.open(url, '_blank');
	}

	function updateUrl(newPath: string) {
		if (typeof window !== 'undefined') {
			const url = newPath === '/' ? '/' : newPath;
			window.history.pushState({}, '', url);
		}
	}

	function setDirectoryFromUrl() {
		// Only set directory from URL if no initialPath is provided
		if (initialPath) return;

		if (typeof window !== 'undefined') {
			const urlPath = window.location.pathname;
			if (urlPath === '/') {
				currentPath = '~/';
				path = currentPath;
			} else if (urlPath === '/about') {
				if (directories['~/about']) {
					currentPath = '~/about';
					path = currentPath;
				}
			} else if (urlPath === '/projects') {
				if (directories['~/projects']) {
					currentPath = '~/projects';
					path = currentPath;
				}
			}
		}
	}

	function handlePopState() {
		setDirectoryFromUrl();
	}

	function handleKeydown(event: KeyboardEvent) {
		if (event.metaKey || event.ctrlKey) {
			return;
		}

		if (event.key === 'Backspace') {
			prompt = prompt.slice(0, -1);
		} else if (event.key === 'Enter') {
			executeCommand(prompt);
			prompt = '';
			scrollToBottom();
			event.preventDefault();
		} else if (event.key.length === 1) {
			prompt += event.key;
			event.preventDefault();
		}
	}

	onMount(() => {
		// Set initial directory from prop or URL
		if (initialPath) {
			currentPath = initialPath;
			path = initialPath;
		} else {
			setDirectoryFromUrl();
		}

		window.addEventListener('keydown', handleKeydown);
		window.addEventListener('popstate', handlePopState);
		return () => {
			window.removeEventListener('keydown', handleKeydown);
			window.removeEventListener('popstate', handlePopState);
		};
	});
</script>

<div class="terminal" bind:this={terminalElement}>
	{#each output as item}
		{#if item.type === 'command'}
			<div class="command-line">
				<span class="user">{item.user}@{item.host}</span>
				<span class="path">{item.path}</span>
				<span class="command-text">{item.content}</span>
			</div>
		{:else}
			<div class="output-line">
				{#each parseLinks(item.content) as part}
					{#if part.type === 'link'}
						<button
							class="terminal-link terminal-link-{part.linkType}"
							on:click={() => handleLinkClick(part.linkType, part.target)}
						>
							{part.content}
						</button>
					{:else}
						{part.content}
					{/if}
				{/each}
			</div>
		{/if}
	{/each}
	<div class="prompt">
		<span class="user">{user}@{host}</span>
		<span class="path">{path}</span>
		<span class="input">{prompt}▉</span>
	</div>
</div>

<style>
	@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:ital,wght@0,100..800;1,100..800&display=swap');

	.terminal {
		font-family: 'JetBrains Mono', monospace;
		font-size: 20px;
		font-weight: 500;
		height: 100vh;
		overflow-y: hidden;
		overflow-x: hidden;
		padding: 20px;
		box-sizing: border-box;
	}

	@media (max-width: 768px) {
		.terminal {
			font-size: 12px;
			padding: 10px;
		}
	}

	@media (max-width: 480px) {
		.terminal {
			font-size: 10px;
			padding: 5px;
		}
	}

	.prompt,
	.command-line {
		display: flex;
		gap: 8px;
	}

	.command-line {
		margin-bottom: 4px;
	}

	.command-text {
		color: var(--ctp-mocha-text);
	}

	.output-line {
		color: var(--ctp-mocha-text);
		margin-bottom: 4px;
		white-space: pre;
		overflow-x: hidden;
	}

	.user {
		color: var(--ctp-mocha-blue);
	}

	.path {
		color: var(--ctp-mocha-pink);
	}

	.git {
		color: var(--ctp-mocha-lavender);
	}

	.input {
		color: var(--ctp-mocha-text);
	}

	.terminal-link {
		background: none;
		border: none;
		padding: 0;
		margin: 0;
		font-family: inherit;
		font-size: inherit;
		font-weight: inherit;
		cursor: pointer;
		text-decoration: underline;
		color: var(--ctp-mocha-lavender);
	}

	.terminal-link:hover {
		color: var(--ctp-mocha-pink);
	}

	.terminal-link-cmd {
		color: var(--ctp-mocha-lavender);
	}

	.terminal-link-cmd:hover {
		color: var(--ctp-mocha-pink);
	}
</style>
