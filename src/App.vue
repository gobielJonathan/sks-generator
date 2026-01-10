<script setup lang="ts">
import { nextTick, ref, watchEffect } from 'vue';
import readXlsxFile from 'read-excel-file'
import * as XLSX from 'xlsx'
import templateCSV from './assets/template-sks-ulang.xlsx'

type Subject = {
  kode_mata_kuliah: boolean,
  nama_mata_kuliah: boolean,
  sks: number
  is_lab: boolean,
  is_taken_now: boolean
}

type PeriodSubject = {
  [period: string]: Subject[]
}

const FAILED_SUBJECT_STORAGE_KEY = 'failed-subject';
const ROMAN_REGEX = /\b[IVXLCDM]+\b/i
const MAX_SEMESTER = 14;
const MAX_SKS = 146;
const MAX_SKS_PER_SEMESTER = 24;
const GENAP = 0
const GANJIL = 1
const MIN_SKS_TAKE_ENRICHMENT = 60
const SKS_ENRICHMENT = 20
const ENRICHMENT_SUBJECT_CODE = 'Enrichment'
const THESIS_SUBJECT_CODE = 'Thesis'
const SKS_THESIS = 6

const runningSemester = ref(1);
const runningSKS = ref(1);
const generateSKSTableRef = ref<HTMLElement | null>(null);
const failedSubject = ref<PeriodSubject>(safeParseJSON(localStorage.getItem(FAILED_SUBJECT_STORAGE_KEY) || '{}'));
const futureSubject = ref<PeriodSubject>({});

watchEffect(() => {
  localStorage.setItem(FAILED_SUBJECT_STORAGE_KEY, JSON.stringify(failedSubject.value));
});

const previewUploadRepeatFile = async (file: FileList) => {
  const _file = file.item(0);
  if (!_file) return;
  try {
    const rows = await readXlsxFile(_file);
    const data = transformExcelRows(rows)
    console.log(rows, data)
    failedSubject.value = data as unknown as PeriodSubject
  } catch (error) {
    alert('Gagal membaca file. Pastikan format file sesuai dan coba lagi.')
  }
}


function safeParseJSON(jsonString: string) {
  try {
    return JSON.parse(jsonString)
  } catch (e) {
    return null
  }
}

async function downloadCSV() {
  await nextTick() // ensure DOM is rendered

  const ws = XLSX.utils.table_to_sheet(generateSKSTableRef.value)
  const wb = XLSX.utils.book_new()

  XLSX.utils.book_append_sheet(wb, ws, 'Grades')
  XLSX.writeFile(wb, 'grades.xlsx')
}

const generateFutureSubject = () => {

  const subjectPerSemester: Record<number, Subject[]> = { [GENAP]: [], [GANJIL]: [] }

  //seperate all subjects by ganjil / genap semester
  for (const [period, subjects] of Object.entries(failedSubject.value)) {
    const periodInfo = parsePeriodString(period)
    if (!periodInfo) continue
    subjectPerSemester[periodInfo.semester_number % 2 === 1 ? GANJIL : GENAP].push(...subjects)
  }

  const _futureSubject: PeriodSubject = {}
  let _year = new Date().getFullYear()
  let _incrementYear = 0
  let _runningSKS = runningSKS.value
  let _runningSemester = runningSemester.value
  let _countEnrichmentTaken = 0
  let _isAlreadyGetThesis = false

  const _period = createPeriodString(_year + _incrementYear, numberToRoman(runningSemester.value))
  _futureSubject[_period] ??= []
  //get taken now subjects first
  const takenNowSubjects = subjectPerSemester[getSemesterType(_runningSemester)]?.filter(s => s.is_taken_now) || []
  if (takenNowSubjects.some(s => s.kode_mata_kuliah.includes('Enrichment'))) {
    _countEnrichmentTaken++
  }

  _futureSubject[_period].push(...takenNowSubjects)
  //remove taken now subjects from the pool
  takenNowSubjects.forEach(subject => {
    subjectPerSemester[getSemesterType(_runningSemester)] = removeSubjectFromSemester(
      subjectPerSemester[getSemesterType(_runningSemester)] || [],
      subject
    )
  })
  _runningSemester++
  if (_runningSemester % 2 === 0) _incrementYear++

  while (_runningSemester < MAX_SEMESTER && _runningSKS < MAX_SKS) {
    if (subjectPerSemester[GENAP].length === 0 && subjectPerSemester[GANJIL].length === 0) {
      //no more subjects to take
      break
    }

    const _period = createPeriodString(_year + _incrementYear, numberToRoman(_runningSemester))
    _futureSubject[_period] ??= []

    if (_runningSKS >= MIN_SKS_TAKE_ENRICHMENT && _countEnrichmentTaken < 2) {
      //add enrichment if possible
      const enrichmentSubject: Subject = {
        kode_mata_kuliah: ENRICHMENT_SUBJECT_CODE,
        nama_mata_kuliah: 'Enrichment ' + numberToRoman(_countEnrichmentTaken + 1),
        sks: SKS_ENRICHMENT,
        is_lab: false,
        is_taken_now: false
      }
      if (canBeGroup(_futureSubject[_period].concat(enrichmentSubject))) {
        _futureSubject[_period].push(enrichmentSubject)
        _runningSKS += SKS_ENRICHMENT
        _countEnrichmentTaken++
      }

    }
    if (_countEnrichmentTaken === 2 && _isAlreadyGetThesis === false) {
      //add thesis if possible, student get thesis after max subject left max 8
      if (Object.values(subjectPerSemester).flat().length > 8) {
        //still have more than 8 subjects left, skip thesis
        continue
      }
      const thesisSubject: Subject = {
        kode_mata_kuliah: THESIS_SUBJECT_CODE,
        nama_mata_kuliah: 'Thesis',
        sks: SKS_THESIS,
        is_lab: false,
        is_taken_now: false
      }
      if (canBeGroup(_futureSubject[_period].concat(thesisSubject))) {
        _futureSubject[_period].push(thesisSubject)
        _runningSKS += SKS_THESIS
        _isAlreadyGetThesis = true
      }
    }

    const subjects = subjectPerSemester[getSemesterType(_runningSemester)] || []

    for (const subject of subjects) {
      if (canBeGroup([..._futureSubject[_period], subject])) {
        _runningSKS += subject.sks
        _futureSubject[_period].push(subject)

        subjectPerSemester[getSemesterType(_runningSemester)] = removeSubjectFromSemester(
          subjectPerSemester[getSemesterType(_runningSemester)] || [],
          subject
        )
      }
    }

    if (_runningSemester % 2 === 0) _incrementYear++
    _runningSemester++
  }

  futureSubject.value = _futureSubject
}

