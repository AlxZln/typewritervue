<template>
  <div>
    <div class="app-page">
      <main :class="['workspace', `screen-${screen}`]">
        <div v-if="screen !== 'result'" :class="['settings-wrap', { 'is-running': running }]">
          <nav class="settings" aria-label="Настройки теста">
            <div class="mode-options">
              <q-btn flat no-caps :class="{ selected: mode === 'time' }" :aria-pressed="mode === 'time'"
                @click="changeMode('time')" icon="timer">Время</q-btn>
              <q-btn flat no-caps :class="{ selected: mode === 'words' }" :aria-pressed="mode === 'words'"
                @click="changeMode('words')" icon="title">Слова</q-btn>
            </div>
            <span class="divider" />
            <div class="amount-options">
              <q-btn v-for="value in presets" :key="value" flat no-caps :class="{ selected: amount === value }"
                :aria-pressed="amount === value" :aria-label="`${value} ${amountLabel}`" @click="setAmount(value)">{{
                  value }}</q-btn>
              <q-btn flat class="custom-button" aria-label="Своё значение"
                :class="{ selected: !presets.includes(amount) }" @click="openCustom" icon="tune">
                <q-tooltip>Своё значение · слова или символы</q-tooltip></q-btn>
            </div>
          </nav>
          <span v-if="unit === 'characters' && mode === 'words'" class="unit-caption">количество символов</span>
        </div>

        <section v-if="screen === 'start'" class="start-actions">
          <q-btn unelevated no-caps class="button-primary start-button" @click="start()">Начать тест</q-btn>
        </section>

        <section v-if="screen === 'test'" class="test" aria-label="Тест скорости печати">
          <div class="progress" aria-live="off">{{ progress }}</div>
          <div :class="['typing-area', { unfocused: !focused }]" @click="focusInput">
            <div ref="wordsViewport" class="words-viewport" aria-hidden="true">
              <div class="words">
                <span v-for="(word, wordIndex) in displayWords" :key="wordIndex"
                  :class="['word', { 'word-error': word.hasError }]">
                  <span v-if="word.extraSpaces" class="extra">{{ word.extraSpaces }}</span>
                  <span v-for="(letter, letterIndex) in word.letters" :key="letterIndex"
                    :class="['letter', letter.status, { caret: letter.hasCaret }]">{{ letter.character }}</span>
                  <span v-for="(letter, extraIndex) in word.extraText" :key="`extra-${extraIndex}`"
                    class="letter extra">{{ letter }}</span>
                  <span v-if="word.hasEndCaret" class="caret end-caret" />
                </span>
              </div>
            </div>
            <textarea ref="typingInput" class="typing-input" aria-label="Введите текст теста" :value="session.typedText"
              autocomplete="off" autocapitalize="off" :spellcheck="false" @input="onInput" @focus="focused = true"
              @blur="focused = false" @paste.prevent @drop.prevent @keydown.enter.prevent @keydown.end.prevent
              @keydown.home.prevent @keydown.left.prevent @keydown.right.prevent @keydown.up.prevent
              @keydown.down.prevent @keydown.esc.prevent="typingInput.blur()" @select="
                typingInput.setSelectionRange(
                  session.typedText.length,
                  session.typedText.length,
                )
                " />
            <button v-if="!focused" class="focus-prompt" @click="focusInput">
              Нажмите здесь, чтобы продолжить
            </button>
          </div>
          <div class="test-actions">
            <q-btn unelevated no-caps class="button-secondary" @click="start(false)">Заново</q-btn>
          </div>
          <p v-if="session.startedAt === null" class="typing-hint">
            Начните печатать, чтобы запустить тест
          </p>
        </section>

        <section v-if="screen === 'result'" class="results" aria-label="Результаты теста" aria-live="polite">
          <dl class="stats">
            <div>
              <dt>тип теста</dt>
              <dd>
                {{ testTypeLabel }}
                {{ amount }}
              </dd>
            </div>
            <div>
              <dt><abbr title="Символов в минуту">cpm</abbr></dt>
              <dd>{{ result.cpm }}</dd>
            </div>
            <div>
              <dt>точность</dt>
              <dd>{{ result.accuracy }}%</dd>
            </div>
            <div>
              <dt>символы</dt>
              <dd>{{ result.characters }}</dd>
            </div>
            <div>
              <dt>ошибки</dt>
              <dd>{{ result.errors }}</dd>
            </div>
          </dl>
          <div class="result-actions">
            <q-btn unelevated no-caps class="button-primary" @click="start()">Заново</q-btn>
          </div>
          <q-btn flat no-caps class="change-settings" @click="screen = 'start'">Изменить настройки</q-btn>
        </section>
      </main>
    </div>
  </div>

  <q-dialog v-model="customOpen" @hide="screen === 'test' && focusInput()">
    <q-card class="custom-dialog">
      <form @submit.prevent="applyCustom">
        <h1>Настройки теста</h1>
        <label v-if="mode === 'words'" class="custom-label" for="count-unit">Объём текста</label>
        <select v-if="mode === 'words'" id="count-unit" v-model="customUnit">
          <option value="words">Количество слов</option>
          <option value="characters">Количество символов</option>
        </select>
        <label class="custom-label" for="custom-amount">{{ customAmountLabel }}</label>
        <input id="custom-amount" v-model.number="customValue" type="number" min="1" :max="customLimit" autofocus
          required />
        <p class="custom-help">
          От 1 до {{ customLimit }}<template v-if="mode === 'time'"> секунд</template>
        </p>
        <div class="dialog-actions">
          <q-btn flat no-caps label="Отмена" @click="customOpen = false" /><q-btn unelevated no-caps type="submit"
            class="button-primary" label="Применить" :disable="!customValid" />
        </div>
      </form>
    </q-card>
  </q-dialog>

