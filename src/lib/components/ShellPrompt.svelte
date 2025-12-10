<script lang="ts">
	import { onMount } from 'svelte';

	export let user = 'rkmr';
	export let host = 'rkmr.dev';
	export let path = '~/';
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
		help: 'Available commands: help, clear, cd, ls, run',
		clear: '',
		ls: '',
		cd: '',
		run: ''
	};

	const directoryContent = {
		'~/about':
			'About Me\n--------\nI am a passionate software engineer with expertise in web development,\nopen source projects, and modern programming languages.\n\nSkills: JavaScript, TypeScript, Svelte, React, Node.js, Python',
		'~/projects':
			'My Projects\n-----------\n1. rkmr.dev - Personal portfolio website\n2. Terminal Portfolio - Interactive shell interface\n3. Various open source contributions on GitHub\n\nVisit: github.com/rkmr'
	};

	const runOutputs = {
		'~/': '         ░██                                           ░██                       \n         ░██                                           ░██                       \n░██░████ ░██    ░██░█████████████  ░██░████      ░████████  ░███████  ░██    ░██ \n░███     ░██   ░██ ░██   ░██   ░██ ░███         ░██    ░██ ░██    ░██ ░██    ░██ \n░██      ░███████  ░██   ░██   ░██ ░██          ░██    ░██ ░█████████  ░██  ░██  \n░██      ░██   ░██ ░██   ░██   ░██ ░██          ░██   ░███ ░██          ░██░██   \n░██      ░██    ░██░██   ░██   ░██ ░██      ░██  ░█████░██  ░███████     ░███    \n\n I make stupid stuff on the interweb\n[link:cmd:cd ~/about && run]About[/link] | [link:cmd:cd ~/projects && run]Projects[/link] | [link:url:https://github.com/rkvmar]Github[/link]',
		'~/about':
			'           ░██                                 ░██    \n           ░██                                 ░██    \n ░██████   ░████████   ░███████  ░██    ░██ ░████████ \n      ░██  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n ░███████  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n░██   ░██  ░███   ░██ ░██    ░██ ░██   ░███    ░██    \n ░█████░██ ░██░█████   ░███████   ░█████░██     ░████\n\nI thinkn\'t, therefore I amn\'t',
		'~/projects':'                                 ░██                          ░██               \n                                                              ░██               \n░████████  ░██░████  ░███████    ░██  ░███████   ░███████  ░████████  ░███████  \n░██    ░██ ░███     ░██    ░██   ░██ ░██    ░██ ░██    ░██    ░██    ░██        \n░██    ░██ ░██      ░██    ░██   ░██ ░█████████ ░██           ░██     ░███████  \n░███   ░██ ░██      ░██    ░██   ░██ ░██        ░██    ░██    ░██           ░██ \n░██░█████  ░██       ░███████    ░██  ░███████   ░███████      ░████  ░███████  \n░██                              ░██                                            \n░██                            ░███\n\nYou think I do stuff?'
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
				const commands = target.split('&&').map(cmd => cmd.trim());
				for (const cmd of commands) {
					await typeAndExecuteCommand(cmd);
					await new Promise(resolve => setTimeout(resolve, commandPause));
				}
			} else {
				await typeAndExecuteCommand(target);
			}
		}
	}

	export async function typeAndExecuteCommand(command: string) {
		clearPrompt();
		await typeText(command, typingDelay);
		await new Promise(resolve => setTimeout(resolve, commandPause));
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
		} else if (cmd === 'about' && currentPath === '~/about') {
			output = [...output, { type: 'result', content: directoryContent['~/about'] }];
		} else if (cmd === 'projects' && currentPath === '~/projects') {
			output = [...output, { type: 'result', content: directoryContent['~/projects'] }];
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
		} else if (targetPath === '..') {
			if (currentPath !== '~/') {
				currentPath = '~/';
				path = currentPath;
			}
		} else if (targetPath.startsWith('~/')) {
			if (directories[targetPath]) {
				currentPath = targetPath;
				path = currentPath;
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
		window.addEventListener('keydown', handleKeydown);
		return () => {
			window.removeEventListener('keydown', handleKeydown);
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
		padding: 20px;
		box-sizing: border-box;
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
		white-space: pre-wrap;
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
