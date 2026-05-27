<script setup lang="ts">
import { reactive } from 'vue'
import { parse, type ParseResult } from 'papaparse'

type Row = Record<string, string>

interface Test {
  label: string
  useWorker: boolean
  status: 'idle' | 'loading' | 'done' | 'error'
  data: ParseResult<Row> | null
  duration?: number
  error?: string
}

const CSV_URL = `${location.origin}/test.csv`

const tests = reactive<Test[]>([
  { label: 'worker: true',  useWorker: true,  status: 'idle', data: null },
  { label: 'worker: false', useWorker: false, status: 'idle', data: null },
])

function parseCsv(useWorker: boolean): Promise<ParseResult<Row>> {
  return new Promise((resolve, reject) => {
    parse<Row>(CSV_URL, {
      worker: useWorker,
      download: true,
      header: true,
      skipEmptyLines: true,
      complete: resolve,
      error: reject,
    })
  })
}

async function runTest(test: Test) {
  test.status = 'loading'
  test.data = null
  const start = performance.now()
  try {
    test.data = await parseCsv(test.useWorker)
    test.status = 'done'
    test.duration = Math.round(performance.now() - start)
  } catch (e) {
    test.status = 'error'
    test.error = String(e)
  }
}

function hasUndefined(rows: Row[] | undefined) {
  return rows?.some(row => row == null || Object.values(row).some(v => v === undefined || v === null || String(v) === 'undefined'))
}
</script>

<template>
  <div>
    <h1>PapaParse worker regression test</h1>
    <p>CSV: <a :href="CSV_URL" target="_blank">{{ CSV_URL }}</a></p>

    <div>
      <button @click="tests.forEach(runTest)">Run all</button>
      <button v-for="t in tests" :key="t.label" @click="runTest(t)">Run {{ t.label }}</button>
    </div>

    <div v-for="t in tests" :key="t.label">
      <h2>
        {{ t.label }}
        <span v-if="t.status === 'done'">{{ hasUndefined(t.data?.data) ? '✗ BUG: undefined values' : '✓ OK' }}</span>
        <span v-if="t.status === 'loading'">running…</span>
      </h2>
      <p v-if="t.status === 'done'">{{ t.data?.data.length }} rows · {{ t.duration }}ms</p>
      <p v-if="t.status === 'error'">Error: {{ t.error }}</p>
      <pre v-if="t.status === 'done'">{{ JSON.stringify(t.data, null, 2) }}</pre>
      <p v-else-if="t.status === 'idle'">No test run yet.</p>
    </div>
  </div>
</template>

