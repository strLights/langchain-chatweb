<script setup lang='ts'>
import type { Ref } from 'vue'
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { useRoute } from 'vue-router'
import { storeToRefs } from 'pinia'
import { NAutoComplete, NButton, NImage, NInput, NModal, NSpin, useDialog, useMessage } from 'naive-ui'
import html2canvas from 'html2canvas'
import { Message } from './components'
import { useScroll } from './hooks/useScroll'
import { useChat } from './hooks/useChat'
import { useUsingContext } from './hooks/useUsingContext'
import HeaderComponent from './components/Header/index.vue'
import { SvgIcon } from '@/components/common'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { useChatStore, usePromptStore, useUserStore } from '@/store'
import { t } from '@/locales'
import { chatOpenAI, chatfileOpenai } from '@/api/chat'
import { changeModels, llmModels } from '@/api/llm/models'
import { modelsStore } from '@/store/modules/models/models-setting'
import type { ModelInfo } from '@/store/modules/user/helper'
import spark from '@/assets/spark-icon.ico'
import qianwen from '@/assets/qwen.png'
import chatgpt from '@/assets/chatgpt.png'
import chatglm from '@/assets/chatglm.png'

let controller = new AbortController()

// const openLongReply = import.meta.env.VITE_GLOB_OPEN_LONG_REPLY === 'true'

const store = modelsStore()
const userStore = useUserStore()
const route = useRoute()
const dialog = useDialog()
const ms = useMessage()
const chatStore = useChatStore()

const { isMobile } = useBasicLayout()
const { addChat, updateChat, updateChatSome, getChatByUuidAndIndex } = useChat()
const { scrollRef, scrollToBottom, scrollToBottomIfAtBottom } = useScroll()
const { usingContext, toggleUsingContext } = useUsingContext()

const { uuid } = route.params as { uuid: string }

const dataSources = computed(() => chatStore.getChatByUuid(+uuid))
const conversationList = computed(() => dataSources.value.filter(item => (!item.inversion && !!item.conversationOptions)))

// 页面加载spin
const show = ref(false)

const prompt = ref<string>('')
const loading = ref<boolean>(false)
const inputRef = ref<Ref | null>(null)

// 添加PromptStore
const promptStore = usePromptStore()

// 历史记录相关
const history: any = ref([])
// 使用storeToRefs，保证store修改后，联想部分能够重新渲染
const { promptList: promptTemplate } = storeToRefs<any>(promptStore)

// 是否开启知识库问答
const active = ref<boolean>(false)

// 语言模型列表
const loadModel = ref([])
const LlmData = ref([
  {
    label: 'chatglm2-6b',
    value: 'chatglm2-6b',
    icon: chatglm,
    status: false,
  },
  {
    label: '通义千问',
    value: 'Qwen-7B-Chat',
    icon: qianwen,
    status: false,
  },
  {
    label: '讯飞星火',
    value: 'xinghuo-api',
    icon: spark,
    status: false,
  },
  {
    label: 'chatgpt3.5',
    value: 'gpt-3.5-turbo',
    icon: chatgpt,
    status: false,
  },
])
const pickLlm = ref('')
const resultData = ref([])

function updateUserInfo(options: Partial<ModelInfo>) {
  userStore.updateUserInfo(options)
}

