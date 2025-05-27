<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	export let onFileOpen = (content: string) => {};
	export let markdownCode = '';

	// export let hasUnsavedChanges = true;

	let directoryHandle: FileSystemDirectoryHandle | null = null;
	let currentFileHandle: FileSystemFileHandle | null = null;
	let isNewFile = true;
	let showFileList = false;
	let fileList: FileSystemFileHandle[] = [];
	let selectedFileIndex = -1;
	let hasUnsavedChanges = false;
	let originalFileContent = '';

	// エディタの内容が変更されたときにフラグを立てる;
	$: if (!showFileList) {
		hasUnsavedChanges = markdownCode !== originalFileContent;

		if (browser) {
			if (hasUnsavedChanges) {
				window.addEventListener('beforeunload', handleUnload);
				window.addEventListener('popstate', handlePopState);
			} else {
				window.removeEventListener('beforeunload', handleUnload);
				window.removeEventListener('popstate', handlePopState);
			}
		}
	}

	onMount(() => {
		if (browser) {
			window.addEventListener('keydown', handleGlobalKeyDown);
			// loadFiles();
		}
	});

	onDestroy(() => {
		if (browser) {
			window.removeEventListener('keydown', handleGlobalKeyDown);
		}
	});

	// ページを離れる際に警告するコールバック
	function handleUnload(e: BeforeUnloadEvent) {
		e.preventDefault();
		// 表示するメッセージを返す
		// が、今は反映されないので空文字列を返す
		return '';
	}

	// ブラウザバック時に警告するコールバック
	function handlePopState(e: PopStateEvent) {
		e.preventDefault();
	}

	async function loadFiles() {
		console.log('Loading files...');
		if (!('showDirectoryPicker' in window)) {
			alert('お使いのブラウザは Directory Picker API をサポートしていません。');
			return;
		}

		try {
			if (!directoryHandle) {
				/* @ts-ignore */
				directoryHandle = await window.showDirectoryPicker();
			}
			const newFileList: FileSystemFileHandle[] = [];
			/* @ts-ignore */
			for await (const entry of directoryHandle!.values()) {
				if (entry.kind === 'file') {
					newFileList.push(entry);
				}
			}
			fileList = newFileList;
		} catch (error: any) {
			if (error.name !== 'AbortError') {
				console.error('フォルダへのアクセス中にエラーが発生しました:', error);
				alert('フォルダへのアクセス中にエラーが発生しました。');
			}
		}
	}

	async function saveFile(forceDialog = false) {
		if (!('showSaveFilePicker' in window)) {
			alert('お使いのブラウザは Save File Picker API をサポートしていません。');
			return;
		}

		try {
			if (isNewFile || forceDialog || !currentFileHandle) {
				/* @ts-ignore */
				currentFileHandle = await window.showSaveFilePicker();
				isNewFile = false;
			}

			if (currentFileHandle) {
				const writable = await currentFileHandle.createWritable();
				await writable.write(markdownCode);
				await writable.close();
				hasUnsavedChanges = false;
				originalFileContent = markdownCode;
				// onFileSave();
				// alert('ファイルを保存しました。');
			}
		} catch (error: any) {
			if (error.name !== 'AbortError') {
				console.error('ファイルの保存中にエラーが発生しました:', error);
				alert('ファイルの保存中にエラーが発生しました。');
			}
		}
	}

	async function openFileFromHandle(fileHandle: FileSystemFileHandle) {
		if (hasUnsavedChanges && !confirm('未保存の変更があります。続行しますか？')) {
			return;
		}

		try {
			const file = await fileHandle.getFile();
			const fileContent = await file.text();
			// markdownCode = fileContent;
			currentFileHandle = fileHandle;
			isNewFile = false;
			originalFileContent = fileContent;
			onFileOpen(fileContent);
			hasUnsavedChanges = false;
			// alert('ファイルを開きました。');
		} catch (error) {
			console.error('ファイルの読み込み中にエラーが発生しました:', error);
			alert('ファイルの読み込み中にエラーが発生しました。');
		}
	}

	async function openSelectedFile() {
		if (0 <= selectedFileIndex && selectedFileIndex < fileList.length) {
			await openFileFromHandle(fileList[selectedFileIndex]);
			showFileList = false;
			selectedFileIndex = -1;
		}
	}

	function handleGlobalKeyDown(event: KeyboardEvent) {
		if ((event.ctrlKey || event.metaKey) && event.key === 's') {
			event.preventDefault();
			saveFile();
		} else if ((event.ctrlKey || event.metaKey) && event.key === 'o') {
			event.preventDefault();
			if (directoryHandle == null) {
				loadFiles();
			}
			showFileList = !showFileList;
			selectedFileIndex = -1; // ファイル一覧表示時に選択をリセット
		} else if (showFileList) {
			if (event.key === 'ArrowDown' || event.key.toLowerCase() === 'j') {
				event.preventDefault();
				selectedFileIndex = Math.min(selectedFileIndex + 1, fileList.length - 1);
			} else if (event.key === 'ArrowUp' || event.key.toLowerCase() === 'k') {
				event.preventDefault();
				selectedFileIndex = Math.max(selectedFileIndex - 1, 0);
			} else if (event.key === 'Enter') {
				openSelectedFile();
			} else if (event.key === 'Escape') {
				showFileList = false;
				selectedFileIndex = -1;
			}
		}
	}