</template>


<script setup>
import { computed, nextTick, onBeforeUnmount, reactive, ref } from 'vue'

const passages = [
  'Задача организации, в особенности же рамки и место обучения кадров в значительной степени обуславливает создание направлений прогрессивного развития. Таким образом постоянное информационное обеспечение нашей деятельности играет важную роль в формировании новых предложений. С другой стороны постоянный количественный рост позволяет оценить значение дальнейшей работы.',
  'Раннее утро наполняет комнату мягким светом. За окном просыпается город, а на столе уже остывает чашка чая. В такие минуты особенно приятно открыть книгу и провести немного времени в тишине. Каждый новый день приносит маленькие открытия и возможность попробовать что-нибудь интересное.',
  'Дорога уходила далеко за горизонт. По обе стороны тянулись зелёные поля, и лёгкий ветер шевелил высокую траву. Мы решили идти дальше, чтобы увидеть озеро среди старых деревьев. Иногда самое удивительное путешествие начинается с простого желания выйти из дома.',
  'Умение быстро печатать приходит с практикой. Не стоит торопиться: сначала найдите удобный ритм и старайтесь попадать по нужным клавишам. Постепенно движения станут увереннее, а ошибок будет меньше. Главное — заниматься регулярно и получать удовольствие от собственного прогресса.',
  'Вечером небольшой книжный магазин превращается в особенное место. Тёплые лампы освещают полки, тихо шелестят страницы, и каждый посетитель выбирает свою историю. Здесь можно найти рассказ о далёких странах, научиться новому делу или просто отдохнуть от повседневной суеты.',
  'Хорошие идеи часто появляются неожиданно. Во время прогулки, разговора с другом или работы над обычной задачей мы замечаем то, что раньше ускользало от внимания. Записывайте мысли, задавайте вопросы и оставляйте место для экспериментов. Любопытство помогает увидеть знакомый мир по-новому.',
]

const mode = ref('words')
const unit = ref('words')
const durationSeconds = ref(60)
const wordCount = ref(50)
const characterCount = ref(250)
const screen = ref('start')
const typingInput = ref(null)
const wordsViewport = ref(null)
const focused = ref(false)
const customOpen = ref(false)
const customValue = ref(50)
const customUnit = ref('words')

const session = reactive({
  words: [],
  passageIndex: -1,
  typedText: '',
  startedAt: null,
  elapsedSeconds: 0,
  characterAttemptCount: 0,
  correctCharacterAttemptCount: 0,
})
let timer = null

const amount = computed(() => {
  if (mode.value === 'time') return durationSeconds.value
  if (unit.value === 'characters') return characterCount.value
  return wordCount.value
})

const presets = computed(() => {
  if (mode.value === 'time') return [15, 30, 60, 120]
  if (unit.value === 'characters') return [100, 250, 500, 1000]
  return [10, 25, 50, 100]
})

const amountLabel = computed(() => {
  if (mode.value === 'time') return 'секунд'
  if (unit.value === 'characters') return 'символов'
  return 'слов'
})

const testTypeLabel = computed(() => {
  if (mode.value === 'time') return 'время'
  if (unit.value === 'characters') return 'символы'
  return 'слова'
})

const customAmountLabel = computed(() => {
  if (mode.value === 'time') return 'Время в секундах'
  if (customUnit.value === 'characters') return 'Количество символов'
  return 'Количество слов'
})

const customLimit = computed(() => {
  if (mode.value === 'time') return 3600
  return 5000
})

