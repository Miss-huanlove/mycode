<template>
  <div class="chat-wrapper">
    <div style="margin-bottom: 10px;">大帅比欢哥</div>
    <div class="chat-container">
      
      <div class="message-box" ref="msgBoxRef">
        <div v-for="(item, idx) in messages" :key="idx" class="message-item">
          <div class="avatar">{{ item.role === 'user' ? '我' : '帅欢' }}</div>
          <div class="bubble">
            <div
              v-if="item.role === 'ai'"
              v-html="renderMarkdown(item.content)"
              class="markdown-body"
            />
            <div v-else class="plain-text">{{ item.content }}</div>
          </div>
        </div>
        <div v-if="isLoading" class="loading">欢哥 思考中…</div>
      </div>

      <div class="input-bar">
        <input
          v-model="inputText"
          @keyup.enter="sendMessage"
          placeholder="输入问题..."
          :disabled="isLoading"
        />
        <button @click="sendMessage" :disabled="isLoading">
          {{ isLoading ? '发送中...' : '发送' }}
        </button>
        <button @click="stopGenerate" v-if="isLoading">停止</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, nextTick, onUnmounted } from 'vue'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'

// 初始化 Markdown + 代码高亮
const md = new MarkdownIt({
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(str, { language: lang }).value
      } catch (__) {}
    }
    return ''
  },
  html: true,
  linkify: true,
  breaks: true
})

// 状态
const inputText = ref('')
const messages = reactive([{ role: 'ai', content: '你好，我是 你的大帅比 欢哥，有什么想问我的～' }])
const msgBoxRef = ref(null)
const isLoading = ref(false)
let abortController = null

// ===================== 填写你的 KEY =====================
const API_KEY = import.meta.env.VITE_DEEPSEEK_API_KEY
const API_URL = 'https://api.deepseek.com/chat/completions'
// ======================================================

// Markdown 渲染
function renderMarkdown(content) {
  return md.render(content)
}

// 发送消息
async function sendMessage() {
  const text = inputText.value.trim()
  if (!text || isLoading.value) return

  messages.push({ role: 'user', content: text })
  inputText.value = ''
  isLoading.value = true
  await nextTick()
  scrollToBottom()

  const aiMsg = { role: 'ai', content: '' }
  messages.push(aiMsg)
  await nextTick()
  scrollToBottom()

  abortController = new AbortController()

  try {
    const res = await fetch(API_URL, {
      method: 'POST',
      signal: abortController.signal,
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${API_KEY}`
      },
      body: JSON.stringify({
        model: 'deepseek-chat',
        stream: true,
        messages: [
          { role: 'system', content: '你是专业AI助手，支持Markdown、代码高亮。' },
          { role: 'user', content: text }
        ]
      })
    })

    const reader = res.body.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const chunk = decoder.decode(value)
      const lines = chunk.split('\n').filter(i => i.trim())

      for (const line of lines) {
        if (!line.startsWith('data: ')) continue
        const data = line.replace('data: ', '')
        if (data === '[DONE]') continue

        try {
          const json = JSON.parse(data)
          const content = json.choices?.[0]?.delta?.content || ''
          if (content) {
            aiMsg.content += content
            nextTick(() => scrollToBottom())
          }
        } catch (e) {}
      }
    }
  } catch (err) {
    if (err.name !== 'AbortError') {
      aiMsg.content = '出错：' + err.message
    }
  } finally {
    isLoading.value = false
  }
}

// 停止生成
function stopGenerate() {
  if (abortController) abortController.abort()
  isLoading.value = false
}

// 滚动到底部
function scrollToBottom() {
  const el = msgBoxRef.value
  if (el) el.scrollTop = el.scrollHeight
}

onUnmounted(stopGenerate)
</script>

<style>
@import 'highlight.js/styles/github-dark.css';

.markdown-body {
  line-height: 1.6;
  word-break: break-word;
}
.markdown-body pre {
  background: #1e1e1e;
  color: #fff;
  padding: 12px;
  border-radius: 8px;
  overflow-x: auto;
  margin: 8px 0;
}
.markdown-body code {
  background: #f5f5f5;
  padding: 2px 6px;
  border-radius: 4px;
}
.markdown-body h1,
.markdown-body h2,
.markdown-body h3 {
  margin: 10px 0;
  font-weight: bold;
}
.markdown-body ul,
.markdown-body ol {
  padding-left: 20px;
  margin: 8px 0;
}
</style>

<style scoped>
.chat-wrapper {
  width: 100%;
  height: 100vh;
  background: #f7f8fa;
  display: flex;
  align-items: center;
  flex-direction: column;
  justify-content: center;
}
.chat-container {
  width: 90%;
  max-width: 800px;
  height: 90vh;
  background: #fff;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
}
.message-box {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.message-item {
  display: flex;
  gap: 10px;
  max-width: 80%;
}
.message-item:nth-child(odd) {
  align-self: flex-end;
  flex-direction: row-reverse;
}
.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #42b983;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  font-size: 12px;
}
.bubble {
  background: #f5f5f5;
  padding: 12px 16px;
  border-radius: 12px;
  min-width: 40px;
}
.message-item:nth-child(odd) .bubble {
  background: #e6f7ef;
}
.input-bar {
  display: flex;
  padding: 12px;
  gap: 8px;
  border-top: 1px solid #eee;
}
input {
  flex: 1;
  padding: 12px 16px;
  border: 1px solid #ddd;
  border-radius: 24px;
  outline: none;
}
button {
  padding: 0 18px;
  background: #42b983;
  color: #fff;
  border: none;
  border-radius: 24px;
  cursor: pointer;
}
button:nth-child(3) {
  background: #ff4d4f;
}
.loading {
  color: #999;
  font-size: 14px;
}
</style>