<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";
import { marked } from "marked";
import hljs from "highlight.js";

// 配置 marked
marked.setOptions({
  breaks: true,
  gfm: true,
});

// 配置代码高亮
marked.use({
  renderer: {
    code(args: any) {
      const code = args.text || args;
      const language = args.lang || "plaintext";
      let highlighted = code;
      if (hljs.getLanguage(language)) {
        highlighted = hljs.highlight(code, { language }).value;
      }
      return `<pre><code class="hljs language-${language}">${highlighted}</code></pre>`;
    },
  },
});

// 状态管理
const markdown = ref(`# 欢迎使用 Markdown 编辑器

这是一个类似 Typora 的 Markdown 编辑器。

## 功能特性

- **实时预览** - 边输入边看效果
- **语法高亮** - 代码块自动着色
- **简洁界面** - 专注写作体验
- **工具栏** - 快速格式化文本
- **快捷键** - 提高输入效率
- **字数统计** - 实时字数显示

## 代码示例

\`\`\`javascript
function hello(name) {
  console.log(\`Hello, \${name}!\`);
}
hello('Typora');
\`\`\`

## 列表

- 项目 1
- 项目 2
- 项目 3

## 引用

> 这是一个引用块

**祝你写作愉快！**
`);

const fileName = ref("untitled.md");
const isDarkMode = ref(false);
const editorRef = ref<HTMLTextAreaElement>();
const isFullScreen = ref(false);
const autoSaveInterval = ref<ReturnType<typeof setInterval> | null>(null);

// 计算属性
const wordCount = computed(() => {
  return markdown.value.replace(/\s+/g, " ").split(" ").filter(w => w.length > 0).length;
});

const charCount = computed(() => {
  return markdown.value.length;
});

// 方法
const insertText = (before: string, after: string = "") => {
  if (!editorRef.value) return;
  
  const textarea = editorRef.value;
  const start = textarea.selectionStart;
  const end = textarea.selectionEnd;
  const selectedText = markdown.value.substring(start, end);
  const beforeText = markdown.value.substring(0, start);
  const afterText = markdown.value.substring(end);
  
  markdown.value = beforeText + before + selectedText + after + afterText;
  
  setTimeout(() => {
    textarea.focus();
    textarea.selectionStart = start + before.length;
    textarea.selectionEnd = start + before.length + selectedText.length;
  }, 0);
};

const insertHeading = (level: number) => {
  const heading = "#".repeat(level) + " ";
  insertText(heading);
};

const insertBold = () => insertText("**", "**");
const insertItalic = () => insertText("*", "*");
const insertCode = () => insertText("`", "`");
const insertLink = () => insertText("[文字](", ")");
const insertImage = () => insertText("![描述](", ")");
const insertList = () => insertText("- ");
const insertOrderedList = () => insertText("1. ");
const insertBlockquote = () => insertText("> ");
const insertCodeBlock = () => insertText("\`\`\`javascript\n", "\n\`\`\`");
const insertTable = () => {
  const table = "\n| 列1 | 列2 | 列3 |\n|-----|-----|-----|\n| 数据 | 数据 | 数据 |\n";
  insertText(table);
};

const newFile = () => {
  if (markdown.value && !confirm("放弃当前文档？")) return;
  markdown.value = "";
  fileName.value = "untitled.md";
};

const downloadMarkdown = () => {
  const element = document.createElement("a");
  element.setAttribute("href", "data:text/plain;charset=utf-8," + encodeURIComponent(markdown.value));
  element.setAttribute("download", fileName.value);
  element.style.display = "none";
  document.body.appendChild(element);
  element.click();
  document.body.removeChild(element);
};

const downloadHTML = () => {
  const htmlContent = `<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${fileName.value}</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      max-width: 900px;
      margin: 0 auto;
      padding: 20px;
      line-height: 1.6;
      color: #333;
    }
    h1, h2, h3 { color: #222; }
    code { background: #f4f4f4; padding: 2px 6px; border-radius: 3px; }
    pre { background: #f4f4f4; padding: 12px; border-radius: 4px; overflow-x: auto; }
    blockquote { border-left: 4px solid #ddd; padding-left: 1em; color: #666; }
  </style>
</head>
<body>
  ${marked(markdown.value)}
</body>
</html>`;
  
  const element = document.createElement("a");
  element.setAttribute("href", "data:text/html;charset=utf-8," + encodeURIComponent(htmlContent));
  element.setAttribute("download", fileName.value.replace(".md", ".html"));
  element.style.display = "none";
  document.body.appendChild(element);
  element.click();
  document.body.removeChild(element);
};

const toggleFullScreen = () => {
  isFullScreen.value = !isFullScreen.value;
};

