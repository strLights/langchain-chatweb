<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { NCollapse, NCollapseItem, NButton, NInput, NInputNumber, NSelect, NDivider, NUpload, NUploadDragger, NP, NCheckbox, NTag, NDataTable, useMessage } from 'naive-ui'
import type { DataTableColumns } from 'naive-ui'
import { getKnowledge, getKnowledgeDocs } from '../../api/llm/knowledge_base'
import { SvgIcon } from '@/components/common'
const message = useMessage()
type Song = {
  no: number
  title: string
}
const createColumns = ({
  play,
}: {
  play: (row: Song) => void
}): DataTableColumns<Song> => {
  return [
    {
      title: '序号',
      key: 'no',
      width: 50,
    },
    {
      title: '文档名称',
      key: 'title',
      resizable: true,
    },
  ]
}
const tableData = []
const type = ref('samples')
const maxText = ref(250)
const length = ref(0)
const chinese = ref(false)
const options = [
  {
    label: '新建知识库',
    value: 'create',
  },
]
columns: createColumns({
    play (row: Song) {
        message.info(`Play ${row.title}`)
    }
}),
// 获取知识库名称
const getKnowledgeBase = async() => {
  getKnowledge().then((res) => {
    const { code, msg, data } = res.data
    if (code === 200) {
      data.forEach((ele) => {
        options.push({
          label: ele,
          value: ele,
        })
      })
      options.push({
        label: '新建知识库',
        value: 'create',
      })
    }
    else {
      message.error(msg)
    }
  }).catch((err) => {
    // console.log(err)
    message.error(err)
  })
}
// 获取知识库中文档
const getBaseDocs = async () => {
    const param = {
        knowledge_base_name: type.value
    }
    getKnowledgeDocs(param).then((res) => {
    const { code, msg, data } = res.data
    if (code === 200) {
        tableData.push(data)
    }
    else {
      message.error(msg)
    }
  }).catch((err) => {
    // console.log(err)
    message.error(err)
  })
}
onMounted(() => {
  getKnowledgeBase()
  getBaseDocs()
})
</script>

<template>
  <div class="flex flex-col w-full h-full">
    <main class="container">
      <div id="scrollRef" ref="scrollRef" class="h-full overflow-hidden overflow-y-auto">
        <div>
          <label class="label">请选择或新建知识库：</label>
          <NSelect v-model:value="type" style="border-radius: 0.5rem; margin-top: 0.5rem; height: 40px;" :options="options" />
        </div>
        <div v-if="type === 'create'">
          <div class="border">
            <div>
              <label class="label">新建知识库名称</label>
              <NInput style="margin-top: 10px;" placeholder="新知识库名称，不支持中文命名" />
            </div>
            <div class="flex between">
              <div style="width: 49%;">
                <label class="label">向量库类型</label>
                <NSelect v-model:value="type" style="border-radius: 0.5rem; margin-top: 0.5rem; height: 40px;" :options="options" />
              </div>
              <div style="width: 49%;">
                <label class="label">Embedding模型</label>
                <NSelect v-model:value="type" style="border-radius: 0.5rem; margin-top: 0.5rem; height: 40px;" :options="options" />
              </div>
            </div>
            <div style="margin-top: 15px;">
              <NButton style="width: 100%; font-size: 16px;border-radius: 0.5rem;">
                新建
              </NButton>
            </div>
          </div>
        </div>
        <div v-else class="management">
          <div class="add">
            <div class="upload mt-4">
              <label>上传知识文件：</label>
              <NUpload
                style="margin-top: 10px; border-radius: 0.5rem;"
                multiple
                directory-dnd
                action="https://www.mocky.io/v2/5e4bafc63100007100d8b70f"
                :max="15"
              >
                <NUploadDragger class="flex">
                  <div class="flex justify-between items-center" style="margin-right: 24px;">
                    <SvgIcon class="text-4xl" icon="icons8:upload-2" />
                  </div>
                  <div>
                    <NP style="font-size: 16px;margin: 8px 0 0 0;text-align: left;">
                      Drag and drop files here
                    </NP>
                    <NP depth="3" style="margin: 8px 0 0 0;text-align: left;">
                      Limit 200MB per file • HTML, MD, JSON, CSV, PDF, PNG, JPG, JPEG, BMP, EML, MSG, RST, RTF, TXT, XML, DOCX, EPUB, ODT, PPT, PPTX, TSV, HTM
                    </NP>
                  </div>
                </NUploadDragger>
              </NUpload>
            </div>
            <div class="file-config border">
              <NCollapse arrow-placement="right" :default-expanded-names="1" :accordion="true">
                <NCollapseItem title="文件处理配置" name="1">
                  <div class="flex justify-between items-center">
                    <div>
                      <label>单段文本最大长度：</label>
                      <NInputNumber v-model:value="maxText" style="margin-top: 0.5rem;" :max="1000" />
                    </div>
                    <div>
                      <label for="">相邻文本重合长度：</label>
                      <NInputNumber v-model:value="length" style="margin-top: 0.5rem;" :max="1000" />
                    </div>
                    <div>
                      <NCheckbox v-model:checked="chinese">
                        开启中文标题加强
                      </NCheckbox>
                    </div>
                  </div>
                </NCollapseItem>
              </NCollapse>
            </div>
            <NButton style="margin-top: 20px;">
              添加文件到知识库
            </NButton>
          </div>
          <NDivider />
          <div class="docs">
            <NP style="font-size: 16px;margin: 8px 0 0 0;text-align: left;">
              知识库<code class="code">{{ type }}</code>中已有文件：
            </NP>
            <NTag :bordered="false" type="info" style="--n-padding: 16px;line-height:24px; height: auto; width: 100%; border-radius: 0.5rem; font-size: 16px; margin-top: 15px;">
              知识库中包含源文件与向量库，请从下表中选择文件后操作
            </NTag>
            <NDataTable striped style="margin-top: 15px;" />
          </div>
          <NDivider />
          <div class="flex between">
            <NButton type="info">依据源文件重建向量库</NButton>
            <NButton>删除知识库</NButton>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style lang="less" scoped>
.container {
    width: 100%;
    padding: 6rem 1rem 10rem;
    max-width: 46rem;
    margin: 0 auto;
    .border {
        border: 1px solid rgba(0, 0, 0, 0.2);
        border-radius: 0.5rem;
        padding: 1rem;
        margin-top: 20px;
    }
    .between {
        margin-bottom: 20px;
        justify-content: space-between;
    }
    :deep(.n-upload-dragger) {
        border-radius: 0.5rem;
    }
    :deep(.n-button) {
        padding: 12px 24px;
        height: auto;
    }
    .code {
        color: rgb(9, 171, 59);
        overflow-wrap: break-word;
        padding: 0.2em 0.4em;
        border-radius: 0.25rem;
        background-color: rgb(248,249,251);
    }
}
</style>