</script>

<div class="file-status-container">
	<!-- <div></div> -->
	<div class="file-status">
		{#if isNewFile}
			新規ファイル
		{:else}
			{currentFileHandle?.name}
		{/if}

		{#if hasUnsavedChanges}
			<div class="unsaved-marker"></div>
		{:else}
			<div class="unsaved-marker-placeholder"></div>
		{/if}
	</div>
	<div class="help-shortcut">
		<span class="help-label">ヘルプ</span>
		<span class="instruction-key">Ctrl</span>
		<span class="instruction-key-combination">+</span>
		<span class="instruction-key">/</span>
	</div>
</div>
{#if showFileList}
	<div class="file-operations-container">
		<div class="file-list-overlay">
			<h4>ファイル一覧</h4>
			<ul>
				{#each fileList as file, index}
					<li class:selected={index === selectedFileIndex}>
						{file.name}
					</li>
				{/each}
			</ul>
			{#if fileList.length === 0}
				<p class="message">フォルダ内にファイルが見つかりません。</p>
			{/if}
			<p class="instructions">↑↓/J/K: 選択, Enter: 開く, Esc: 閉じる</p>
		</div>
	</div>
{/if}

<style>
	.help-shortcut {
		color: #ccc;
		display: flex;
		align-items: center;
		font-size: 0.9em;
		gap: 8px;
		padding: 0 16px;
	}

	.help-label {
		margin-right: 8px;
	}

	.instruction-key {
		background-color: #555;
		border-radius: 4px;
		padding: 0 8px;
	}

	.file-status-container {
		position: fixed;
		top: 0;
		width: 100%;
		height: 40px;
		background-color: #222;
		display: flex;
		justify-content: space-between;
	}

	.file-status {
		height: 100%;
		padding: 0 20px;
		display: flex;
		align-items: center;
		color: #ccc;
		font-size: 0.9em;
		gap: 16px;
	}

	.unsaved-marker-placeholder {
		width: 10px;
		height: 10px;
		background-color: #00000000;
	}

	.unsaved-marker {
		width: 10px;
		height: 10px;
		border-radius: 100%;
		background-color: #fff;
	}

	.file-operations-container {
		position: fixed;
		width: 100%;
		height: 400px; /* 適宜調整 */
	}

	.file-list-overlay {
		position: absolute;
		top: 50%;
		left: 50%;
		min-width: 75%;
		transform: translate(-50%, -50%);
		background-color: #222;
		border: 1px solid #555;
		padding: 20px 0;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
		z-index: 10;
		text-align: start;
		color: #fff;
	}

	.file-list-overlay h4 {
		margin-top: 0;
		margin-left: 20px;
	}

	.file-list-overlay ul {
		list-style: none;
		padding: 0;
		margin: 10px 0;
		overflow-y: auto;
		text-align: left;
	}

	.file-list-overlay li {
		padding: 2px 20px;
		cursor: pointer;
		border: solid 1px #00000000;
	}

	.file-list-overlay li.selected {
		background-color: #04395e;
		border: solid 1px #0078d4;
		color: #fff;
	}

	.file-list-overlay .message {
		margin-left: 20px;
	}

	.file-list-overlay .instructions {
		margin-top: 15px;
		margin-left: 20px;
		font-size: 0.9em;
		color: #777;
	}
</style>