function transformExcelRows(rows: any[][]) {
  const result: Record<string, any[]> = {}
  let currentSemester = null

  // skip header row
  for (let i = 1; i < rows.length; i++) {
    const row = rows[i] || []

    const [
      semester,
      _kode,
      nama,
      sks
    ] = row
    const kode = _kode || nama

    // detect new semester
    if (semester && typeof semester === 'string' && semester.trim()) {
      currentSemester = semester.trim()
      if (!result[currentSemester]) {
        result[currentSemester] = []
      }
    }

    // skip invalid rows
    if (!currentSemester) continue
    if (!kode || !nama || !sks) continue

    result[currentSemester]?.push({
      kode_mata_kuliah: String(kode).trim(),
      nama_mata_kuliah: String(nama).trim(),
      sks: Number(sks)
    })
  }

  return result
}

function numberToRoman(num: number): string {
  if (typeof num !== 'number' || num <= 0 || num >= 4000) {
    throw new Error('Number must be between 1 and 3999')
  }

  const map = [
    { value: 1000, symbol: 'M' },
    { value: 900, symbol: 'CM' },
    { value: 500, symbol: 'D' },
    { value: 400, symbol: 'CD' },
    { value: 100, symbol: 'C' },
    { value: 90, symbol: 'XC' },
    { value: 50, symbol: 'L' },
    { value: 40, symbol: 'XL' },
    { value: 10, symbol: 'X' },
    { value: 9, symbol: 'IX' },
    { value: 5, symbol: 'V' },
    { value: 4, symbol: 'IV' },
    { value: 1, symbol: 'I' }
  ]

  let result = ''

  for (const { value, symbol } of map) {
    while (num >= value) {
      result += symbol
      num -= value
    }
  }

  return result
}


function romanToNumber(roman: string): number {
  if (!roman) return 0

  const map = {
    I: 1,
    V: 5,
    X: 10,
    L: 50,
    C: 100,
    D: 500,
    M: 1000
  }

  roman = roman.toUpperCase()

  let total = 0
  let prev = 0

  for (let i = roman.length - 1; i >= 0; i--) {
    const curr = map[roman[i]]
    if (!curr) return NaN // invalid roman character

    if (curr < prev) {
      total -= curr
    } else {
      total += curr
      prev = curr
    }
  }

  return total
}

function parsePeriodString(input: string) {
  if (!input || typeof input !== 'string') return null

  const match = input.match(
    /^\s*(\d{2})\.(\d+)\s*\/\s*([IVXLCDM]+)\s*$/i
  )

  if (!match) return null

  const [, yearShort, period, semester = ''] = match

  return {
    year: 2000 + Number(yearShort),
    period: Number(period),
    semester: semester.toLowerCase() ?? '',
    semester_number: romanToNumber(semester)
  }
}

