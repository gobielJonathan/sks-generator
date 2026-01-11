<script setup lang="ts">
import { nextTick, ref, watchEffect } from 'vue';
import * as XLSX from 'xlsx'

type Subject = {
  kode_mata_kuliah: string,
  nama_mata_kuliah: string,
  sks: number
  is_lab: boolean,
  is_taken_now: boolean
}

type PeriodSubject = {
  [period: string]: Subject[]
}

const FAILED_SUBJECT_STORAGE_KEY = 'list-subject';
const FAILED_SKS_STORAGE_KEY = 'sks-subject';
const FAILED_SEMESTER_STORAGE_KEY = 'semester-subject';


const ROMAN_REGEX = /\b[IVXLCDM]+\b/i
const MAX_SEMESTER = 14;
const MAX_SKS = 146;
const MAX_SKS_PER_SEMESTER = 24;
const GENAP = 0
const GANJIL = 1
const MIN_SKS_TAKE_ENRICHMENT = 60
const SKS_ENRICHMENT = 20
const ENRICHMENT_SUBJECT_CODE = 'Enrichment'

const alreadyEnrichmentFirst = ref(false);
const alreadyEnrichmentSecond = ref(false);
const alreadyThesis = ref(false);

const THESIS_SUBJECT_CODE = 'Thesis'
const SKS_THESIS = 6

const runningSemester = ref(Number(localStorage.getItem(FAILED_SEMESTER_STORAGE_KEY) || '1'));
const runningSKS = ref(Number(localStorage.getItem(FAILED_SKS_STORAGE_KEY) || '1'));

const generateSKSTableRef = ref<HTMLElement | null>(null);
const generateListSKSTableRef = ref<HTMLElement | null>(null);
const groupFailedSubject = ref<PeriodSubject>({});
const failedSubjects = ref<(Subject & { period: string })[]>(
  safeParseJSON(localStorage.getItem(FAILED_SUBJECT_STORAGE_KEY) || '[]')
)
const futureSubject = ref<PeriodSubject>({});

watchEffect(() => {
  localStorage.setItem(FAILED_SUBJECT_STORAGE_KEY, JSON.stringify(failedSubjects.value));
});

watchEffect(() => {
  localStorage.setItem(FAILED_SKS_STORAGE_KEY, String(runningSKS.value));
});
watchEffect(() => {
  localStorage.setItem(FAILED_SEMESTER_STORAGE_KEY, String(runningSemester.value));
});

function safeParseJSON(jsonString: string) {
  try {
    return JSON.parse(jsonString)
  } catch (e) {
    return null
  }
}

async function downloadCSV() {
  await nextTick() // ensure DOM is rendered
  const ws = XLSX.utils.table_to_sheet(generateListSKSTableRef.value)
  const wb = XLSX.utils.book_new()

  XLSX.utils.book_append_sheet(wb, ws, 'sks-plan')
  XLSX.utils.sheet_add_dom(ws, generateSKSTableRef.value, { origin: -1 })
  XLSX.writeFile(wb, 'sks-plan.xlsx')
}

const generateFutureSubject = () => {

  groupFailedSubject.value = failedSubjects.value.reduce((acc, curr) => {
    acc[curr.period] ??= []
    const _curr = {
      ...curr,
      kode_mata_kuliah: curr.kode_mata_kuliah.toLowerCase().includes("enrichment") ? ENRICHMENT_SUBJECT_CODE : curr.kode_mata_kuliah,
    }
    acc[curr.period].push(_curr)
    return acc
  }, {})

  const subjectPerSemester: Record<number, Subject[]> = { [GENAP]: [], [GANJIL]: [] }

  //seperate all subjects by ganjil / genap semester
  for (const [period, subjects] of Object.entries(groupFailedSubject.value)) {
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
  } else {
    if (alreadyEnrichmentFirst.value) {
      _countEnrichmentTaken++
    }
    if (alreadyEnrichmentSecond.value) {
      _countEnrichmentTaken++
    }
    if (alreadyThesis.value) {
      _isAlreadyGetThesis = true
    }
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

  while (_runningSemester <= MAX_SEMESTER && _runningSKS < MAX_SKS) {
    if (subjectPerSemester[GENAP]?.length === 0 && subjectPerSemester[GANJIL]?.length === 0) {
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
  console.log('_futureSubject', _futureSubject);
  futureSubject.value = _futureSubject

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

  <div style="display: flex; align-items: center; column-gap: 8px;">
    <input type="checkbox" v-model="alreadyEnrichmentFirst" name="" id="enrich-1"> <label for="enrich-1">Encrichment
      1</label>
    <input type="checkbox" v-model="alreadyEnrichmentSecond" name="" id="enrich-2"> <label for="enrich-2">Encrichment
      2</label>
    <input type="checkbox" v-model="alreadyThesis" name="" id="thesis"> <label for="thesis">Thesis</label>
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
        <th></th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="subject, index in failedSubjects" :key="index">
        <td>
          <input type="checkbox" :checked="subject.is_lab" @change="e => {

          }">
          <span style="margin: 0 .2rem;">/</span>
          <input type="checkbox" :checked="subject.is_taken_now" @change="e => {
          }">
        </td>
        <td>
          <input type="text" v-model="subject.period">
        </td>
        <td>
          <input type="text" v-model="subject.kode_mata_kuliah">
        </td>
        <td>
          <input type="text" v-model="subject.nama_mata_kuliah">
        </td>
        <td>
          <input type="text" v-model.number="subject.sks">
        </td>
        <td>
          <button @click="() => {
            failedSubjects = failedSubjects.filter((_, idx) => index !== idx)
          }" class="btn">X</button>
        </td>
      </tr>

    </tbody>

    <tfoot>
      <tr>
        <td colspan="6">
          <button @click="failedSubjects.push({})" class="add-button btn">tambahkan data</button>
        </td>
      </tr>
    </tfoot>
  </table>

  <div style="margin-top: 1rem;">
    <button @click="generateFutureSubject">Hasilkan Rencana Pengambilan mata kuliah</button>

    <table ref="generateListSKSTableRef" class="repeat-table">
      <thead>
        <tr>
          <th>Periode / Semester</th>
          <th>Kode mata kuliah</th>
          <th>Mata kuliah</th>
          <th>SKS</th>
        </tr>
      </thead>
      <tbody>
        <template v-for="[period, subjects] in Object.entries(groupFailedSubject)">
          <tr v-for="subject, index in subjects">
            <td v-if="index === 0" :rowspan="subjects.length">{{ period }}
            </td>
            <td>{{ subject.kode_mata_kuliah }}</td>
            <td>{{ subject.nama_mata_kuliah }}</td>
            <td>{{ subject.sks }}</td>
          </tr>
        </template>
      </tbody>
    </table>

    <p>Rencana pengambilan mata kuliah</p>

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
          <tr v-if="subjects.length > 0">
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
.btn {
  cursor: pointer;
  width: 100%;
  border: none;
}

.add-button {

  padding: .5rem 0;
}

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