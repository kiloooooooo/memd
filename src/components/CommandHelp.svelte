<script lang="ts">
	export let showHelp: boolean; // 親からバインドされる表示フラグ

	// Escキーでヘルプを閉じる
	function handleKeyDown(event: KeyboardEvent) {
		if (event.key === 'Escape') {
			event.preventDefault();
			showHelp = false; // 親のバインド変数も更新される
		}
	}

	// スロットを使って、親から渡されたコンテンツ（ボタンなど）を受け取れるようにする
	// 今回はショートカットで直接制御するため、スロットは必須ではないが、柔軟性のため保持
</script>

{#if showHelp}
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="overlay command-help-overlay" on:keydown={handleKeyDown}>
		<h4>コマンドヘルプ</h4>
		<table>
			<thead>
				<tr>
					<th></th>
					<th></th>
				</tr>
			</thead>
			<tbody>
				<tr>
					<td>編集 / プレビュー中:</td>
					<td></td>
				</tr>
				<tr>
					<td>Ctrl + S</td>
					<td>ファイルを保存する</td>
				</tr>
				<tr>
					<td>Ctrl + P</td>
					<td>プレビュー表示を切り替える</td>
				</tr>
				<tr>
					<td>Ctrl + O</td>
					<td>
						ファイル一覧を表示/非表示<br />
						(フォルダ選択)
					</td>
				</tr>
				<tr>
					<td>Ctrl + /</td>
					<td>このヘルプを表示/非表示</td>
				</tr>
				<tr>
					<td>ファイル一覧表示中:</td>
					<td></td>
				</tr>
				<tr>
					<td>↑ ↓ または J K</td>
					<td>ファイルを選択</td>
				</tr>
				<tr>
					<td>Enter</td>
					<td>選択したファイルを開く</td>
				</tr>
				<tr>
					<td>Esc</td>
					<td>ファイル一覧を閉じる</td>
				</tr>
			</tbody>
		</table>
	</div>
{/if}

<style>
	.overlay {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		background-color: #222;
		border: 1px solid #555;
		padding: 20px 0;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
		z-index: 100;
		text-align: start;
	}

	.command-help-overlay {
		width: 75%;
		max-width: 600px;
	}

	.command-help-overlay h4 {
		margin-top: 0;
		margin-left: 20px;
		color: #fff;
	}

	.command-help-overlay table {
		width: 100%;
		border-collapse: collapse;
		margin-top: 20px;
		margin-left: 20px;
		color: #fff;
	}

	.command-help-overlay th,
	.command-help-overlay td {
		/* border: 1px solid #ddd; */
		/* padding: 10px; */
		padding: 8px 0;
		text-align: left;
	}

	.command-help-overlay th {
		/* background-color: #444; */
		font-weight: bold;
	}

	.command-help-overlay td:first-child {
		font-family: monospace;
		font-weight: bold;
		white-space: nowrap;
	}
</style>
