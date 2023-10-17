<script setup lang='ts'>
import type { CSSProperties } from 'vue'
import { computed, ref, watch, onMounted } from 'vue'
import { router } from '@/router'
import { NButton, NLayoutSider, NRadioGroup, NUpload, useMessage, NSelect, NCollapse, NCollapseItem, NInputNumber, NSlider } from 'naive-ui'
import List from './List.vue'
import step from './step.vue'
import filelist from './filelist.vue'
import Footer from './Footer.vue'
import { useAppStore, useChatStore } from '@/store'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { SvgIcon } from '@/components/common'
import { getKnowledge } from '@/api/llm/knowledge_base'

const appStore = useAppStore()
const chatStore = useChatStore()
const message = useMessage()

const { isMobile } = useBasicLayout()
const show = ref(false)
const chatType = ref('LLM')
const chatBase = ref('samples')
const menu = ref(1)
const temperature = ref(0.7)
const chatSteps = ref(3)
const marks = {
  0: '0.00',
  1: '1.00',
}
const songs = [
  {
    value: 1,
    label: '对话',
    icon: 'ph:chat-text-light',
  },
  // {
  //   value: 2,
  //   label: '模型',
  // },
  {
    value: 3,
    label: '知识库管理',
    icon: 'heroicons:inbox-stack',
  },
  // {
  //   value: 4,
  //   label: '提示词',
  // },
]
const options = [
  {
    label: 'LLM对话',
    value: 'LLM',
  },
  {
    label: '知识库对话',
    value: 'knowledge',
  },
  {
    label: '搜索引擎问答',
    value: 'search',
  },
]
const knowledgeList = ref([])
const collapsed = computed(() => appStore.siderCollapsed)

function handleAdd() {
  menu.value = 1
  chatStore.addHistory({ title: 'New Chat', uuid: Date.now(), isEdit: false })
  if (isMobile.value)
    appStore.setSiderCollapsed(true)
}

function handleUpdateCollapsed() {
  appStore.setSiderCollapsed(!collapsed.value)
}

const getMobileClass = computed<CSSProperties>(() => {
  if (isMobile.value) {
    return {
      position: 'fixed',
      zIndex: 50,
      backgroundColor: '#ddd',
    }
  }
  return {}
})

const getBtnClass = computed<CSSProperties>(() => {
  if (isMobile.value) {
    return {
      position: 'fixed',
      zIndex: 50,
      backgroundColor: '#ddd',
    }
  }
  return {}
})

const mobileSafeArea = computed(() => {
  if (isMobile.value) {
    return {
      paddingBottom: 'env(safe-area-inset-bottom)',
    }
  }
  return {}
})
//
const siderBtn = (param) => {
  if(param === 1) {
    router.push({
      path: '/',
    })
  } else {
    router.push({
      path: '/knowledge/base',
    })
  }
}

watch(
  isMobile,
  (val) => {
    appStore.setSiderCollapsed(val)
  },
  {
    immediate: true,
    flush: 'post',
  },
)

const getKnowledgeBase = async () => {
  getKnowledge().then(res => {
    const { code, msg, data } = res.data
    if (code === 200) {
      // console.log(data)
      data.forEach(ele => {
        knowledgeList.value.push({
          label: ele,
          value: ele,
        })
      })
    } else {
      message.error(msg)
    }
  }).catch(err => {
    console.log(err)
    message.error(err)
  })
}
const storeBase = (val: string) => {
  localStorage.setItem('knowledgeBase', val)
}
const storeChatMode = (val: string) => {
  localStorage.setItem('chatMode', val)
}
onMounted(() => {
  getKnowledgeBase()
})
</script>

