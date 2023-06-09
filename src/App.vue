<script setup lang="ts">
import { computed, ref } from 'vue';

type CoverageFile = {
  url: string,
  text: string,
  ranges: {
    start: number,
    end: number,
  }[],
  rate: number,
}

const coverageFiles = ref<CoverageFile[]>([])
const fileName = ref('')
const fileFilter = ref('')

const filteredCoverageFiles = computed(() => {
  return coverageFiles.value.filter(f => f.url.includes(fileFilter.value))
})

const selectFile = () => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = '.json' // json file only
  input.onchange = (e) => {
    const target = e.target as HTMLInputElement
    const file: File = (target.files as FileList)[0]
    fileName.value = file.name
    readFile(file).then((result) => {
      coverageFiles.value = (JSON.parse(result) as CoverageFile[])
        .filter(f => f.text?.length > 0 && (f.url.endsWith('.css') || f.url.endsWith('.js')))
        .map(f => ({
          ...f,
          rate: f.ranges.reduce((acc, range) => acc + (range.end - range.start), 0) * 100 / f.text.length
        })).sort((a, b) => a.rate - b.rate)
    })
  }
  input.click()
}

const readFile = async (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (e) => {
      const target = e.target as FileReader
      const result = target.result as string
      resolve(result)
    }
    reader.onerror = (e) => {
      const target = e.target as FileReader
      const error = target.error
      reject(error)
    }
    reader.readAsText(file)
  })
}

const extractFile = (iFile: number) => {
  const file = coverageFiles.value[iFile]
  const ranges = file.ranges
  const text = file.text
  const extractedText = ranges.map(range => {
    if (text.slice(range.start - 7, range.start) === '@media ') {
      return text.slice(range.start - 7, range.end)
    }
    return text.slice(range.start, range.end)
  }
  ).join('')

  saveFile(extractedText, file.url.split('/').pop() || '')
}

const saveFile = (text: string, filename: string) => {
  const blob = new Blob([text], {
    type: 'text/plain',
    endings: 'native',
  })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
}


</script>

<template>
  <header>
    <img class="logo" src="/logo.svg" alt="Logo" />
    <h1>Coverage Extractor</h1>
    <button @click="selectFile">Select Coverage JSON File</button>
    <span v-if="fileName">{{ fileName }}</span>
    <a href="https://github.com/openhoangnc/coverage-extractor" target="_blank">
      Github</a>
  </header>
  <div class="card-header" v-if="coverageFiles.length">
    <span class="file-name">File
      <input type="text" v-model="fileFilter" placeholder="Filter" />
    </span>
    <span class="unused-col">Unused Rate</span>
    <span class="actions-col">Actions</span>
  </div>
  <div v-if="coverageFiles" class="card" v-for="(file, iFile) in filteredCoverageFiles" :key="file.url">
    <a class="file-name" :href="file.url" target="_blank">{{ file.url }}</a>
    <span class="unused-col">{{ (100 - file.rate).toFixed(2) }}%</span>
    <button class="actions-col" @click="extractFile(iFile)">Extract</button>
  </div>
</template>

<style scoped>
.logo {
  height: 36px;
  width: 36px;
}

header {
  display: flex;
  align-items: center;
  gap: 8px;
  height: 36px;
  padding-bottom: 20px;
  justify-content: space-between;
}

header :nth-last-child(1) {
  flex: 1;
  text-align: right;
}

.card {
  padding: 4px;
  font-weight: bold;
  border-bottom: 1px solid #cccccc72;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding-left: 8px;
}

.unused-col {
  width: 120px;
  text-align: right;
}

.actions-col {
  width: 100px;
  text-align: center;
}

.card-header {
  font-weight: bold;
  background-color: #383838;
  font-size: large;
  padding: 4px;
  font-weight: bold;
  border-bottom: 1px solid #cccccc72;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding-left: 12px;
  position: -webkit-sticky;
  /* Safari */
  position: sticky;
  top: 0;
}

.card:hover {
  background-color: #7a6df02d;
}

.file-name {
  flex: 1;
  margin-right: 8px;
}

.card-header .file-name {
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: flex-start;
}

.file-name input {
  flex: 1;
  height: 24px;
  padding: 0 8px;
}
</style>