const customValid = computed(() => {
  const customAmount = Number(customValue.value)
  return Number.isInteger(customAmount) && customAmount >= 1 && customAmount <= customLimit.value
})

const running = computed(
  () => session.startedAt !== null && screen.value === 'test'
)

const inputEvaluation = computed(() => evaluateInput(session.typedText, session.words))

const displayWords = computed(() => {
  return inputEvaluation.value.words.map((word, wordIndex) => {
    const isActive = wordIndex === inputEvaluation.value.activeWordIndex
    const letters = []
    let hasError =
      word.extraSpaceCount > 0 ||
      (word.submitted && word.typedText !== word.expectedText)

    for (let letterIndex = 0; letterIndex < word.expectedText.length; letterIndex++) {
      let letterStatus = ''
      if (letterIndex < word.typedText.length) {
        letterStatus = 'incorrect'
        if (word.typedText[letterIndex] === word.expectedText[letterIndex]) {
          letterStatus = 'correct'
        }
      } else if (word.submitted) {
        letterStatus = 'incorrect'
      }

      if (letterStatus === 'incorrect') hasError = true
      letters.push({
        character: word.expectedText[letterIndex],
        status: letterStatus,
        hasCaret: isActive && word.typedText.length === letterIndex,
      })
    }

    const extraText = word.typedText.slice(word.expectedText.length)
    if (extraText) hasError = true

    return {
      letters,
      extraText,
      extraSpaces: '␣'.repeat(word.extraSpaceCount),
      hasError,
      hasEndCaret: isActive && word.typedText.length >= word.expectedText.length,
    }
  })
})

const progress = computed(() => {
  if (mode.value === 'time') {
    return Math.max(0, Math.ceil(durationSeconds.value - session.elapsedSeconds))
  }

  const { words, activeWordIndex, complete } = inputEvaluation.value
  if (unit.value === 'characters') {
    let completedCharacterCount = 0
    for (let wordIndex = 0; wordIndex < activeWordIndex; wordIndex++) {
      completedCharacterCount += words[wordIndex].expectedText.length + 1
    }
    const activeWord = words[activeWordIndex]
    if (activeWord) {
      completedCharacterCount += Math.min(
        activeWord.typedText.length,
        activeWord.expectedText.length,
      )
    }
    return `${completedCharacterCount} / ${characterCount.value}`
  }

  let completedWordCount = activeWordIndex
  if (complete) completedWordCount++
  return `${completedWordCount} / ${session.words.length}`
})

const result = computed(() => {
  let accuracy = 0
  if (session.characterAttemptCount > 0) {
    accuracy = Math.round(
      (session.correctCharacterAttemptCount / session.characterAttemptCount) * 100,
    )
  }

  return {
    cpm: Math.round(
      (inputEvaluation.value.correctCharacterCount / Math.max(session.elapsedSeconds, 1)) * 60,
    ),
    accuracy,
    characters: session.typedText.length,
    errors: inputEvaluation.value.errorCount,
  }
})

function generateWords(targetCount, textUnit, passageIndex) {
  let passageText = passages[passageIndex]
  let nextPassageIndex = passageIndex
  let generatedCount = passageText.split(' ').length
  if (textUnit === 'characters') generatedCount = passageText.length

  while (generatedCount < targetCount) {
    nextPassageIndex = (nextPassageIndex + 1) % passages.length
    const nextPassage = passages[nextPassageIndex]
    passageText += ' ' + nextPassage
    if (textUnit === 'characters') {
      generatedCount = passageText.length
    } else {
      generatedCount += nextPassage.split(' ').length
    }
  }

  if (textUnit === 'characters') {
    // Сохраняем точное число символов без пустого последнего слова.
    return passageText.slice(0, targetCount).replace(/ $/, '.').split(' ')
  }
  return passageText.split(' ').slice(0, targetCount)
}

function evaluateInput(typedText, expectedWords) {
  console.log(typedText)
  const words = expectedWords.map((expectedText) => ({
    expectedText,
    typedText: '',
    extraSpaceCount: 0,
    submitted: false,
  }))
  let activeWordIndex = 0
  let correctCharacterCount = 0
  let errorCount = 0

  for (const character of typedText) {
    const word = words[activeWordIndex]
    if (!word) break

    if (character !== ' ') {
      const expectedCharacter = word.expectedText[word.typedText.length]
      if (character === expectedCharacter) {
        correctCharacterCount++
      } else {
        errorCount++
      }
      word.typedText += character
      continue
    }

    if (!word.typedText || activeWordIndex === words.length - 1) {
      word.extraSpaceCount++
      errorCount++
      continue
    }

    const skippedCharacterCount = Math.max(
      0,
      word.expectedText.length - word.typedText.length,
    )
    errorCount += skippedCharacterCount
    word.submitted = true
    correctCharacterCount++
    activeWordIndex++
  }

  const activeWord = words[activeWordIndex]
  const complete =
    words.length > 0 &&
    activeWordIndex === words.length - 1 &&
    activeWord.typedText.length >= activeWord.expectedText.length

  return { words, activeWordIndex, correctCharacterCount, errorCount, complete }
}

