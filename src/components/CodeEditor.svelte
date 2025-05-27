<script lang="ts">
	export let code = '';
	export let tabSize = 2; // タブサイズ（スペース数）

	// 親コンポーネントから渡されるコールバック関数
	// 初期値として空の関数を設定することで、親が渡さなかった場合でもエラーにならないようにする
	export let onCodeUpdate = (newCode: string) => {};
	export let onCursorMoved = (start: number, end: number) => {};

	let textarea: HTMLTextAreaElement;
	let lineNumbersContainer: HTMLDivElement;
	let lineNumbers = 1;

	// 内部的に管理するカーソル位置プロパティ
	// on:inputやon:keydownで更新し、常に最新の状態に保つ
	let currentSelectionStart = 0;
	let currentSelectionEnd = 0;

	// 外部からフォーカス及びカーソル位置設定を行う関数
	export function focusAndRestoreCursor(start: number, end: number) {
		if (textarea) {
			textarea.focus();
			textarea.selectionStart = start;
			textarea.selectionEnd = end;
			// 内部状態も更新
			currentSelectionStart = start;
			currentSelectionEnd = end;

			// コールバックを呼ぶ
			onCursorMoved(currentSelectionStart, currentSelectionEnd);
		}
	}

	// code の変更を監視し、行番号を更新
	$: {
		lineNumbers = code.split('\n').length;
		// dispatch の代わりに、onCodeUpdate コールバックを呼び出す
		onCodeUpdate(code);
	}

	// textarea のスクロールイベントを監視
	function handleTextareaScroll() {
		if (lineNumbersContainer && textarea) {
			lineNumbersContainer.scrollTop = textarea.scrollTop;
		}
	}

	function getCurrentLine(text: string, cursor: number): [string, number, number] {
		const lineStart = text.lastIndexOf('\n', cursor - 1) + 1;
		const nextLF = text.indexOf('\n', cursor);
		const lineEnd = nextLF < 0 ? text.length : text.indexOf('\n', cursor);
		return [text.substring(lineStart, lineEnd), lineStart, lineEnd];
	}

	function handleKeydown(event: KeyboardEvent) {
		currentSelectionStart = textarea.selectionStart;
		currentSelectionEnd = textarea.selectionEnd;
		const value = textarea.value;

		if (event.key === 'Tab' && !event.shiftKey) {
			// Shiftキーが押されていないTab
			event.preventDefault(); // デフォルトのタブ入力を防ぐ
			const indent = ' '.repeat(tabSize);

			if (currentSelectionStart === currentSelectionEnd) {
				// 単一行のインデント
				const [currentLine, lineStart, lineEnd] = getCurrentLine(value, currentSelectionStart);
				// リストであって、何も入力されていない場合は行頭にインデントを付加
				if (currentLine.match(/^\s*(-|\*|\d+\.)\s+$/)) {
					const lineStart = value.lastIndexOf('\n', currentSelectionStart - 1) + 1;
					code = value.substring(0, lineStart) + indent + value.substring(lineStart);
				} else {
					code =
						value.substring(0, currentSelectionStart) +
						indent +
						value.substring(currentSelectionEnd);
				}
				// カーソル位置を更新
				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + indent.length;
				});
			} else {
				// 複数行のインデント
				const linesBeforeSelection = value.substring(0, currentSelectionStart).split('\n');
				const firstLineStart =
					currentSelectionStart - linesBeforeSelection[linesBeforeSelection.length - 1].length;

				const selectedText = value.substring(firstLineStart, currentSelectionEnd);
				const indentedText = selectedText
					.split('\n')
					.map((line) => indent + line)
					.join('\n');

				code =
					value.substring(0, firstLineStart) + indentedText + value.substring(currentSelectionEnd);

				// 選択範囲を維持
				requestAnimationFrame(() => {
					textarea.selectionStart = firstLineStart;
					textarea.selectionEnd = firstLineStart + indentedText.length;
				});
			}
		} else if (event.key === 'Tab' && event.shiftKey) {
			// Shift+Tab
			event.preventDefault(); // デフォルトのタブ入力を防ぐ

			const indent = ' '.repeat(tabSize);

			if (currentSelectionStart === currentSelectionEnd) {
				// 単一行のインデント解除
				const lineStart = value.lastIndexOf('\n', currentSelectionStart - 1) + 1;
				const currentLine = value.substring(lineStart, currentSelectionEnd);

				if (currentLine.startsWith(indent)) {
					code =
						value.substring(0, lineStart) +
						currentLine.substring(indent.length) +
						value.substring(currentSelectionEnd);
					requestAnimationFrame(() => {
						textarea.selectionStart = textarea.selectionEnd = currentSelectionStart - indent.length;
					});
				}
			} else {
				// 複数行のインデント解除
				const linesBeforeSelection = value.substring(0, currentSelectionStart).split('\n');
				const firstLineStart =
					currentSelectionStart - linesBeforeSelection[linesBeforeSelection.length - 1].length;

				const selectedText = value.substring(firstLineStart, currentSelectionEnd);
				const unindentedText = selectedText
					.split('\n')
					.map((line) => {
						return line.startsWith(indent) ? line.substring(indent.length) : line;
					})
					.join('\n');

				code =
					value.substring(0, firstLineStart) +
					unindentedText +
					value.substring(currentSelectionEnd);

				// 選択範囲を維持
				requestAnimationFrame(() => {
					textarea.selectionStart = firstLineStart;
					textarea.selectionEnd = firstLineStart + unindentedText.length;
				});
			}
		} else if (event.key === '`') {
			event.preventDefault();
			// バッククオート自動挿入
			code =
				value.substring(0, currentSelectionStart) + '``' + value.substring(currentSelectionEnd);
			requestAnimationFrame(() => {
				textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
			});
		} else if (event.key === '(') {
			// 丸括弧自動挿入
			event.preventDefault();
			code =
				value.substring(0, currentSelectionStart) + '()' + value.substring(currentSelectionEnd);
			requestAnimationFrame(() => {
				textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
			});
		} else if (event.key === ')') {
			// 閉じ丸括弧入力スキップ
			if (value[currentSelectionStart] === ')') {
				// カーソル直後にすでに閉じ括弧が挿入されている
				// スキップ
				event.preventDefault();
				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
				});
			}
		} else if (event.key === '[') {
			// 角括弧自動挿入
			event.preventDefault();
			code =
				value.substring(0, currentSelectionStart) + '[]' + value.substring(currentSelectionEnd);
			requestAnimationFrame(() => {
				textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
			});
		} else if (event.key === ']') {
			// 閉じ角括弧入力スキップ
			if (value[currentSelectionStart] === ']') {
				// カーソル直後にすでに閉じ括弧が挿入されている
				// スキップ
				event.preventDefault();
				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
				});
			}
		} else if (event.key === '{') {
			// 波括弧自動挿入
			event.preventDefault();
			code =
				value.substring(0, currentSelectionStart) + '{}' + value.substring(currentSelectionEnd);
			requestAnimationFrame(() => {
				textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
			});
		} else if (event.key === '}') {
			// 閉じ波括弧入力スキップ
			if (value[currentSelectionStart] === '}') {
				// カーソル直後にすでに閉じ括弧が挿入されている
				// スキップ
				event.preventDefault();
				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
				});
			}
		} else if (
			event.key === 'Backspace' &&
			currentSelectionStart === currentSelectionEnd &&
			currentSelectionStart > 0
		) {
			// Backspaceキー (閉じ括弧自動削除)
			// カーソルが単一で、かつ行頭でない場合
			const charToDelete = value[currentSelectionStart - 1]; // 削除しようとしている文字
			const charAfterCursor = value[currentSelectionStart]; // カーソル直後の文字

			let shouldPreventDefault = false;

			if (charToDelete === '(' && charAfterCursor === ')') {
				code =
					value.substring(0, currentSelectionStart - 1) +
					value.substring(currentSelectionStart + 1);
				shouldPreventDefault = true;
			} else if (charToDelete === '[' && charAfterCursor === ']') {
				code =
					value.substring(0, currentSelectionStart - 1) +
					value.substring(currentSelectionStart + 1);
				shouldPreventDefault = true;
			} else if (charToDelete === '{' && charAfterCursor === '}') {
				code =
					value.substring(0, currentSelectionStart - 1) +
					value.substring(currentSelectionStart + 1);
				shouldPreventDefault = true;
			} else if (charToDelete === '`' && charAfterCursor === '`') {
				code =
					value.substring(0, currentSelectionStart - 1) +
					value.substring(currentSelectionStart + 1);
				shouldPreventDefault = true;
			}

			if (shouldPreventDefault) {
				event.preventDefault();
				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart - 1;
				});
			}
		} else if (event.key === 'Enter') {
			// Enterキー (箇条書き自動継続)
			event.preventDefault();

			const [currentLine, lineStart, lineEnd] = getCurrentLine(value, currentSelectionStart);

			// 前の行が箇条書きであるかをチェック
			// - 、* 、+ 、数字. のいずれかに続くスペースを許容
			const listItemMatch = currentLine.match(/^\s*([-*+]|\d+\.)\s*(.*)$/);

			// コードブロックかどうかチェック
			const codeBlockMatch = currentLine.match(/^```.*```/);

			if (listItemMatch) {
				const listItemPrefix = listItemMatch[1]; // `-`, `1.`, `*` など
				const contentAfterPrefix = listItemMatch[2].trim(); // プレフィックスの後の内容

				if (contentAfterPrefix === '') {
					// 箇条書きプレフィックスのみの行でEnterを押した場合、プレフィックスを削除して改行
					code = value.substring(0, lineStart) + value.substring(currentSelectionStart);
					requestAnimationFrame(() => {
						textarea.selectionStart = textarea.selectionEnd = lineStart;
					});
				} else {
					// 箇条書きプレフィックスを継続
					let nextPrefix = listItemPrefix;
					if (listItemPrefix.match(/^\d+\.$/)) {
						// 番号付きリストの場合、番号をインクリメント
						const num = parseInt(listItemPrefix.slice(0, -1), 10);
						nextPrefix = `${num + 1}.`;
					}
					const indentSpace = listItemMatch[0].match(/^\s*/)![0]; // 行頭のインデントを保持

					code =
						value.substring(0, currentSelectionStart) +
						'\n' +
						indentSpace +
						nextPrefix +
						' ' +
						value.substring(currentSelectionEnd);
					requestAnimationFrame(() => {
						textarea.selectionStart = textarea.selectionEnd =
							currentSelectionStart + 1 + indentSpace.length + nextPrefix.length + 1;
					});
				}
			} else if (codeBlockMatch) {
				// コードブロックの場合は2行改行し、カーソルをブロック中央行へ
				code =
					value.substring(0, currentSelectionStart) + '\n\n' + value.substring(currentSelectionEnd);

				requestAnimationFrame(() => {
					textarea.selectionStart = textarea.selectionEnd = currentSelectionStart + 1;
				});
			} else {
				// インデントを維持
				const indentSpace = currentLine.match(/^\s*/)![0];
				if (event.shiftKey) {
					// 下に行を挿入
					code = value.substring(0, lineEnd) + '\n' + indentSpace + value.substring(lineEnd);
					requestAnimationFrame(() => {
						textarea.selectionStart = textarea.selectionEnd = lineEnd + 1;
					});
				} else {
					// 通常の改行
					code =
						value.substring(0, currentSelectionStart) +
						'\n' +
						indentSpace +
						value.substring(currentSelectionEnd);
					requestAnimationFrame(() => {
						textarea.selectionStart = textarea.selectionEnd =
							currentSelectionStart + 1 + indentSpace.length;
					});
				}
			}
		}

		onCursorMoved(currentSelectionStart, currentSelectionEnd);
	}

	function handleInput() {
		if (textarea) {
			currentSelectionStart = textarea.selectionStart;
			currentSelectionEnd = textarea.selectionEnd;
		}
	}

	// 行番号表示のための計算
	function getLineNumberStyle(index: number) {
		return `grid-row: ${index + 1}`;
	}