// function handleReset() {
//   userStore.resetUserInfo()
//   ms.success(t('common.success'))
//   window.location.reload()
// }
// 切换模型
const changeModel = (val: any) => {
  show.value = true
  const data = {
    model_name: pickLlm.value,
    new_model_name: val.value,
    controller_address: 'http://120.133.83.144:20001',
  }
  if (!val.status) {
    changeModels(data).then((res) => {
      const { code, msg } = res?.data
      pickLlm.value = val
      localStorage.setItem('pickedModel', pickLlm.value)
      if (code === 200) {
        ms.success(msg)
        updateUserInfo({ name: val.value, logo: val.icon })
      }
      else {
        ms.error(msg)
      }
      show.value = false
    }).catch((err) => {
      show.value = false
      ms.error(err)
    })
  }
  else {
    pickLlm.value = val.value
    localStorage.setItem('pickedModel', pickLlm.value)
    updateUserInfo({ name: val.value, logo: val.icon })
    ms.success(`模型已切换成${val.label}`)
    show.value = false
  }
}
// 获取加载的模型和api
const getLoadModel = () => {
  const param = {
    controller_address: 'http://120.133.83.144:20001',
    placeholder: 'string',
  }
  llmModels(param).then((res) => {
    const { code, msg, data } = res.data
    if (code === 200) {
      loadModel.value = data
      LlmData.value.forEach((element) => {
        data.forEach((ele: any) => {
          if (element.value === ele)
            element.status = true
          if (!ele.includes('api'))
            pickLlm.value = ele
        })
      })
    }
    else {
      ms.error(msg)
    }
  }).catch((err) => {
    ms.error(err)
  })
}
// 未知原因刷新页面，loading 状态不会重置，手动重置
dataSources.value.forEach((item, index) => {
  if (item.loading)
    updateChatSome(+uuid, index, { loading: false })
})
// 提交文本
function handleSubmit() {
  if (store.Chatgpt)
    onConversation2()
  if (store.chatglm)
    onConversation()
}
// 对话模型
async function onConversation() {
  const message = prompt.value
  if (usingContext.value) {
    for (let i = 0; i < dataSources.value.length; i = i + 2) { // 获取历史记录
      if (i % 2 === 1) {
        history.value.push({
          role: 'assistant',
          content: dataSources.value[i].text,
        })
      }
      else {
        history.value.push({
          role: 'user',
          content: dataSources.value[i].text,
        })
      }
    }
    // history.value.push({ 'Human': dataSources.value[i].text, 'Assistant': dataSources.value[i + 1].text.split('\n\n数据来源：\n\n')[0] })
  }
  else { history.value.length = 0 }
  if (!message || message.trim() === '')
    return

  controller = new AbortController()

  addChat(
    +uuid,
    {
      dateTime: new Date().toLocaleString(),
      text: message,
      knowledge: false,
      inversion: true,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: null },
    },
  )
  scrollToBottom()

  loading.value = true
  prompt.value = ''

  let options: Chat.ConversationRequest = {}
  const lastContext = conversationList.value[conversationList.value.length - 1]?.conversationOptions

  if (lastContext && usingContext.value)
    options = { ...lastContext }

  addChat(
    +uuid,
    {
      dateTime: new Date().toLocaleString(),
      text: '',
      loading: true,
      knowledge: false,
      inversion: false,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
    },
  )
  scrollToBottom()

  try {
    const lastText = ''
    const data = {
      query: message,
      history: history.value,
      model_name: pickLlm.value,
      stream: true,
      temperature: localStorage.getItem('temperature'),
    }
    const baseData = {
      query: message,
      knowledge_base_name: localStorage.getItem('knowledgeBase'),
      top_k: localStorage.getItem('matchItems'),
      score_threshold: localStorage.getItem('threshold'),
      history: history.value,
      model_name: pickLlm.value,
      stream: true,
      temperature: localStorage.getItem('temperature'),
      local_doc_url: false,
    }
    const fetchChatAPIOnce = async () => {
      let result = ''
      if (localStorage.getItem('chatMode') === 'knowledge') {
        const response = await fetch('api/chat/knowledge_base_chat', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(baseData),
        })
        if (!response.body)
          return
        const reader = response.body.pipeThrough(new TextDecoderStream()).getReader()
        while (true) {
          const { value, done } = await reader.read()
          if (done)
            break
          // value = value?.replace('undefined', '')
          console.log('knowledge data -', JSON.parse(value))
          if (JSON.parse(value).answer)
            result += JSON.parse(value).answer
          resultData.value = JSON.parse(value).docs
          // output.value += value?.replace('undefined', '')
          try {
            updateChat(
              +uuid,
              dataSources.value.length - 1,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (result ?? ''),
                source: resultData.value,
                knowledge: true,
                inversion: false,
                error: false,
                loading: false,
                conversationOptions: null,
                requestOptions: { prompt: message, options: { ...options } },
              },
            )
          }
          catch (error: any) {}
        }
      }
      // else if (localStorage.getItem('chatMode') === 'LLM' || null) {
      else {
        const resData = {
          text: '',
        }
        const response = await fetch('api/chat/chat', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(data),
        })
        if (!response.body)
          return
        const reader = response.body.pipeThrough(new TextDecoderStream()).getReader()
        while (true) {
          const { value, done } = await reader.read()
          if (done)
            break
          // value = value?.replace('undefined', '')
          console.log('received data -', value)
          resData.text += value
          // resdata = value
          // resData = JSON.parse(resData)
          // output.value += value?.replace('undefined', '')
          try {
            updateChat(
              +uuid,
              dataSources.value.length - 1,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (resData.text ?? ''),
                knowledge: false,
                inversion: false,
                error: false,
                loading: false,
                conversationOptions: null,
                requestOptions: { prompt: message, options: { ...options } },
              },
            )
          }
          catch (error: any) {}
        }
      }

      // const res = await chat(data)
      // const result = active.value ? `${res.data.response.text}\n\n数据来源：\n\n[${res.data.url.split('/static/')[1]}](http://127.0.0.1:3000${res.data.url})` : res.data.text
      // updateChat(
      //   +uuid,
      //   dataSources.value.length - 1,
      //   {
      //     dateTime: new Date().toLocaleString(),
      //     text: lastText + (resdata ?? ''),
      //     // text: result ?? '',
      //     inversion: false,
      //     error: false,
      //     loading: false,
      //     conversationOptions: null,
      //     requestOptions: { prompt: message, options: { ...options } },
      //   },
      // )
      scrollToBottomIfAtBottom()
      loading.value = false
      /* await fetchChatAPIProcess<Chat.ConversationResponse>({
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            updateChat(
              +uuid,
              dataSources.value.length - 1,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (data.text ?? ''),
                inversion: false,
                error: false,
                loading: true,
                conversationOptions: { conversationId: data.conversationId, parentMessageId: data.id },
                requestOptions: { prompt: message, options: { ...options } },
              },
            )

            if (openLongReply && data.detail.choices[0].finish_reason === 'length') {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }

            scrollToBottomIfAtBottom()
          }
          catch (error) {
            //
          }
        },
      }) */
      updateChatSome(+uuid, dataSources.value.length - 1, { loading: false })
    }

    await fetchChatAPIOnce()
  }
  catch (error: any) {
    const errorMessage = error?.message ?? t('common.wrong')

    if (error.message === 'canceled') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          loading: false,
        },
      )
      scrollToBottomIfAtBottom()
      return
    }

    const currentChat = getChatByUuidAndIndex(+uuid, dataSources.value.length - 1)

    if (currentChat?.text && currentChat.text !== '') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          text: `${currentChat.text}\n[${errorMessage}]`,
          error: false,
          loading: false,
        },
      )
      return
    }

    updateChat(
      +uuid,
      dataSources.value.length - 1,
      {
        dateTime: new Date().toLocaleString(),
        text: errorMessage,
        inversion: false,
        knowledge: false,
        error: true,
        loading: false,
        conversationOptions: null,
        requestOptions: { prompt: message, options: { ...options } },
      },
    )
    scrollToBottomIfAtBottom()
  }
  finally {
    loading.value = false
  }
}
async function onConversation2() {
  const message = prompt.value
  if (usingContext.value) {
    for (let i = 0; i < dataSources.value.length; i = i + 2)
      history.value.push([`Human:${dataSources.value[i].text}`, `Assistant:${dataSources.value[i + 1].text.split('\n\n数据来源：\n\n')[0]}`])
  }
  else { history.value.length = 0 }
  if (!message || message.trim() === '')
    return

  controller = new AbortController()

  addChat(
    +uuid,
    {
      dateTime: new Date().toLocaleString(),
      text: message,
      inversion: true,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: null },
    },
  )
  scrollToBottom()

  loading.value = true
  prompt.value = ''

  let options: Chat.ConversationRequest = {}
  const lastContext = conversationList.value[conversationList.value.length - 1]?.conversationOptions

  if (lastContext && usingContext.value)
    options = { ...lastContext }

  addChat(
    +uuid,
    {
      dateTime: new Date().toLocaleString(),
      text: '',
      loading: true,
      inversion: false,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
    },
  )
  scrollToBottom()

  try {
    const lastText = ''
    const fetchChatAPIOnce = async () => {
      const res = active.value ? await chatfileOpenai({ message, api_key: store.Openaikey, basePath: store.Openaipath, history: history.value }) : await chatOpenAI({ message, api_key: store.Openaikey, basePath: store.Openaipath, history: history.value })
      const result = active.value ? `${res.data.response.text}\n\n数据来源：\n\n[${res.data.url.split('/static/')[1]}](http://127.0.0.1:3000${res.data.url})` : res.data.text
      updateChat(
        +uuid,
        dataSources.value.length - 1,
        {
          dateTime: new Date().toLocaleString(),
          text: lastText + (result ?? ''),
          inversion: false,
          error: false,
          loading: false,
          conversationOptions: null,
          requestOptions: { prompt: message, options: { ...options } },
        },
      )
      scrollToBottomIfAtBottom()
      loading.value = false
      /* await fetchChatAPIProcess<Chat.ConversationResponse>({
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            updateChat(
              +uuid,
              dataSources.value.length - 1,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (data.text ?? ''),
                inversion: false,
                error: false,
                loading: true,
                conversationOptions: { conversationId: data.conversationId, parentMessageId: data.id },
                requestOptions: { prompt: message, options: { ...options } },
              },
            )

            if (openLongReply && data.detail.choices[0].finish_reason === 'length') {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }

            scrollToBottomIfAtBottom()
          }
          catch (error) {
            //
          }
        },
      }) */
      updateChatSome(+uuid, dataSources.value.length - 1, { loading: false })
    }

    await fetchChatAPIOnce()
  }
  catch (error: any) {
    const errorMessage = error?.message ?? t('common.wrong')

    if (error.message === 'canceled') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          loading: false,
        },
      )
      scrollToBottomIfAtBottom()
      return
    }

    const currentChat = getChatByUuidAndIndex(+uuid, dataSources.value.length - 1)

    if (currentChat?.text && currentChat.text !== '') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          text: `${currentChat.text}\n[${errorMessage}]`,
          error: false,
          loading: false,
        },
      )
      return
    }

    updateChat(
      +uuid,
      dataSources.value.length - 1,
      {
        dateTime: new Date().toLocaleString(),
        text: errorMessage,
        inversion: false,
        error: true,
        loading: false,
        conversationOptions: null,
        requestOptions: { prompt: message, options: { ...options } },
      },
    )
    scrollToBottomIfAtBottom()
  }
  finally {
    loading.value = false
  }
}