const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value;
};

// 快捷键处理
const handleKeyDown = (e: KeyboardEvent) => {
  if (e.ctrlKey || e.metaKey) {
    switch (e.key) {
      case "b":
        e.preventDefault();
        insertBold();
        break;
      case "i":
        e.preventDefault();
        insertItalic();
        break;
      case "k":
        e.preventDefault();
        insertLink();
        break;
      case "s":
        e.preventDefault();
        downloadMarkdown();
        break;
    }
  }
};

// 自动保存到本地存储
const autoSave = () => {
  localStorage.setItem("markdown_content", markdown.value);
  localStorage.setItem("markdown_filename", fileName.value);
};

// 生命周期
onMounted(() => {
  // 恢复草稿
  const savedContent = localStorage.getItem("markdown_content");
  if (savedContent) {
    markdown.value = savedContent;
  }
  const savedFileName = localStorage.getItem("markdown_filename");
  if (savedFileName) {
    fileName.value = savedFileName;
  }
  
  // 自动保存
  autoSaveInterval.value = setInterval(autoSave, 5000);
  
  // 添加快捷键监听
  if (editorRef.value) {
    editorRef.value.addEventListener("keydown", handleKeyDown);
  }
});

onUnmounted(() => {
  if (autoSaveInterval.value) {
    clearInterval(autoSaveInterval.value);
  }
  if (editorRef.value) {
    editorRef.value.removeEventListener("keydown", handleKeyDown);
  }
});
</script>

<template>
  <div class="editor-container" :class="{ 'dark-mode': isDarkMode, 'fullscreen': isFullScreen }">
    <!-- 主工具栏 -->
    <div class="main-toolbar">
      <div class="toolbar-group">
        <button class="toolbar-btn" @click="newFile" title="新建文件 (Ctrl+N)">
          <span>✕</span>
        </button>
        <button class="toolbar-btn" @click="downloadMarkdown" title="下载 Markdown (Ctrl+S)">
          <span>💾</span>
        </button>
        <button class="toolbar-btn" @click="downloadHTML" title="导出为 HTML">
          <span>📄</span>
        </button>
      </div>

      <div class="file-info">
        <input v-model="fileName" class="file-name-input" />
      </div>

      <div class="toolbar-group">
        <span class="stat-item">{{ charCount }} 字符</span>
        <span class="stat-item">{{ wordCount }} 字</span>
      </div>

      <div class="toolbar-group">
        <button class="toolbar-btn" @click="toggleFullScreen" :title="isFullScreen ? '退出全屏' : '全屏'">
          <span>{{ isFullScreen ? '⛶' : '⛶' }}</span>
        </button>
        <button class="toolbar-btn" @click="toggleDarkMode" :title="isDarkMode ? '亮色模式' : '暗色模式'">
          <span>{{ isDarkMode ? '☀️' : '🌙' }}</span>
        </button>
      </div>
    </div>

    <!-- 格式化工具栏 -->
    <div class="format-toolbar">
      <div class="toolbar-group">
        <button class="format-btn" @click="insertHeading(1)" title="H1 标题">H1</button>
        <button class="format-btn" @click="insertHeading(2)" title="H2 标题">H2</button>
        <button class="format-btn" @click="insertHeading(3)" title="H3 标题">H3</button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button class="format-btn" @click="insertBold" title="加粗 (Ctrl+B)"><strong>B</strong></button>
        <button class="format-btn" @click="insertItalic" title="斜体 (Ctrl+I)"><em>I</em></button>
        <button class="format-btn" @click="insertCode" title="代码">Code</button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button class="format-btn" @click="insertLink" title="链接 (Ctrl+K)">🔗</button>
        <button class="format-btn" @click="insertImage" title="图片">🖼️</button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button class="format-btn" @click="insertList" title="无序列表">•</button>
        <button class="format-btn" @click="insertOrderedList" title="有序列表">1.</button>
        <button class="format-btn" @click="insertBlockquote" title="引用">"</button>
      </div>

      <div class="toolbar-divider"></div>

      <div class="toolbar-group">
        <button class="format-btn" @click="insertCodeBlock" title="代码块">{ }</button>
        <button class="format-btn" @click="insertTable" title="表格">⊞</button>
      </div>
    </div>

    <!-- 编辑区域 -->
    <div class="editor-wrapper">
      <div class="editor-pane">
        <textarea
          ref="editorRef"
          v-model="markdown"
          class="editor"
          placeholder="在这里写入 Markdown..."
          @keydown="handleKeyDown"
        ></textarea>
      </div>

      <div class="preview-pane">
        <div class="preview-content" v-html="marked(markdown)"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.editor-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f5f5f5;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    "Helvetica Neue", Arial, sans-serif;
  transition: background-color 0.3s, color 0.3s;
}