function setAmount(selectedAmount) {
  if (mode.value === 'time') durationSeconds.value = selectedAmount
  else if (unit.value === 'characters') characterCount.value = selectedAmount
  else wordCount.value = selectedAmount

  if (screen.value === 'test') start()
}

function changeMode(selectedMode) {
  mode.value = selectedMode
  if (screen.value === 'test') start()
}

function openCustom() {
  customValue.value = amount.value
  customUnit.value = unit.value
  customOpen.value = true
}

function applyCustom() {
  if (!customValid.value) return
  unit.value = customUnit.value
  setAmount(Number(customValue.value))
  customOpen.value = false
}

async function focusInput() {
  await nextTick()
  typingInput.value?.focus({ preventScroll: true })
  typingInput.value?.setSelectionRange(session.typedText.length, session.typedText.length)
}

function stopTimer() {
  clearInterval(timer)
  timer = null
}

async function start(sameText = false) {
  stopTimer()

  if (!sameText || session.words.length === 0) {
    const passageOffset = 1 + Math.floor(Math.random() * (passages.length - 1))
    session.passageIndex = (session.passageIndex + passageOffset) % passages.length
    let targetCount = amount.value
    let textUnit = unit.value
    if (mode.value === 'time') {
      targetCount = 180
      textUnit = 'words'
    }
    session.words = generateWords(targetCount, textUnit, session.passageIndex)
  }

  session.typedText = ''
  session.startedAt = null
  session.elapsedSeconds = 0
  session.characterAttemptCount = 0
  session.correctCharacterAttemptCount = 0
  screen.value = 'test'

  await nextTick()
  if (wordsViewport.value) wordsViewport.value.scrollTop = 0
  await focusInput()
}

function updateTimer() {
  if (session.startedAt === null || screen.value !== 'test') return
  session.elapsedSeconds = (Date.now() - session.startedAt) / 1000

  if (mode.value === 'time' && session.elapsedSeconds >= durationSeconds.value) {
    session.elapsedSeconds = durationSeconds.value
    stopTimer()
    screen.value = 'result'
  }
}

async function onInput(event) {
  updateTimer()
  if (screen.value !== 'test') return

  const typedText = event.target.value.replace(/[\n\r\t]/g, '')
  if (typedText && session.startedAt === null) {
    session.startedAt = Date.now()
    timer = setInterval(updateTimer, 100)
  }

  // При удалении символов статистика попыток не меняется.
  if (typedText.length > session.typedText.length) {
    const previousEvaluation = inputEvaluation.value
    const currentEvaluation = evaluateInput(typedText, session.words)
    session.characterAttemptCount++
    if (currentEvaluation.correctCharacterCount > previousEvaluation.correctCharacterCount) {
      session.correctCharacterAttemptCount++
    }
    if (currentEvaluation.activeWordIndex > previousEvaluation.activeWordIndex) {
      const submittedWord = previousEvaluation.words[previousEvaluation.activeWordIndex]
      session.characterAttemptCount += Math.max(
        0,
        submittedWord.expectedText.length - submittedWord.typedText.length,
      )
    }
  }

  session.typedText = typedText
  event.target.value = typedText

  if (mode.value === 'words' && inputEvaluation.value.complete) {
    updateTimer()
    stopTimer()
    screen.value = 'result'
  } else if (
    mode.value === 'time' &&
    inputEvaluation.value.activeWordIndex > session.words.length - 30
  ) {
    session.words = generateWords(session.words.length + 180, 'words', session.passageIndex)
  }

  await nextTick()
  const caret = wordsViewport.value?.querySelector('.caret')
  if (!caret || !wordsViewport.value) return

  const lineHeight = parseFloat(getComputedStyle(caret.closest('.words')).lineHeight)
  const caretTop = caret.getBoundingClientRect().top - wordsViewport.value.getBoundingClientRect().top
  if (caretTop < 0 || caretTop > wordsViewport.value.clientHeight - lineHeight * 2) {
    wordsViewport.value.scrollTop = Math.max(0, wordsViewport.value.scrollTop + caretTop - lineHeight)
  }
}

onBeforeUnmount(stopTimer)
</script>