async function handleRegenerate(index: number) {
  if (loading.value)
    return

  controller = new AbortController()

  const { requestOptions } = dataSources.value[index]

  const message = requestOptions?.prompt ?? ''

  let options: Chat.ConversationRequest = {}

  if (requestOptions.options)
    options = { ...requestOptions.options }

  loading.value = true

  updateChat(
    +uuid,
    index,
    {
      dateTime: new Date().toLocaleString(),
      text: '',
      inversion: false,
      knowledge: false,
      error: false,
      loading: true,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
    },
  )

  try {
    const lastText = ''
    const data = {
      query: message,
      history: history.value,
      model_name: pickLlm.value,
      stream: true,
      temperature: 0.7,
    }
    const baseData = {
      query: message,
      knowledge_base_name: localStorage.getItem('knowledgeBase'),
      top_k: 3,
      score_threshold: 1,
      history: [],
      model_name: pickLlm.value,
      stream: true,
      temperature: 0.7,
      local_doc_url: false,
    }
    const fetchChatAPIOnce = async () => {
      let result = ''
      if (localStorage.getItem('chatMode') === 'knowledge') {
        const response = await fetch('api/chat/knowledge_base_chat', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(baseData),
        })
        if (!response.body)
          return
        const reader = response.body.pipeThrough(new TextDecoderStream()).getReader()
        while (true) {
          const { value, done } = await reader.read()
          if (done)
            break
          // value = value?.replace('undefined', '')
          console.log('knowledge data -', JSON.parse(value))
          if (JSON.parse(value).answer)
            result += JSON.parse(value).answer
          resultData.value = JSON.parse(value).docs
          // output.value += value?.replace('undefined', '')
          try {
            updateChat(
              +uuid,
              index,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (result ?? ''),
                source: resultData.value,
                knowledge: true,
                inversion: false,
                error: false,
                loading: false,
                conversationOptions: null,
                requestOptions: { prompt: message, options: { ...options } },
              },
            )
          }
          catch (error: any) {}
        }
      }
      // else if (localStorage.getItem('chatMode') === 'LLM' || null) {
      else {
        const resData = {
          text: '',
        }
        const response = await fetch('api/chat/chat', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(data),
        })
        if (!response.body)
          return
        const reader = response.body.pipeThrough(new TextDecoderStream()).getReader()
        while (true) {
          const { value, done } = await reader.read()
          if (done)
            break
          // value = value?.replace('undefined', '')
          console.log('received data -', value)
          resData.text += value
          // resdata = value
          // resData = JSON.parse(resData)
          // output.value += value?.replace('undefined', '')
          try {
            updateChat(
              +uuid,
              index,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (resData.text ?? ''),
                knowledge: false,
                inversion: false,
                error: false,
                loading: false,
                conversationOptions: null,
                requestOptions: { prompt: message, options: { ...options } },
              },
            )
          }
          catch (error: any) {}
        }
      }
      scrollToBottomIfAtBottom()
      loading.value = false
      updateChatSome(+uuid, index, { loading: false })
    }

    await fetchChatAPIOnce()
  }
  catch (error: any) {
    if (error.message === 'canceled') {
      updateChatSome(
        +uuid,
        index,
        {
          loading: false,
        },
      )
      return
    }
    const errorMessage = error?.message ?? t('common.wrong')
    if (localStorage.getItem('chatMode') === 'knowledge') {
      updateChat(
        +uuid,
        index,
        {
          dateTime: new Date().toLocaleString(),
          text: errorMessage,
          inversion: false,
          knowledge: true,
          source: [],
          error: true,
          loading: false,
          conversationOptions: null,
          requestOptions: { prompt: message, options: { ...options } },
        },
      )
    }
    else {
      updateChat(
        +uuid,
        index,
        {
          dateTime: new Date().toLocaleString(),
          text: errorMessage,
          inversion: false,
          knowledge: false,
          error: true,
          loading: false,
          conversationOptions: null,
          requestOptions: { prompt: message, options: { ...options } },
        },
      )
    }
  }
  finally {
    loading.value = false
  }
}