.editor-container.dark-mode {
  background-color: #1e1e1e;
  color: #e0e0e0;
}

.editor-container.dark-mode .main-toolbar {
  background-color: #2d2d2d;
  border-bottom-color: #444;
}

.editor-container.dark-mode .format-toolbar {
  background-color: #2d2d2d;
  border-bottom-color: #444;
}

.editor-container.dark-mode .editor {
  background-color: #1e1e1e;
  color: #e0e0e0;
}

.editor-container.dark-mode .preview-pane {
  background-color: #1a1a1a;
}

.editor-container.dark-mode .preview-content {
  background-color: #2d2d2d;
  color: #e0e0e0;
}

.editor-container.fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 9999;
}

/* 主工具栏 */
.main-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 50px;
  padding: 0 20px;
  background-color: #fff;
  border-bottom: 1px solid #e0e0e0;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  flex-wrap: wrap;
  gap: 15px;
}

.file-info {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 200px;
}

.file-name-input {
  border: none;
  background: none;
  text-align: center;
  font-size: 14px;
  font-weight: 500;
  color: #333;
  outline: none;
  padding: 4px 8px;
  border-radius: 4px;
  min-width: 150px;
}

.file-name-input:hover {
  background-color: #f5f5f5;
}

.file-name-input:focus {
  background-color: #e8f4f8;
  color: #0066cc;
}

.editor-container.dark-mode .file-name-input {
  color: #e0e0e0;
}

.editor-container.dark-mode .file-name-input:hover {
  background-color: #444;
}

.editor-container.dark-mode .file-name-input:focus {
  background-color: #333;
  color: #66b3ff;
}

.toolbar-group {
  display: flex;
  gap: 8px;
  align-items: center;
}

.toolbar-btn {
  background: none;
  border: none;
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
  color: #666;
  font-size: 16px;
  transition: all 0.2s;
  white-space: nowrap;
}

.toolbar-btn:hover {
  background-color: #f0f0f0;
  color: #333;
}

.editor-container.dark-mode .toolbar-btn {
  color: #999;
}

.editor-container.dark-mode .toolbar-btn:hover {
  background-color: #444;
  color: #fff;
}

.stat-item {
  font-size: 12px;
  color: #999;
  white-space: nowrap;
}

.editor-container.dark-mode .stat-item {
  color: #777;
}

/* 格式化工具栏 */
.format-toolbar {
  display: flex;
  gap: 0;
  padding: 8px 20px;
  background-color: #fff;
  border-bottom: 1px solid #e0e0e0;
  align-items: center;
  flex-wrap: wrap;
}

.toolbar-divider {
  width: 1px;
  height: 24px;
  background-color: #ddd;
  margin: 0 8px;
}

.editor-container.dark-mode .toolbar-divider {
  background-color: #444;
}

.format-btn {
  background: none;
  border: none;
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
  color: #666;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.2s;
  min-width: 40px;
  white-space: nowrap;
}

.format-btn:hover {
  background-color: #f0f0f0;
  color: #333;
}

.format-btn:active {
  background-color: #e0e0e0;
  transform: scale(0.95);
}

.editor-container.dark-mode .format-btn {
  color: #999;
}

.editor-container.dark-mode .format-btn:hover {
  background-color: #444;
  color: #fff;
}

/* 编辑区域 */
.editor-wrapper {
  display: flex;
  flex: 1;
  overflow: hidden;
  gap: 0;
}

.editor-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #e0e0e0;
  background-color: #fff;
}

.editor-container.dark-mode .editor-pane {
  border-right-color: #444;
  background-color: #1e1e1e;
}

.editor {
  flex: 1;
  padding: 20px;
  border: none;
  outline: none;
  resize: none;
  font-family: "Monaco", "Menlo", "Ubuntu Mono", monospace;
  font-size: 14px;
  line-height: 1.6;
  color: #333;
  background-color: #fff;
  tab-size: 2;
}

.preview-pane {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background-color: #fafafa;
}

.preview-content {
  max-width: 900px;
  margin: 0 auto;
  background-color: #fff;
  padding: 20px;
  border-radius: 4px;
  line-height: 1.8;
  color: #333;
}

.preview-content :deep(h1) {
  font-size: 2em;
  margin: 0.67em 0;
  font-weight: bold;
  border-bottom: 2px solid #ddd;
  padding-bottom: 0.3em;
}

.preview-content :deep(h2) {
  font-size: 1.5em;
  margin: 0.75em 0;
  font-weight: bold;
  border-bottom: 1px solid #ddd;
  padding-bottom: 0.3em;
}