function canBeGroup(subjects: Subject[]) {
  //enrichment engga boleh diambil barengan sama lab
  if (subjects.some(s => s.kode_mata_kuliah === ENRICHMENT_SUBJECT_CODE) &&
    subjects.some(s => s.is_lab)) {
    return false
  }

  return subjects.reduce((acc, curr) => acc + curr.sks, 0) < MAX_SKS_PER_SEMESTER
}

function getSemesterType(semesterNumber: number) {
  return semesterNumber % 2 === 1 ? GANJIL : GENAP
}

function removeSubjectFromSemester(
  subjects: Subject[],
  subjectToDelete: Subject
) {
  return subjects.filter(subject => subject.kode_mata_kuliah !== subjectToDelete.kode_mata_kuliah)
}

function createPeriodString(
  year: number,
  roman_semester: string
) {
  const yearShort = String(year).slice(-2).padStart(2, '0')
  const periodStr = romanToNumber(roman_semester) % 2 === 1 ? '1' : '2'
  return `${yearShort}.${periodStr} / ${roman_semester}`
}

</script>

<template>
  <h2>SKS Generator</h2>
  <div style="display: flex; align-items: center; column-gap: 8px;">
    <p>semester yang sedang berjalan</p>
    <input type="text" style="width: 24px;" v-model.number="runningSemester">
    <span>semester yang tersisa : {{ MAX_SEMESTER - runningSemester }}, maximal semester: {{
      MAX_SEMESTER }}</span>
  </div>

  <div style="display: flex; align-items: center; column-gap: 8px;">
    <p>SKS Kumulatif</p>
    <input type="text" style="width: 24px;" v-model.number="runningSKS">
    <span>semester yang tersisa : {{ MAX_SKS - runningSKS }}, maximal semester: {{
      MAX_SKS }}</span>
  </div>

  <div>
    <p>upload data pengulangan</p>
    <a style="margin-right: 8px;" :href="templateCSV" download>Download Template</a>

    <input type="file"
      accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel"
      @change="e => previewUploadRepeatFile(e.target.files)">

  </div>

  <table class="repeat-table">
    <thead>
      <tr>
        <th>
          lab / taken now
        </th>
        <th>Periode / Semester</th>
        <th>Kode mata kuliah</th>
        <th>Mata kuliah</th>
        <th>SKS</th>
      </tr>
    </thead>
    <tbody>
      <template v-for="[period, subjects] in Object.entries(failedSubject)">
        <tr v-for="subject, index in subjects">
          <td>
            <input type="checkbox" :checked="subject.is_lab" @change="e => {
              failedSubject[period][index].is_lab = e.target.checked
            }">
            <span style="margin: 0 .2rem;">/</span>
            <input type="checkbox" :checked="subject.is_taken_now" @change="e => {
              failedSubject[period][index].is_taken_now = e.target.checked
            }">
          </td>
          <td v-if="index === 0" :rowspan="subjects.length">{{ period }}
          </td>
          <td>{{ subject.kode_mata_kuliah }}</td>
          <td>{{ subject.nama_mata_kuliah }}</td>
          <td>{{ subject.sks }}</td>
        </tr>
      </template>

    </tbody>
  </table>

  <div style="margin-top: 1rem;">
    <button @click="generateFutureSubject">Hasilkan Rencana Pengambilan mata kuliah</button>

    <table ref="generateSKSTableRef" class="repeat-table" id="rencana-sks-table">
      <thead>
        <tr>
          <th>Periode / Semester</th>
          <th>Kode mata kuliah</th>
          <th>Mata kuliah</th>
          <th>SKS</th>
        </tr>
      </thead>
      <tbody>
        <template v-for="[period, subjects] in Object.entries(futureSubject)">
          <tr v-for="subject, index in subjects">
            <td v-if="index === 0" :rowspan="subjects.length">{{ period }}
            </td>
            <td>{{ subject.kode_mata_kuliah }}</td>
            <td>{{ subject.nama_mata_kuliah }}</td>
            <td>{{ subject.sks }}</td>
          </tr>
          <tr>
            <td colspan="3"></td>
            <td><b>{{subjects.reduce((acc, subject) => acc + subject.sks, 0)}}</b></td>
          </tr>
        </template>
      </tbody>
      <tfoot>
        <tr>
          <td colspan="3"></td>
          <td><b>{{Object.values(futureSubject).flat().reduce((acc, subject) => acc + subject.sks, 0)}}</b></td>
        </tr>
      </tfoot>
    </table>

    <button style="margin-top: 8px;" @click="downloadCSV">Download CSV</button>
  </div>

</template>


<style scoped>
.repeat-table {
  margin-top: 1rem;
  border-collapse: collapse;
  width: 100%;

  th,
  td {
    border: 1px solid black;
    padding: 8px;
    text-align: left;
  }
}
</style>