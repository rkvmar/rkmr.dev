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
			let previousPath: string | null = null;
			let history: string[] = [];
			let historyIndex = -1;

	export let typingDelay = 50;
	export let commandPause = 100;

	let terminalElement: HTMLElement;

	const directories = new Map<string, string[]>([
			['~', ['about', 'projects']],
			['~/about', []],
			['~/projects', []]
		]);

const commands: Record<string, string> = {
		help: 'Available commands: help, clear, cd, ls, run, open, history',
		clear: '',
		ls: '',
		cd: '',
		run: '',
		open: '',
		fimsh: 'he? he romtater??',
		history: ''
	};

	const runOutputs: Record<string, string> = {
			'~/': '         ░██                                           ░██                       \n         ░██                                           ░██                       \n░██░████ ░██    ░██░█████████████  ░██░████      ░████████  ░███████  ░██    ░██ \n░███     ░██   ░██ ░██   ░██   ░██ ░███         ░██    ░██ ░██    ░██ ░██    ░██ \n░██      ░███████  ░██   ░██   ░██ ░██          ░██    ░██ ░█████████  ░██  ░██  \n░██      ░██   ░██ ░██   ░██   ░██ ░██          ░██   ░███ ░██          ░██░██   \n░██      ░██    ░██░██   ░██   ░██ ░██      ░██  ░█████░██  ░███████     ░███    \n\n I make stupid stuff on the interwebs\n[link:cmd:cd ~/about && clear && run]About[/link] | [link:cmd:cd ~/projects && clear && run]Projects[/link] | [link:cmd:open https://github.com/rkvmar]Github[/link]',
			'~/about':
				'           ░██                                 ░██    \n           ░██                                 ░██    \n ░██████   ░████████   ░███████  ░██    ░██ ░████████ \n      ░██  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n ░███████  ░██    ░██ ░██    ░██ ░██    ░██    ░██    \n░██   ░██  ░███   ░██ ░██    ░██ ░██   ░███    ░██    \n ░█████░██ ░██░█████   ░███████   ░█████░██     ░████\n\nHai!\nI make things on the interwebs.\nyou can find some of my projects in [link:cmd:cd /projects && clear && run]Projects[/link]\n[link:cmd:cd && clear && run]Home[/link]',
			'~/projects':
				'                                 ░██                          ░██               \n                                                              ░██               \n░████████  ░██░████  ░███████    ░██  ░███████   ░███████  ░████████  ░███████  \n░██    ░██ ░███     ░██    ░██   ░██ ░██    ░██ ░██    ░██    ░██    ░██        \n░██    ░██ ░██      ░██    ░██   ░██ ░█████████ ░██           ░██     ░███████  \n░███   ░██ ░██      ░██    ░██   ░██ ░██        ░██    ░██    ░██           ░██ \n░██░█████  ░██       ░███████    ░██  ░███████   ░███████      ░████  ░███████  \n░██                              ░██                                            \n░██                            ░███\n\n[link:cmd:open https://headways.rkmr.dev]Headways[/link] - Realtime transit system explorer\n[link:cmd:open kanji.rkmr.dev]Kanji[/link] - Kanji training website based on "Kanji Look + Learn" workbook\n[link:cmd:open https://github.com/rkvmar/CelesteSplitsGenerator]CSG[/link] - Speedrun splits generator for Celeste\nBayAreaGame - Unofficial App to play Jet Lag: The Game (Link coming soon)\n\n[link:cmd:cd && clear && run]Home[/link]'
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
				linkType: match[1] || '', // 'url' or 'cmd'
				target: match[2] || '',
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

	async function handleLinkClick(linkType: string | undefined, target: string | undefined) {
			if (linkType === 'url' && target) {
				window.open(target, '_blank');
			} else if (linkType === 'cmd' && target) {
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
		if (command.trim()) {
			history = [...history, command.trim()];
			historyIndex = history.length;
		}
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
		} else if (cmd === 'history') {
						handleHistory();
					} else if (commands[trimmedCommand]) {
						output = [...output, { type: 'result', content: commands[trimmedCommand] }];
		} else {
			output = [
				...output,
				{ type: 'result', content: `zsh: command not found: ${trimmedCommand}` }
			];
		}
	}

	/**
	 * Resolve a path like a real shell.
	 * - Supports `~`, `~/` as home
	 * - Supports `.`, `./` as current
	 * - Supports `..`, `../` as parent
	 * - Supports `/` as root (maps to `~`)
	 * - Supports `-` as previous directory
	 * - Handles multi-segment paths: `../projects`, `~/about/..`, etc.
	 * - Handles trailing slashes
	 */
	function resolvePath(input: string): string | null {
		if (input === '-' && previousPath !== null) {
			return previousPath;
		}

		let resolved: string;

		if (input === '/' || input === '//') {
			// Treat / as root -> ~
			resolved = '~';
		} else if (input.startsWith('~/') || input === '~') {
			// Absolute from home
			resolved = input === '~' ? '~' : '~' + input.slice(1);
		} else if (input.startsWith('/')) {
			// Absolute path from / means ~/...
			resolved = '~' + input;
		} else {
			// Relative to current path
			resolved = currentPath === '~/' || currentPath === '~'
				? '~/' + input
				: currentPath + '/' + input;
		}

		// Normalize: split by /, process each segment
		const parts = resolved.split('/');
		const stack: string[] = [];

		for (const part of parts) {
			if (part === '' || part === '.') {
				// skip empty (from leading/trailing slash) and .
				continue;
			}
			if (part === '..') {
				if (stack.length > 0 && stack[stack.length - 1] !== '~') {
					stack.pop();
				} else if (stack.length === 0 || stack[stack.length - 1] === '~') {
					// Can't go above ~, so ignore
					continue;
				}
			} else {
				stack.push(part);
			}
		}

		// Reconstruct: if it started with ~, preserve that
			if (resolved.startsWith('~')) {
				if (stack.length === 0 || (stack.length === 1 && stack[0] === '~')) {
					return '~';
				}
				return '~/' + stack.slice(1).join('/');
			}

		return stack.join('/');
	}

	function handleCd(rawTarget: string) {
		// Trim and normalize
		let target = rawTarget.trim();

		// `cd` with no args goes home
		if (!target) {
			target = '~';
		}

		// `cd -` goes to previous dir
		if (target === '-') {
			if (previousPath === null) {
				output = [...output, { type: 'result', content: 'cd: no previous directory' }];
				return;
			}
			const resolved = resolvePath('-');
			if (resolved && directories.has(resolved)) {
				// Swap previous and current
				const oldPath = currentPath;
				currentPath = resolved;
				path = currentPath;
				previousPath = oldPath;
				updateUrl(pathToUrl(currentPath));
				return;
			} else {
				output = [...output, { type: 'result', content: `cd: no such file or directory: ${rawTarget}` }];
				return;
			}
		}

		const resolved = resolvePath(target);

		if (resolved === null) {
			output = [...output, { type: 'result', content: `cd: no such file or directory: ${rawTarget}` }];
			return;
		}

		if (directories.has(resolved)) {
			previousPath = currentPath;
			currentPath = resolved;
			path = currentPath;
			updateUrl(pathToUrl(currentPath));
		} else {
			output = [...output, { type: 'result', content: `cd: no such file or directory: ${rawTarget}` }];
		}
	}

	function handleLs() {
			const items = directories.get(currentPath) || [];
			if (items.length > 0) {
				output = [...output, { type: 'result', content: items.join('  ') }];
			}
		}

	function handleRun() {
			// Normalize path for lookup: map ~ or ~/ to ~/ for runOutputs keys
			const lookupPath = currentPath === '~' ? '~/' : currentPath;
			const runOutput = runOutputs[lookupPath];
			if (runOutput) {
				output = [...output, { type: 'result', content: runOutput }];
			} else {
				output = [...output, { type: 'result', content: 'Nothing to run in this directory.' }];
			}
		}
	function handleHistory() {
			if (history.length === 0) {
				output = [...output, { type: 'result', content: 'No commands in history.' }];
				return;
			}
			const numbered = history.map((cmd, i) => `${i + 1}  ${cmd}`).join('\n');
			output = [...output, { type: 'result', content: numbered }];
		}

		function handleOpen(url: string) {
			if (!url.startsWith('http://') && !url.startsWith('https://')) {
				url = 'https://' + url;
			}
			window.open(url, '_blank');
		}

	function pathToUrl(p: string): string {
			if (p === '~' || p === '~/') return '/';
			return '/' + p.replace('~/', '');
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
				// Normalize: remove trailing slash (except for root)
				const normalized = urlPath === '/' ? '/' : urlPath.replace(/\/+$/, '');

				if (normalized === '/') {
					currentPath = '~';
					path = currentPath;
				} else {
					const dirPath = '~' + normalized;
					if (directories.has(dirPath)) {
						currentPath = dirPath;
						path = currentPath;
					}
				}
			}
		}

	function handleTabCompletion() {
			if (!prompt.trim()) return;

			const parts = prompt.trim().split(' ');
			const cmd = parts[0];

			// Only complete paths for cd and ls
			if ((cmd === 'cd' || cmd === 'ls') && parts.length > 1) {
				const partial = parts.slice(1).join(' ');

				// Get all possible completions from directories
				const allDirs = Array.from(directories.keys());
				const currentDirEntries = directories.get(currentPath) || [];

				// Build a list of completable names from current directory
				const candidates = currentDirEntries.filter((name) => name.startsWith(partial));

				if (candidates.length === 1) {
					parts[parts.length - 1] = candidates[0];
					prompt = parts.join(' ') + ' ';
				} else if (candidates.length > 1) {
					// Find common prefix and show possibilities
					const commonPrefix = findLongestCommonPrefix(candidates);
					if (commonPrefix && commonPrefix.length > partial.length) {
						parts[parts.length - 1] = commonPrefix;
						prompt = parts.join(' ');
					} else {
						output = [...output, { type: 'result', content: candidates.join('  ') }];
					}
				}
			}
		}

		function findLongestCommonPrefix(strings: string[]): string {
			if (strings.length === 0) return '';
			let prefix = strings[0];
			for (let i = 1; i < strings.length; i++) {
				while (strings[i].indexOf(prefix) !== 0) {
					prefix = prefix.slice(0, -1);
					if (prefix === '') return '';
				}
			}
			return prefix;
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
				if (prompt.trim()) {
					history = [...history, prompt.trim()];
					historyIndex = history.length;
				}
				executeCommand(prompt);
				prompt = '';
				scrollToBottom();
				event.preventDefault();
			} else if (event.key === 'ArrowUp') {
				event.preventDefault();
				if (historyIndex > 0) {
					historyIndex--;
					prompt = history[historyIndex];
				}
			} else if (event.key === 'ArrowDown') {
				event.preventDefault();
				if (historyIndex < history.length - 1) {
					historyIndex++;
					prompt = history[historyIndex];
				} else {
					historyIndex = history.length;
					prompt = '';
				}
			} else if (event.key === 'Tab') {
				event.preventDefault();
				handleTabCompletion();
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
							class="terminal-link terminal-link-{part.linkType || 'cmd'}"
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
