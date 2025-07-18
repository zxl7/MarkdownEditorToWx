<script setup lang="ts">
import { useAIConfig } from '@/components/ai/useAIConfig'
import { Button } from '@/components/ui/button'
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
import { Textarea } from '@/components/ui/textarea'
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from '@/components/ui/tooltip'
import { Check, Copy, Send, Settings, Trash } from 'lucide-vue-next'
import { nextTick, onMounted, ref, watch } from 'vue'

const props = defineProps<{ open: boolean }>()
const emit = defineEmits([`update:open`])

const dialogVisible = ref(props.open)
watch(() => props.open, val => (dialogVisible.value = val))
watch(dialogVisible, (val) => {
  emit(`update:open`, val)
  if (val)
    scrollToBottom(true)
})

const input = ref(``)
const inputHistory = ref<string[]>([])
const historyIndex = ref<number | null>(null)

const configVisible = ref(false)
const loading = ref(false)
const copiedIndex = ref<number | null>(null)
const memoryKey = `ai_memory_context`

interface ChatMessage {
  role: `user` | `assistant` | `system`
  content: string
  done?: boolean
}

const messages = ref<ChatMessage[]>([])
const { apiKey, endpoint, model, temperature, maxToken, type } = useAIConfig()

onMounted(async () => {
  const saved = localStorage.getItem(memoryKey)
  messages.value = saved ? JSON.parse(saved) : getDefaultMessages()
  await scrollToBottom(true)
})

function getDefaultMessages(): ChatMessage[] {
  return [{ role: `assistant`, content: `你好，我是 AI 助手，有什么可以帮你的？` }]
}

function handleConfigSaved() {
  configVisible.value = false
  scrollToBottom(true)
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === `Enter` && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
  else if (e.key === `ArrowUp`) {
    e.preventDefault()
    if (inputHistory.value.length === 0)
      return
    if (historyIndex.value === null) {
      historyIndex.value = inputHistory.value.length - 1
    }
    else if (historyIndex.value > 0) {
      historyIndex.value--
    }
    input.value = inputHistory.value[historyIndex.value] || ``
  }
  else if (e.key === `ArrowDown`) {
    e.preventDefault()
    if (historyIndex.value === null)
      return
    if (historyIndex.value < inputHistory.value.length - 1) {
      historyIndex.value++
      input.value = inputHistory.value[historyIndex.value] || ``
    }
    else {
      historyIndex.value = null
      input.value = ``
    }
  }
}

async function scrollToBottom(force = false) {
  await nextTick()
  const container = document.querySelector(`.chat-container`)
  if (container) {
    const isNearBottom = container.scrollTop + container.clientHeight >= container.scrollHeight - 50
    if (force || isNearBottom) {
      container.scrollTop = container.scrollHeight
      // 添加微小的延迟确保滚动完成
      await new Promise(resolve => setTimeout(resolve, 50))
    }
  }
}

async function copyToClipboard(text: string, index: number) {
  try {
    await navigator.clipboard.writeText(text)
    copiedIndex.value = index
    setTimeout(() => (copiedIndex.value = null), 1500)
    await scrollToBottom(true)
  }
  catch (err) {
    console.error(`复制失败:`, err)
  }
}

function resetMessages() {
  messages.value = getDefaultMessages()
  try {
    localStorage.setItem(memoryKey, JSON.stringify(messages.value))
  }
  catch (e) {
    console.error(`清空消息失败:`, e)
  }
  scrollToBottom(true)
}

async function sendMessage() {
  if (!input.value.trim() || loading.value)
    return

  inputHistory.value.push(input.value.trim())
  historyIndex.value = null

  loading.value = true
  const userInput = input.value.trim()
  messages.value.push({ role: `user`, content: userInput })
  input.value = ``
  const replyMessage: ChatMessage = { role: `assistant`, content: ``, done: false }
  messages.value.push(replyMessage)
  await scrollToBottom(true)

  const payload = {
    model: model.value,
    messages: [
      { role: `system`, content: `你是一个专业的 Markdown 编辑器助手，请用简洁中文回答。` },
      ...messages.value.slice(-12)
        .filter((msg, index, arr) =>
          !(index === arr.length - 1 && msg.role === `assistant` && !msg.done)
          && !(index === 0 && msg.role === `assistant`),
        )
        .slice(-10),
    ],
    temperature: temperature.value,
    max_tokens: maxToken.value,
    stream: true,
  }

  const headers: Record<string, string> = {
    'Content-Type': `application/json`,
  }

  if (apiKey.value && type.value !== `default`) {
    headers.Authorization = `Bearer ${apiKey.value}`
  }

  try {
    const url = new URL(endpoint.value)
    if (!url.pathname.endsWith(`/chat/completions`)) {
      url.pathname = url.pathname.replace(/\/?$/, `/chat/completions`)
    }

    const res = await window.fetch(url.toString(), {
      method: `POST`,
      headers: {
        'Authorization': `Bearer ${apiKey.value}`,
        'Content-Type': `application/json`,
      },
      body: JSON.stringify(payload),
    })

    if (!res.ok || !res.body)
      throw new Error(`响应错误：${res.status} ${res.statusText}`)

    const reader = res.body.getReader()
    const decoder = new TextDecoder(`utf-8`)
    let buffer = ``

    while (true) {
      const { value, done } = await reader.read()
      if (done) {
        const last = messages.value[messages.value.length - 1]
        if (last.role === `assistant`) {
          last.done = true
          await scrollToBottom(true)
        }
        break
      }

      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split(`\n`)
      buffer = lines.pop() || ``

      for (const line of lines) {
        if (line.trim() === ``)
          continue
        try {
          const eventData = line.replace(/^data: /, ``)
          if (eventData === `[DONE]`)
            continue
          const json = JSON.parse(eventData)
          const delta = json.choices?.[0]?.delta?.content
          if (delta) {
            const lastMessage = messages.value[messages.value.length - 1]
            lastMessage.content += delta
            await scrollToBottom()
          }
        }
        catch (e) {
          console.error(`解析失败:`, e)
        }
      }
    }
  }
  catch (e) {
    console.error(`请求失败:`, e)
    messages.value[messages.value.length - 1].content = `❌ 请求失败: ${(e as Error).message}`
    await scrollToBottom(true)
  }
  finally {
    try {
      localStorage.setItem(memoryKey, JSON.stringify(messages.value))
    }
    catch (e) {
      console.error(`保存失败:`, e)
    }
    loading.value = false
  }
}
</script>

