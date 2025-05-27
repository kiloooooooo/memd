<script lang="ts">
	import { browser } from '$app/environment';
	import { onDestroy, onMount } from 'svelte';
	import CodeEditor from '../components/CodeEditor.svelte';
	import MdPreview from '../components/MdPreview.svelte';
	import FileOperations from '../components/FileOperations.svelte';
	import CommandHelp from '../components/CommandHelp.svelte';
	// marked のようなMarkdownパーサーをインポート（別途インストールが必要）
	// import { marked } from 'marked';

	let myCode = '';
	let isPreviewMode = false;

	let showCommandHelp = false;

	// $: hasUnsavedChanges = false;

	// エディタのtextarea要素へのRef
	let codeEditorRef: any = null;

	// エディタのカーソル位置を保持
	let savedSelectionStart = 0;
	let savedSelectionEnd = 0;

	// ファイルを開いたときに呼び出されるコールバック関数
	function handleFileOpen(content: string) {
		myCode = content;
		// hasUnsavedChanges = false;
	}

	// // ファイルが保存されたときに呼び出されるコールバック関数
	// function handleFileSave() {
	// 	hasUnsavedChanges = false;
	// }

	// CodeEditorから呼び出されるコールバック関数
	function handleCodeUpdate(newCode: string) {
		myCode = newCode;
		// hasUnsavedChanges = true;
		// console.log('Code updated:', myCode);
	}

	// CodeEditorのカーソル移動コールバック
	function handleCursorMove(start: number, end: number) {
		savedSelectionStart = start;
		savedSelectionEnd = end;
	}

	// CodeEditorから呼び出されるプレビュー切り替え用コールバック関数
	function handleTogglePreview() {
		isPreviewMode = !isPreviewMode;
		console.log('Preview mode:', isPreviewMode);

		if (!isPreviewMode) {
			// プレビューからエディタに戻る際に、カーソル位置を復元し、フォーカスする
			requestAnimationFrame(() => {
				if (codeEditorRef) {
					// focusAndRestoreCursor 関数を呼び出し、保存したカーソル位置を引数として渡す
					codeEditorRef.focusAndRestoreCursor(savedSelectionStart, savedSelectionEnd);
					console.log('Restored selection and focused.');
				}
			});
		}
	}

	// キーボードイベントを監視
	function handleKeydown(event: KeyboardEvent) {
		// Ctrl+E または Ctrl+P でプレビュー切り替え
		if ((event.ctrlKey || event.metaKey) && (event.key === 'e' || event.key === 'p')) {
			event.preventDefault(); // デフォルトの動作を抑制
			handleTogglePreview(); // 表示切替
			return; // 他のキーハンドリングに進まない
		}
		// Ctrl+Shift+/でヘルプ表示
		if ((event.ctrlKey || event.metaKey) && event.key === '/') {
			event.preventDefault();
			console.log('show command help');
			showCommandHelp = !showCommandHelp;
		}
	}

	// コンポーネントのマウント時にイベントリスナーを追加
	onMount(() => {
		if (browser) {
			window.addEventListener('keydown', handleKeydown);
		}
	});

	// コンポーネントの破棄時にイベントリスナーを削除
	onDestroy(() => {
		if (browser) {
			window.removeEventListener('keydown', handleKeydown);
		}
	});
</script>

<FileOperations onFileOpen={handleFileOpen} markdownCode={myCode} />
<CommandHelp showHelp={showCommandHelp} />
{#if isPreviewMode}
	<MdPreview markdownCode={myCode} />
{:else}
	<CodeEditor
		bind:this={codeEditorRef}
		code={myCode}
		onCodeUpdate={handleCodeUpdate}
		onCursorMoved={handleCursorMove}
	/>
{/if}
