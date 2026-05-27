<script setup lang="ts">
import { reactive } from 'vue'
import { parse } from 'papaparse'

type Row = Record<string, string>

interface Test {
  label: string
  useWorker: boolean
  status: 'idle' | 'loading' | 'done' | 'error'
  data: Row[]
  duration?: number
  error?: string
}

const CSV_URL = `${location.origin}/test.csv`

const tests = reactive<Test[]>([
  { label: 'worker: true',  useWorker: true,  status: 'idle', data: [] },
  { label: 'worker: false', useWorker: false, status: 'idle', data: [] },
])

function parseCsv(useWorker: boolean): Promise<Row[]> {
  return new Promise((resolve, reject) => {
    parse<Row>(CSV_URL, {
      worker: useWorker,
      download: true,
      header: true,
      skipEmptyLines: true,
      complete: (r) => resolve(r.data),
      error: reject,
    })
  })
}

async function runTest(test: Test) {
  test.status = 'loading'
  test.data = []
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

function hasUndefined(rows: Row[]) {
  return rows.some(row => Object.values(row).some(v => v === undefined || v === null || String(v) === 'undefined'))
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
        <span v-if="t.status === 'done'">{{ hasUndefined(t.data) ? '✗ BUG: undefined values' : '✓ OK' }}</span>
        <span v-if="t.status === 'loading'">running…</span>
      </h2>
      <p v-if="t.status === 'done'">{{ t.data.length }} rows · {{ t.duration }}ms</p>
      <p v-if="t.status === 'error'">Error: {{ t.error }}</p>
      <table v-if="t.data.length">
        <thead>
          <tr><th v-for="h in Object.keys(t.data[0])" :key="h">{{ h }}</th></tr>
        </thead>
        <tbody>
          <tr v-for="(row, i) in t.data" :key="i">
            <td v-for="h in Object.keys(t.data[0])" :key="h">{{ row[h] ?? 'undefined' }}</td>
          </tr>
        </tbody>
      </table>
      <p v-else-if="t.status === 'idle'">No test run yet.</p>
    </div>

    <details>
      <summary>Raw JSON</summary>
      <div v-for="t in tests" :key="t.label">
        <strong>{{ t.label }}</strong>
        <pre>{{ JSON.stringify(t.data, null, 2) }}</pre>
      </div>
    </details>
  </div>
</template>