function handleExport() {
  if (loading.value)
    return

  const d = dialog.warning({
    title: t('chat.exportImage'),
    content: t('chat.exportImageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: async () => {
      try {
        d.loading = true
        const ele = document.getElementById('image-wrapper')
        const canvas = await html2canvas(ele as HTMLDivElement, {
          useCORS: true,
        })
        const imgUrl = canvas.toDataURL('image/png')
        const tempLink = document.createElement('a')
        tempLink.style.display = 'none'
        tempLink.href = imgUrl
        tempLink.setAttribute('download', 'chat-shot.png')
        if (typeof tempLink.download === 'undefined')
          tempLink.setAttribute('target', '_blank')

        document.body.appendChild(tempLink)
        tempLink.click()
        document.body.removeChild(tempLink)
        window.URL.revokeObjectURL(imgUrl)
        d.loading = false
        ms.success(t('chat.exportSuccess'))
        Promise.resolve()
      }
      catch (error: any) {
        ms.error(t('chat.exportFailed'))
      }
      finally {
        d.loading = false
      }
    },
  })
}

function handleDelete(index: number) {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.deleteMessage'),
    content: t('chat.deleteMessageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.deleteChatByUuid(+uuid, index)
    },
  })
}

function handleClear() {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.clearChat'),
    content: t('chat.clearChatConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.clearChatByUuid(+uuid)
    },
  })
}
// 回车输入文本对话
function handleEnter(event: KeyboardEvent) {
  if (!isMobile.value) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
  else {
    if (event.key === 'Enter' && event.ctrlKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
}
// 停止返回对话
function handleStop() {
  if (loading.value) {
    controller.abort()
    loading.value = false
  }
}

// 可优化部分
// 搜索选项计算，这里使用value作为索引项，所以当出现重复value时渲染异常(多项同时出现选中效果)
// 理想状态下其实应该是key作为索引项,但官方的renderOption会出现问题，所以就需要value反renderLabel实现
const searchOptions = computed(() => {
  if (prompt.value.startsWith('/')) {
    return promptTemplate.value.filter((item: { key: string }) => item.key.toLowerCase().includes(prompt.value.substring(1).toLowerCase())).map((obj: { value: any }) => {
      return {
        label: obj.value,
        value: obj.value,
      }
    })
  }
  else {
    return []
  }
})

// value反渲染key
const renderOption = (option: { label: string }) => {
  for (const i of promptTemplate.value) {
    if (i.value === option.label)
      return [i.key]
  }
  return []
}

const placeholder = computed(() => {
  if (isMobile.value)
    return t('chat.placeholderMobile')
  return t('chat.placeholder')
})

const buttonDisabled = computed(() => {
  return loading.value || !prompt.value || prompt.value.trim() === ''
})

const footerClass = computed(() => {
  let classes = ['p-4']
  if (isMobile.value)
    classes = ['sticky', 'left-0', 'bottom-0', 'right-0', 'p-2', 'pr-3', 'overflow-hidden']
  return classes
})

onMounted(() => {
  scrollToBottom()
  getLoadModel()
  if (inputRef.value && !isMobile.value)
    inputRef.value?.focus()
})

onUnmounted(() => {
  if (loading.value)
    controller.abort()
})
</script>

<template>
  <div class="flex flex-row w-full h-full" style="background-color: rgb(249, 249, 248);">
    <HeaderComponent
      v-if="isMobile" :using-context="usingContext" @export="handleExport"
      @toggle-using-context="toggleUsingContext"
    />
    <div class="LLM-list" style="width: 200px; margin-top: 5rem;">
      <div v-for="(item, index) in LlmData" :key="index" class="btn" :class="{ btn_actived: item.value === pickLlm }">
        <NImage
          v-show="item.icon" width="30" :src="item.icon"
          :previewed-img-props="{ style: { marginRight: '13px' } }"
        />
        <NButton text style="font-size: 15px;font-weight: 400;" @click="changeModel(item)">
          {{ item.label }}
        </NButton>
      </div>
    </div>
    <div class="chat_box">
      <main class="flex flex-col overflow-hidden mainer">
        <div id="scrollRef" ref="scrollRef" class="h-full overflow-hidden overflow-y-auto">
          <div
            id="image-wrapper" class="w-full max-w-screen-xl m-auto dark:bg-[#101014]"
            :class="[isMobile ? 'p-2' : 'p-4']"
          >
            <template v-if="!dataSources.length">
              <div class="flex items-center justify-center mt-4 text-center text-neutral-300">
                <SvgIcon icon="ri:bubble-chart-fill" class="mr-2 text-3xl" />
                <span>Say something~</span>
              </div>
            </template>
            <template v-else>
              <div>
                <Message
                  v-for="(item, index) of dataSources" :key="index" :date-time="item.dateTime" :text="item.text" :knowledge="item.knowledge"
                  :inversion="item.inversion" :source="item.source" :error="item.error" :loading="item.loading" @delete="handleDelete(index)"
                  @regenerate="handleRegenerate(index)"
                />
                <div class="sticky bottom-0 left-0 flex justify-center">
                  <NButton v-if="loading" type="warning" @click="handleStop">
                    <template #icon>
                      <SvgIcon icon="ri:stop-circle-line" />
                    </template>
                    Stop Responding
                  </NButton>
                </div>
              </div>
            </template>
          </div>
        </div>
        <footer :class="footerClass">
          <div class="w-full max-w-screen-xl m-auto p-4">
            <div class="flex items-center justify-between space-x-2 ml-8">
              <!-- <NSwitch v-model:value="active">
            <template #checked>
              知识库
            </template>
            <template #unchecked>
              知识库&nbsp;&nbsp;
            </template>
          </NSwitch> -->
              <!-- <HoverButton @click="handleClear">
                <span class="text-xl text-[#4f555e] dark:text-white">
                  <SvgIcon icon="ri:delete-bin-line" />
                </span>
              </HoverButton>
              <HoverButton v-if="!isMobile" @click="handleExport">
                <span class="text-xl text-[#4f555e] dark:text-white">
                  <SvgIcon icon="ri:download-2-line" />
                </span>
              </HoverButton>
              <HoverButton v-if="!isMobile" @click="toggleUsingContext">
                <span class="text-xl" :class="{ 'text-[#4b9e5f]': usingContext, 'text-[#a8071a]': !usingContext }">
                  <SvgIcon icon="ri:chat-history-line" />
                </span>
              </HoverButton> -->
              <NAutoComplete v-model:value="prompt" :options="searchOptions" :render-label="renderOption">
                <template #default="{ handleInput, handleBlur, handleFocus }">
                  <NInput
                    ref="inputRef" v-model:value="prompt" type="textarea" :placeholder="placeholder" size="large"
                    :autosize="{ minRows: 1, maxRows: isMobile ? 4 : 8 }" @input="handleInput" @focus="handleFocus"
                    @blur="handleBlur" @keypress="handleEnter"
                  />
                </template>
              </NAutoComplete>
              <NButton type="primary" size="large" :disabled="buttonDisabled" @click="handleSubmit">
                <template #icon>
                  <span class="dark:text-black">
                    <SvgIcon icon="ri:send-plane-fill" />
                  </span>
                </template>
              </NButton>
            </div>
          </div>
        </footer>
      </main>
    </div>
    <NModal v-model:show="show" :mask-closable="false" class="modal">
      <NSpin size="large">
        <template #description style="color: #fff;">
          正在切换模型，请稍后...
        </template>
      </NSpin>
    </NModal>
  </div>
</template>

<style scoped lang="less">
.btn {
  display: flex;
  padding: 2rem;
  margin-left: 15px;
  align-items: center;
}

.btn:hover,
.btn_actived {
  border-top-left-radius: 15px;
  border-bottom-left-radius: 15px;
  background: #f2f3f7;
}

:deep(.n-image img) {
  border-radius: 8px;
  margin-right: 5px;
}

.chat_box {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 15px 15px 15px 0;
  width: calc(100% - 200px);

  .mainer {
    background-color: rgb(242, 243, 247);
    border-radius: 1rem;
    height: 100%;
  }
}

.modal {
  background-color: rgba(255, 255, 255, 0);
  box-shadow: none;
  margin: 0;
  width: 100%;
  color: #fff;
}

:deep(.n-spin-description) {
  color: #fff;
}
</style>