<template>
  <Dialog v-model:open="dialogVisible">
    <DialogContent class="max-w-lg w-full rounded-xl">
      <DialogHeader class="space-y-1 flex flex-col items-start">
        <div class="space-x-1 flex items-center">
          <DialogTitle>AI 对话</DialogTitle>

          <TooltipProvider>
            <Tooltip>
              <TooltipTrigger as-child>
                <Button variant="ghost" size="icon" aria-label="配置" @click="configVisible = !configVisible">
                  <Settings class="h-4 w-4" />
                </Button>
              </TooltipTrigger>
              <TooltipContent>配置参数</TooltipContent>
            </Tooltip>

            <Tooltip>
              <TooltipTrigger as-child>
                <Button variant="ghost" size="icon" aria-label="清空对话" @click="resetMessages">
                  <Trash class="h-4 w-4" />
                </Button>
              </TooltipTrigger>
              <TooltipContent>清空对话内容</TooltipContent>
            </Tooltip>
          </TooltipProvider>
        </div>

        <p class="text-sm text-gray-500">
          使用 AI 助手帮助您编写和优化内容
        </p>
      </DialogHeader>

      <AIConfig v-if="configVisible" class="mb-4 w-full border rounded-md p-4" @saved="handleConfigSaved" />

      <div v-if="!configVisible" class="chat-container space-y-3 mb-4 max-h-60 overflow-y-auto pr-1">
        <div
          v-for="(msg, index) in messages" :key="index" class="relative flex"
          :class="msg.role === 'user' ? 'justify-end' : 'justify-start'"
        >
          <div
            class="max-w-xs rounded-xl px-4 py-2 text-sm"
            :class="msg.role === 'user' ? 'bg-black text-white' : 'bg-gray-100 text-gray-800'"
          >
            <div class="whitespace-pre-wrap">
              {{ msg.content }}
            </div>

            <div v-if="msg.role === 'assistant' && msg.done" class="mt-1 flex justify-start">
              <Button
                variant="ghost" size="icon" class="ml-0 h-5 w-5 p-1" aria-label="复制内容"
                @click="copyToClipboard(msg.content, index)"
              >
                <Check v-if="copiedIndex === index" class="h-3 w-3 text-green-600" />
                <Copy v-else class="h-3 w-3 text-gray-500" />
              </Button>
            </div>
          </div>
        </div>
      </div>

      <div v-if="!configVisible" class="relative mt-2">
        <div class="flex items-center border border-black/10 rounded-xl px-3 py-2 pr-12 shadow-sm">
          <Textarea
            v-model="input"
            placeholder="说些什么……(按 Enter 发送，Shift+Enter 换行)"
            rows="2"
            class="custom-scroll w-full resize-none overflow-y-auto border-none focus:border-none focus-visible:outline-none focus-visible:ring-0 focus:ring-0 focus-visible:ring-offset-0"
            @keydown="handleKeydown"
          />
          <Button
            :disabled="!input.trim() || loading" size="icon"
            class="absolute bottom-3 right-3 rounded-full bg-black p-2 text-white hover:bg-gray-800 disabled:opacity-40"
            aria-label="发送" @click="sendMessage"
          >
            <Send class="h-4 w-4" />
          </Button>
        </div>
      </div>
    </DialogContent>
  </Dialog>
</template>

<style scoped>
.custom-scroll::-webkit-scrollbar {
  @apply w-0 h-0;
}
.custom-scroll::-webkit-scrollbar-thumb {
  @apply bg-transparent;
}
.custom-scroll {
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE 10+ */
}
</style>