.preview-content :deep(h3) {
  font-size: 1.2em;
  margin: 0.83em 0;
  font-weight: bold;
}

.preview-content :deep(h4),
.preview-content :deep(h5),
.preview-content :deep(h6) {
  margin: 1em 0;
  font-weight: bold;
}

.preview-content :deep(p) {
  margin: 1em 0;
}

.preview-content :deep(code) {
  background-color: #f4f4f4;
  padding: 2px 6px;
  border-radius: 3px;
  font-family: "Monaco", "Menlo", monospace;
  font-size: 0.9em;
  color: #c7254e;
}

.preview-content :deep(pre) {
  background-color: #f4f4f4;
  padding: 12px;
  border-radius: 4px;
  overflow-x: auto;
  margin: 1em 0;
}

.preview-content :deep(pre code) {
  background-color: transparent;
  padding: 0;
  color: #333;
}

.preview-content :deep(blockquote) {
  border-left: 4px solid #ddd;
  margin: 1em 0;
  padding-left: 1em;
  color: #666;
}

.preview-content :deep(ul),
.preview-content :deep(ol) {
  margin: 1em 0;
  padding-left: 2em;
}

.preview-content :deep(li) {
  margin: 0.5em 0;
}

.preview-content :deep(a) {
  color: #0066cc;
  text-decoration: none;
}

.preview-content :deep(a:hover) {
  text-decoration: underline;
}

.preview-content :deep(strong) {
  font-weight: bold;
}

.preview-content :deep(em) {
  font-style: italic;
}

.preview-content :deep(hr) {
  border: none;
  border-top: 2px solid #ddd;
  margin: 2em 0;
}

.preview-content :deep(table) {
  border-collapse: collapse;
  width: 100%;
  margin: 1em 0;
}

.preview-content :deep(th),
.preview-content :deep(td) {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.preview-content :deep(th) {
  background-color: #f4f4f4;
  font-weight: bold;
}

/* 暗色模式 - 预览区域 */
.editor-container.dark-mode .preview-content :deep(h1),
.editor-container.dark-mode .preview-content :deep(h2),
.editor-container.dark-mode .preview-content :deep(h3),
.editor-container.dark-mode .preview-content :deep(h4),
.editor-container.dark-mode .preview-content :deep(h5),
.editor-container.dark-mode .preview-content :deep(h6) {
  color: #e0e0e0;
  border-bottom-color: #444;
}

.editor-container.dark-mode .preview-content :deep(code) {
  background-color: #333;
  color: #ff7b9c;
}

.editor-container.dark-mode .preview-content :deep(pre) {
  background-color: #333;
}

.editor-container.dark-mode .preview-content :deep(pre code) {
  color: #e0e0e0;
}

.editor-container.dark-mode .preview-content :deep(blockquote) {
  border-left-color: #444;
  color: #999;
}

.editor-container.dark-mode .preview-content :deep(a) {
  color: #66b3ff;
}

.editor-container.dark-mode .preview-content :deep(table) {
  border-color: #444;
}

.editor-container.dark-mode .preview-content :deep(th) {
  background-color: #333;
  color: #e0e0e0;
  border-color: #444;
}

.editor-container.dark-mode .preview-content :deep(td) {
  border-color: #444;
}
</style>
<style>
:root {
  font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
  font-size: 16px;
  line-height: 24px;
  font-weight: 400;

  color: #0f0f0f;
  background-color: #f6f6f6;

  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  -webkit-text-size-adjust: 100%;
}

.container {
  margin: 0;
  padding-top: 10vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  text-align: center;
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: 0.75s;
}

.logo.tauri:hover {
  filter: drop-shadow(0 0 2em #24c8db);
}

.row {
  display: flex;
  justify-content: center;
}

a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}

a:hover {
  color: #535bf2;
}

h1 {
  text-align: center;
}

input,
button {
  border-radius: 8px;
  border: 1px solid transparent;
  padding: 0.6em 1.2em;
  font-size: 1em;
  font-weight: 500;
  font-family: inherit;
  color: #0f0f0f;
  background-color: #ffffff;
  transition: border-color 0.25s;
  box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
}

button {
  cursor: pointer;
}

button:hover {
  border-color: #396cd8;
}
button:active {
  border-color: #396cd8;
  background-color: #e8e8e8;
}

input,
button {
  outline: none;
}

#greet-input {
  margin-right: 5px;
}

@media (prefers-color-scheme: dark) {
  :root {
    color: #f6f6f6;
    background-color: #2f2f2f;
  }

  a:hover {
    color: #24c8db;
  }

  input,
  button {
    color: #ffffff;
    background-color: #0f0f0f98;
  }
  button:active {
    background-color: #0f0f0f69;
  }
}

</style>