</script>

<div class="code-editor-container">
	<div class="line-numbers" bind:this={lineNumbersContainer}>
		{#each Array(lineNumbers) as _, i}
			<div class="line-number" style={getLineNumberStyle(i)}>{i + 1}</div>
		{/each}
	</div>
	<textarea
		bind:this={textarea}
		bind:value={code}
		on:keydown={handleKeydown}
		on:input={handleInput}
		on:scroll={handleTextareaScroll}
		wrap="off"
		spellcheck="false"
	></textarea>
</div>

<style>
	.code-editor-container {
		display: grid;
		grid-template-columns: auto 1fr; /* 行番号とテキストエリアの幅 */
		font-family: monospace;
		font-size: 14px;
		/* border: 1px solid #ccc;
		border-radius: 4px; */
		overflow: hidden;
		min-height: calc(100vh - 40px);
		margin-top: 40px;
	}

	.line-numbers {
		padding: 8px 10px;
		background-color: #555;
		color: #ccc;
		text-align: right;
		user-select: none;
		overflow-y: hidden; /* textareaのスクロールと同期させるため */
		display: flex;
		flex-direction: column;
		align-items: flex-end;
	}

	.line-number {
		height: 18px; /* 行の高さに合わせて調整 */
		line-height: 18px; /* 行の高さに合わせて調整 */
	}

	textarea {
		width: 100%;
		height: 100%;
		padding: 8px;
		border: none;
		outline: none;
		resize: none;
		box-sizing: border-box;
		font-family: inherit;
		font-size: inherit;
		line-height: 18px; /* 行の高さに合わせる */
		background-color: #333;
		color: #fff;
		overflow: auto; /* スクロール可能にする */
	}
</style>
