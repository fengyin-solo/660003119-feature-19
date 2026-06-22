<template>
  <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
    <div class="flex items-center justify-between mb-3">
      <h3 class="text-sm font-bold text-slate-400">完整步骤列表</h3>
      <span v-if="store.matchResult" class="text-xs text-slate-500">
        共 {{ store.matchResult.steps.length }} 步
        <span v-if="searchQuery"> · 匹配 {{ filteredSteps.length }} 步</span>
      </span>
    </div>

    <div class="flex gap-2 mb-3">
      <input
        v-model="searchQuery"
        type="text"
        placeholder="搜索步骤：字符、状态、转移、回溯、失败..."
        class="flex-1 px-3 py-2 bg-slate-900 border border-slate-600 rounded text-sm text-slate-200 placeholder-slate-500 focus:outline-none focus:border-cyan-500"
      />
      <select
        v-model="filterType"
        class="px-2 py-2 bg-slate-900 border border-slate-600 rounded text-sm text-slate-200 focus:outline-none focus:border-cyan-500"
      >
        <option value="all">全部</option>
        <option value="backtrack">仅回溯</option>
        <option value="match">仅匹配</option>
      </select>
    </div>

    <div v-if="filteredSteps.length > 0" class="relative">
      <div ref="listContainer" class="space-y-1 max-h-80 overflow-y-auto pr-1">
        <div
          v-for="step in filteredSteps"
          :key="step.stepIndex"
          @click="handleStepClick(step.stepIndex)"
          class="text-xs font-mono px-2 py-1.5 rounded cursor-pointer transition-colors"
          :class="getStepClass(step)"
        >
          <div class="flex items-center gap-2 flex-wrap">
            <span class="text-slate-500 w-12 shrink-0">[{{ step.stepIndex }}]</span>
            <span class="text-yellow-400 shrink-0" :class="step.char === '\n' ? 'italic' : ''">
              {{ formatChar(step.char) }}
            </span>
            <span class="text-slate-500 shrink-0">→</span>
            <span class="text-green-400 shrink-0">
              {{ step.currentState === -1 ? '—' : 'S' + step.currentState }}
            </span>
            <span class="text-slate-500 shrink-0">→</span>
            <span :class="step.isBacktrack ? 'text-red-400' : 'text-blue-400'" class="shrink-0">
              {{ step.isBacktrack ? '✗ FAIL' : (step.nextState === -1 ? '—' : 'S' + step.nextState) }}
            </span>
            <span class="text-purple-400 ml-1 shrink-0">({{ step.transition }})</span>
            <span class="text-slate-500 text-[10px] shrink-0">@idx{{ step.charIndex }}</span>
            <span v-if="step.isBacktrack" class="text-orange-400 font-bold ml-auto shrink-0">⚠ 回溯</span>
            <span v-else-if="step.isMatch" class="text-green-400 text-[10px] ml-auto shrink-0">✓ 匹配</span>
          </div>
        </div>
      </div>

      <div
        v-if="filteredSteps.length > 20"
        class="absolute right-0 top-0 bottom-0 w-1 bg-slate-700 rounded-full opacity-0 hover:opacity-100 transition-opacity pointer-events-none"
      ></div>
    </div>

    <div v-else-if="store.matchResult && searchQuery" class="text-slate-500 text-sm text-center py-4">
      没有找到匹配的步骤
    </div>
    <div v-else class="text-slate-500 text-sm">等待执行...</div>

    <div v-if="store.matchResult && store.matchResult.steps.length > 0" class="mt-3 pt-3 border-t border-slate-700">
      <div class="flex items-center gap-2 text-xs text-slate-500">
        <span>跳转到步骤:</span>
        <input
          v-model.number="jumpIndex"
          type="number"
          min="0"
          :max="store.matchResult.steps.length - 1"
          class="w-16 px-2 py-1 bg-slate-900 border border-slate-600 rounded text-slate-200 focus:outline-none focus:border-cyan-500"
          @keyup.enter="handleJump"
        />
        <button
          @click="handleJump"
          class="px-2 py-1 bg-cyan-600 hover:bg-cyan-500 rounded text-white"
        >
          跳转
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, nextTick } from 'vue'
import { useRegexStore } from '../store/regex'
import type { MatchStep } from '../types'

const store = useRegexStore()
const searchQuery = ref('')
const filterType = ref('all')
const jumpIndex = ref(0)
const listContainer = ref<HTMLElement | null>(null)

function formatChar(ch: string): string {
  if (ch === '\n') return '\\n'
  if (ch === ' ') return '␣'
  if (ch === '\t') return '\\t'
  return `'${ch}'`
}

const filteredSteps = computed(() => {
  if (!store.matchResult) return []
  let steps = store.matchResult.steps

  if (filterType.value === 'backtrack') {
    steps = steps.filter(s => s.isBacktrack)
  } else if (filterType.value === 'match') {
    steps = steps.filter(s => s.isMatch)
  }

  if (searchQuery.value.trim()) {
    const query = searchQuery.value.toLowerCase().trim()
    const isBacktrackQuery = ['回溯', '失败', 'backtrack', 'fail', '错误', 'error'].some(k => query.includes(k))
    steps = steps.filter(s => {
      const matchBasic =
        s.char.toLowerCase().includes(query) ||
        String(s.currentState).includes(query) ||
        String(s.nextState).includes(query) ||
        s.transition.toLowerCase().includes(query) ||
        String(s.stepIndex).includes(query) ||
        String(s.charIndex).includes(query)
      const matchBacktrack = isBacktrackQuery && s.isBacktrack
      const matchSuccessQuery = ['成功', '匹配', 'success', 'match'].some(k => query.includes(k)) && s.isMatch
      return matchBasic || matchBacktrack || matchSuccessQuery
    })
  }

  return steps
})

function getStepClass(step: MatchStep) {
  if (step.stepIndex === store.currentStep) {
    return 'bg-cyan-900 text-cyan-100 ring-1 ring-cyan-500'
  }
  if (step.isBacktrack) {
    return 'bg-orange-900/30 text-orange-300 hover:bg-orange-900/50'
  }
  return 'bg-slate-900 text-slate-400 hover:bg-slate-700'
}

function handleStepClick(stepIndex: number) {
  store.goToStep(stepIndex)
}

function handleJump() {
  if (jumpIndex.value !== undefined && jumpIndex.value !== null) {
    store.goToStep(jumpIndex.value)
  }
}

watch(() => store.currentStep, () => {
  nextTick(() => {
    if (!listContainer.value) return
    const activeEl = listContainer.value.querySelector('.ring-cyan-500')
    if (activeEl) {
      activeEl.scrollIntoView({ block: 'nearest', behavior: 'smooth' })
    }
  })
})
</script>
