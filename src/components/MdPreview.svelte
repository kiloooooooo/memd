<script lang="ts">
	import DOMPurify from 'dompurify';
	import { Marked } from 'marked';
	import { markedHighlight } from 'marked-highlight';
	import hljs from 'highlight.js';
	import 'highlight.js/styles/github-dark-dimmed.min.css';

	export let markdownCode = '';
	// MarkdownをHTMLに変換する関数（markedなどを使用）
	// $: renderedHtml = marked(myCode); // markedを使用する場合
	// $: renderedHtml = `<pre>${markdownCode}</pre>`;

	const marked = new Marked(
		markedHighlight({
			emptyLangClass: 'hljs',
			langPrefix: 'hljs language-',
			highlight(code, lang, info) {
				const language = hljs.getLanguage(lang) ? lang : 'plaintext';
				return hljs.highlight(code, { language }).value;
			}
		})
	);

	$: renderedHtml = DOMPurify.sanitize(marked.parse(markdownCode) as string);
</script>

<div class="preview-panel">
	{@html renderedHtml}
</div>

<style>
	.preview-panel {
		padding: 0 15px;
		margin-top: 40px;
		color: #fff;
		background-color: #333;
		min-height: calc(100vh - 40px);
		overflow: auto;
		font-family: sans-serif;
	}
</style>