<template>
  <NLayoutSider :collapsed="collapsed" :collapsed-width="0" style="overflow: hidden;" :width="320" :show-trigger="isMobile ? false : 'arrow-circle'"
    collapse-mode="transform" position="absolute" bordered :style="getMobileClass"
    @update-collapsed="handleUpdateCollapsed">
    <Footer style="padding: 3rem 1rem;" />
    <div class="flex flex-col h-full " style="padding: 1rem;" :style="mobileSafeArea">
      <main class="flex flex-col flex-1 min-h-0">
        <div class="flex-col flex justify-between">
          <NRadioGroup v-model:value="menu" name="radiobuttongroup1">
            <!-- <NRadioButton
              v-for="song in songs"
              :key="song.value"
              :value="song.value"
              :label="song.label"
            /> -->
            <div v-for="song in songs" :key="song.value" @click="menu = song.value">
              <NButton text class="side-btn"
                style="border: none; font-size: 1rem; font-weight: 800; padding: 10px 14px;"
                :style="{ backgroundColor: song.value === menu ? '#fff' : 'transparent', borderRadius: song.value === menu ? '10px' : '0', color: song.value === menu ? '#000' : '#333', fontWeight: song.value === menu ? '800' : '300' }"
                :value="song.value"
                @click="siderBtn(song.value)">
                <template #icon>
                  <SvgIcon :icon="song.icon" />
                </template>
                {{ song.label }}
              </NButton>
              <!-- <button block class="side-btn" style="border: none; font-size: 1rem; font-weight: 800;" :value="song.value">
                <SvgIcon icon="ri:stop-circle-line" />
                {{ song.label }}
              </button> -->
            </div>
          </NRadioGroup>
        </div>
        <!-- 知识库界面 -->
        <div v-if="menu === 2">
          <div class="p-4">
            <NUpload action="http://127.0.0.1:1002/api/file" :headers="{
              'naive-info': 'hello!',
            }" :data="{ 'naive-data': 'cool! naive!' }">
              <NButton block>
                文件上传
              </NButton>
            </NUpload>
          </div>
          <div class="p-2 flex-1 min-h-0 pb-4 overflow-hidden">
            <filelist />
          </div>
        </div>
        <!-- 会话界面 -->
        <div v-if="menu === 1">
          <div class="p-2">
            <label>请选择对话模式：</label>
            <NSelect v-model:value="chatType" style="border-radius: 0.5rem; margin-top: 0.5rem; height: 40px;" :options="options" @update:value="storeChatMode(chatType)" />
            <!-- <NButton block @click="handleAdd">
              新建聊天
            </NButton> -->
          </div>
          <!-- <div class="p-2 flex-1 min-h-0 pb-4 overflow-hidden">
            <List />
          </div> -->
        </div>
        <div class="params" v-if="menu === 1">
          <div class="p-2">
            <label>Temperature:</label>
            <NSlider
              style="margin-top: 2rem;--n-fill-color: rgb(22,93,255);--n-fill-color-hover: rgb(22,93,255);--n-dot-border-active: 2px solid rgb(22,93,255)"
              v-model:value="temperature" :step="0.05" :min="0.00" :max="1.00" :marks="marks" show-tooltip />
          </div>
          <div class="p-2">
            <label for="">历史对话轮数：</label>
            <NInputNumber style="margin-top: 0.5rem;" v-model:value="chatSteps" :max="10" />
          </div>
        </div>
        <!-- 知识库对话 -->
        <div v-if="chatType === 'knowledge' && menu === 1">
          <div class="knowledge-setting">
            <NCollapse arrow-placement="right" :default-expanded-names="['1']">
              <NCollapseItem title="知识库配置" name="1">
                <div>
                  <div>
                    <label>请选择知识库：</label>
                    <NSelect v-model:value="chatBase" style="border-radius: 0.5rem;" :options="knowledgeList" @update:value="storeBase(chatBase)" />
                  </div>
                  <div style="margin-top: 0.5rem;">
                    <label for="">匹配知识条数：</label>
                    <NInputNumber v-model:value="chatSteps" :max="10" />
                  </div>
                  <div style="margin-top: 0.5rem;">
                    <label>知识匹配分数阈值：</label>
                    <NSlider
                      style="margin-top: 2rem;--n-fill-color: rgb(22,93,255);--n-fill-color-hover: rgb(22,93,255);--n-dot-border-active: 2px solid rgb(22,93,255)"
                      v-model:value="temperature" :step="0.05" :min="0.00" :max="1.00" :marks="marks" show-tooltip />
                  </div>
                </div>
              </NCollapseItem>
            </NCollapse>
          </div>
        </div>
        <!-- <div class="p-4">
          <NButton block @click="show = true">
            {{ $t('store.siderButton') }}
          </NButton>
        </div> -->
      </main>
    </div>
  </NLayoutSider>
  <template v-if="isMobile">
    <div v-show="!collapsed" class="fixed inset-0 z-40 w-full h-full bg-black/40" @click="handleUpdateCollapsed" />
  </template>
  <PromptStore v-model:visible="show" />
</template>

<style lang="less" scoped>
:deep(.n-layout-sider .n-layout-sider-scroll-container) {
  overflow: hidden !important;
}
.side-btn {
  width: 100%;
  justify-content: left;
  color: #333;
}

.side-btn:hover {
  background-color: #fff;
  border: none !important;
}

:deep(.n-button:not(.n-button--disabled):hover) {
  background-color: #fff;
  color: #000;
}

:deep(.n-button:active) {
  background-color: #fff;
  color: #000;
}

.knowledge-setting {
  padding: 1rem 0.5rem;
}

:deep(.n-slider .n-slider-handles .n-slider-handle-wrapper .n-slider-handle) {
  width: 12px;
  height: 12px;
}

:deep(.n-base-selection) {
  min-height: 40px;
  height: 40px !important;
}

:deep(.n-base-selection .n-base-selection-label) {
  height: 40px;
}

:deep(.n-input) {
  height: 40px;
  line-height: 40px;
}

:deep(.n-slider-handle-indicator) {
  background-color: rgba(255, 255, 255, 0) !important;
  box-shadow: none !important;
}

:deep(.n-slider-handle-indicator.n-slider-handle-indicator--top) {
  margin-bottom: 0 !important;
}
</style>