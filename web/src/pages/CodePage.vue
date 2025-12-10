<template>
  <div class="bg-[#282c34] text-md flex justify-center items-center w-[100vw] flex-1 min-h-0">
    <div class="w-3/4 h-3/4 bg-[#282c34] text-sm scale-133 flex">
      <div class="w-3/4 h-full min-w-0 ">
        <div ref="editorContainer" class="w-full h-full">
        </div>
      </div>

      <div class="flex-1 bg-stone-900 min-w-0 text-xs">
        <div class="h-1/2 text-white flex flex-col">
          <div class="bg-stone-700 p-1 px-2 flex items-center">
            <div>
              输入
            </div>
            <div class="flex-1 min-w-0">
            </div>
            <div class="h-4 w-4 mr-2" @click="handleRun(false)">
              <img :src="saveIcon" alt="run">
            </div>
            <div class="h-4 w-4" @click="handleRun(true)">
              <img :src="runIcon" alt="run">
            </div>
          </div>
          <div class="flex-1 min-h-0 ">
            <div class="w-full h-full overflow-hidden">
              <textarea name="stdin" id="stdin"
                class="w-full h-full resize-none outline-none border-0 p-2 whitespace-nowrap" placeholder="请输入测试样例"
                v-model="stdin"></textarea>
            </div>
          </div>
        </div>
        <div class="h-1/2 text-white flex flex-col">
          <div class="bg-stone-700 p-1 px-2">
            输出
          </div>
          <div class="flex-1 min-h-0 overflow-auto">
            <div class="w-full h-full p-2">
              <!-- {{ stdout === '' ? 'Empty' : stdout }} -->
              <div v-show="time !== 0">
                运行时间：{{ time }} ms
              </div>
              <pre>{{ status !== 'completed' ? status : (stdout === '' ? 'Empty' : stdout) }}</pre>
              <div v-show="stderr !== ''" class="text-red-500">
                {{ stderr }}
              </div>
              <div v-show="log !== ''" class="text-red-500">
                {{ log }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, defineProps, onUnmounted } from 'vue'
import { EditorView } from '@codemirror/view'
import { EditorState } from '@codemirror/state'
import { cpp } from '@codemirror/lang-cpp'
import { basicSetup } from 'codemirror'
import { languageServer } from 'codemirror-languageserver';
import { keymap } from '@codemirror/view'
import { indentMore, indentLess, insertTab, indentWithTab } from "@codemirror/commands";
import { oneDark } from "@codemirror/theme-one-dark";
import runIcon from '../assets/run.svg'
import saveIcon from '../assets/save.svg'
import { useRouter } from 'vue-router'
import { indentUnit } from '@codemirror/language';

const router = useRouter()

const props = defineProps({
  id: {
    type: String,
    default: null,
  }
});

const stdin = ref('')
const stdout = ref('')
const status = ref('completed')
const stderr = ref('')
const time = ref(0)
const log = ref('')
const isLoading = ref(false)
const editorView = ref<EditorView | null>(null)

const serverUri = window.CONFIG.LSP_SERVER !== '__LSP_SERVER_URL_PLACEHOLDER__' ? window.CONFIG.LSP_SERVER : import.meta.env.VITE_LSP_SERVER;
const backend = window.CONFIG.BACKEND !== '__BACKEND_URL_PLACEHOLDER__' ? window.CONFIG.BACKEND : import.meta.env.VITE_BACKEND;
const ls = languageServer({
  serverUri,
  rootUri: 'file:///main.cpp',
  workspaceFolders: [],
  documentUri: `file:///main.cpp`,
  languageId: 'cpp',
});
const editorContainer = ref<HTMLElement | null>(null)

function getStatus(id: string) {
  fetch(backend + `/api/pastes/${props.id}`)
    .then(res => res.json())
    .then(res => {
      stdin.value = res.stdin
      status.value = res.status
      stdout.value = res.stdout
      stderr.value = res.stderr
      time.value = res.execution_time_ms
      log.value = res.compile_log
    })
    .catch(err => {
      console.log(err)
    })
}

onMounted(() => {

  const oldCode = localStorage.getItem('code')
  stdin.value = localStorage.getItem('stdin') || ''
  const state = EditorState.create({
    doc: oldCode || `#include <bits/stdc++.h>\n\nint main() {\n\n}`,
    extensions: [
      basicSetup,
      cpp(),
      ls,
      indentUnit.of("    "),
      EditorView.theme({
        "&": { height: "100%" },
        ".cm-scroller": { overflow: "auto" }
      }),
      EditorView.updateListener.of((update) => {
        // 只有当文档内容发生改变 且 选区也改变时才触发
        if (update.docChanged && update.selectionSet) {
          const cursorParams = update.state.selection.main.head;
          
          update.view.dispatch({
            // 🌟 终极方案：只用 nearest
            // 含义：确保光标在视野内。如果在，不动；如果不在，滚进来。
            effects: EditorView.scrollIntoView(cursorParams, { y: "nearest" })
          });
        }
      }),
      keymap.of([
        { key: "Tab", run: indentMore },
        { key: "Shift-Tab", run: indentLess },
        // 如果需要在行中插入真实 Tab：
        { key: "Mod-Tab", run: insertTab },
      ]),
      oneDark,
    ]
  })
  editorView.value = new EditorView({
    state,
    parent: editorContainer.value as HTMLElement
  })
  console.log(props.id)
  if (props.id !== null && props.id !== undefined && props.id !== '') {
    isLoading.value = true
    fetch(backend + `/api/pastes/${props.id}`)
      .then(res => res.json())
      .then(res => {
        if (!editorView.value) {
          return;
        }
        editorView.value.dispatch({
          changes: {
            from: 0,
            to: editorView.value.state.doc.length,
            insert: res.code
          }
        })
        stdin.value = res.stdin
        status.value = res.status
        stdout.value = res.stdout
        stderr.value = res.stderr
        time.value = res.execution_time_ms
        log.value = res.compile_log
        if (status.value === 'pending' || status.value === 'running') {
          let timer = setInterval(() => {
            getStatus(props.id)
            if (status.value !== 'pending' && status.value !== 'running') {
              clearInterval(timer)
              isLoading.value = false
            }
          }, 1000)
        } else {
          isLoading.value = false
        }
      })
      .catch(err => {
        console.log(err)
      })

  }
})

onUnmounted(() => {
  if (editorView.value) {
    console.log("Destroying CodeMirror EditorView and closing LSP connection...");
    editorView.value.destroy();
    editorView.value = null;
    console.log("EditorView destroyed.");
  }
});

const handleRun = (isRun: boolean) => {
  if (isLoading.value) {
    console.log("别急")
    return
  }
  if (!editorView.value) {
    return;
  }
  isLoading.value = true
  console.log('run')
  fetch(backend + '/api/pastes', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      code: editorView.value.state.doc.toString() ?? '',
      stdin: isRun ? stdin.value : '',
      language: 'c++20',
      run: isRun
    })
  })
    .then(res => res.json())
    .then(res => {
      if (res.message === 'Created') {
        const pasteid = res.paste_id;
        router.push({
          name: 'code',
          params: {
            id: pasteid
          }
        })
      }
    })
    .catch(err => {
      console.log(err)
    })
}

setInterval(() => {
  if (editorView.value) {
    localStorage.setItem('code', editorView.value.state.doc.toString())
    localStorage.setItem('stdin', stdin.value)
  }
}, 1000)



</script>