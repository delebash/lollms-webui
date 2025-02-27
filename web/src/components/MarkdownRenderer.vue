<template>
  <div class="break-all container w-full">
    <div ref="mdRender" class="markdown-content">
      <div v-for="(item, index) in markdownItems" :key="index">
        <code-block
          v-if="item.type === 'code'"
          :host="host"
          :language="item.language"
          :code="item.code"
          :discussion_id="discussion_id"
          :message_id="message_id"
          :client_id="client_id"
          @update-code="updateCode(index, $event)"
        ></code-block>
        <div v-else v-html="item.html"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { nextTick, ref, onMounted, watch } from 'vue';
import feather from 'feather-icons';
import MarkdownIt from 'markdown-it';
import emoji from 'markdown-it-emoji';
import anchor from 'markdown-it-anchor';
import MarkdownItMultimdTable from 'markdown-it-multimd-table';
import implicitFigures from 'markdown-it-implicit-figures';
import 'highlight.js/styles/tomorrow-night-blue.css';
import 'highlight.js/styles/tokyo-night-dark.css';
import attrs from 'markdown-it-attrs';
import CodeBlock from './CodeBlock.vue';
import hljs from 'highlight.js';
import mathjax from 'markdown-it-mathjax';

function escapeHtml(unsafe) {
  return unsafe
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

export default {
  name: 'MarkdownRenderer',
  props: {
    host: {
      type: String,
      required: false,
      default: "http://localhost:9600",
    },
    client_id: {
      type: String,
      required: true,
    },
    markdownText: {
      type: String,
      required: true,
    },
    discussion_id: {
      type: [String, Number],
      default: "0",
      required: false,
    },
    message_id: {
      value: "0",
      type: [String, Number],
      required: false,
    },
  },
  components: {
    CodeBlock,
  },
  setup(props) {
    const md = new MarkdownIt({
      html: true,
      breaks: true, // Enable single line breaks
      highlight: (code, language) => {
        const validLanguage = language && hljs.getLanguage(language) ? language : 'plaintext';
        return hljs.highlight(validLanguage, code).value;
      },
      renderInline: false,
    })
      .use(emoji)
      .use(anchor)
      .use(implicitFigures, {
        figcaption: true,
      })
      .use(attrs)
      .use(MarkdownItMultimdTable, {
        enableRowspan: true,
        enableColspan: true,
        enableGridTables: true,
        enableGridTablesExtra: true,
        enableTableIndentation: true,
        tableCellPadding: ' ',
        tableCellJoiner: '|',
        multilineCellStartMarker: '|>',
        multilineCellEndMarker: '<|',
        multilineCellPadding: ' ',
        multilineCellJoiner: '\n',
      })
      .use(mathjax); // Ensure mathjax is used for inline LaTeX

    const markdownItems = ref([]);
    const updateMarkdown = () => {
      if (props.markdownText) {
        let tokens = md.parse(props.markdownText, {});
        let cumulated = [];
        markdownItems.value = [];
        for (let i = 0; i < tokens.length; i++) {
          if (tokens[i].type !== 'fence') {
            cumulated.push(tokens[i]);
          } else {
            if (cumulated.length > 0) {
              markdownItems.value.push({
                type: 'html',
                html: md.renderer.render(cumulated, md.options, {}),
              });
              cumulated = [];
            }
            markdownItems.value.push({
              type: 'code',
              language: escapeHtml(tokens[i].info),
              code: tokens[i].content,
            });
          }
        }
        if (cumulated.length > 0) {
          markdownItems.value.push({
            type: 'html',
            html: md.renderer.render(cumulated, md.options, {}),
          });
          cumulated = [];
        }
      } else {
        markdownItems.value = [];
      }
      nextTick(() => {
        feather.replace();
        if (window.MathJax) {
          window.MathJax.typesetPromise(); // Ensure MathJax typesets after rendering
        }
      });
    };

    const updateCode = (index, newCode) => {
      markdownItems.value[index].code = newCode;
    };

    watch(() => props.markdownText, updateMarkdown);
    onMounted(() => {
      updateMarkdown();
    });
    return { markdownItems, updateCode };
  },
};
</script>

<style>
/* Your existing styles */
.katex-display {
  display: inline-block;
  margin: 0;
}

.katex {
  display: inline-block;
  white-space: nowrap;
}
.inline-latex {
  display: inline !important;
}
</style>
