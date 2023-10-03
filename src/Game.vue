<script setup lang="ts">
import { onUnmounted } from "vue";
import { getWordOfTheDay, allWords } from "./words";
import Keyboard from "./Keyboard.vue";
import { LetterState } from "./types";

// Get word of the day
const answer = getWordOfTheDay();

// Board state. Each tile is represented as { letter, state }
const board = $ref(
  Array.from({ length: 6 }, () =>
    Array.from({ length: 5 }, () => ({
      letter: "",
      state: LetterState.INITIAL,
    }))
  )
);

// Current active row.
let currentRowIndex = $ref(0);
const currentRow = $computed(() => board[currentRowIndex]);

// Feedback state: message and shake
let message = $ref("");
let grid = $ref("");
let shakeRowIndex = $ref(-1);
let success = $ref(false);

// Keep track of revealed letters for the virtual keyboard
const letterStates: Record<string, LetterState> = $ref({});

// Handle keyboard input.
let allowInput = true;

const onKeyup = (e: KeyboardEvent) => onKey(e.key);

window.addEventListener("keyup", onKeyup);

onUnmounted(() => {
  window.removeEventListener("keyup", onKeyup);
});

function onKey(key: string) {
  if (!allowInput) return;
  if (/^[a-zA-Z]$/.test(key)) {
    fillTile(key.toLowerCase());
  } else if (key === "Backspace") {
    clearTile();
  } else if (key === "Enter") {
    completeRow();
  }
}

function fillTile(letter: string) {
  for (const tile of currentRow) {
    if (!tile.letter) {
      tile.letter = letter;
      break;
    }
  }
}

function clearTile() {
  for (const tile of [...currentRow].reverse()) {
    if (tile.letter) {
      tile.letter = "";
      break;
    }
  }
}

function completeRow() {
  if (currentRow.every((tile) => tile.letter)) {
    const guess = currentRow.map((tile) => tile.letter).join("");
    if (!allWords.includes(guess) && guess !== answer) {
      shake();
      showMessage(`Not in word list`);
      return;
    }

    const answerLetters: (string | null)[] = answer.split("");
    // first pass: mark correct ones
    currentRow.forEach((tile, i) => {
      if (answerLetters[i] === tile.letter) {
        tile.state = letterStates[tile.letter] = LetterState.CORRECT;
        answerLetters[i] = null;
      }
    });
    // second pass: mark the present
    currentRow.forEach((tile) => {
      if (!tile.state && answerLetters.includes(tile.letter)) {
        tile.state = LetterState.PRESENT;
        answerLetters[answerLetters.indexOf(tile.letter)] = null;
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.PRESENT;
        }
      }
    });
    // 3rd pass: mark absent
    currentRow.forEach((tile) => {
      if (!tile.state) {
        tile.state = LetterState.ABSENT;
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.ABSENT;
        }
      }
    });

    allowInput = false;
    if (currentRow.every((tile) => tile.state === LetterState.CORRECT)) {
      // yay!
      setTimeout(() => {
        grid = genResultGrid();
        showMessage(
          ["Genius", "Magnificent", "Impressive", "Splendid", "Great", "Phew"][
            currentRowIndex
          ],
          -1
        );
        success = true;
      }, 1600);
    } else if (currentRowIndex < board.length - 1) {
      // go the next row
      currentRowIndex++;
      setTimeout(() => {
        allowInput = true;
      }, 1600);
    } else {
      // game over :(
      setTimeout(() => {
        showMessage(answer.toUpperCase(), -1);
      }, 1600);
    }
  } else {
    shake();
    showMessage("Not enough letters");
  }
}

function showMessage(msg: string, time = 1000) {
  message = msg;
  if (time > 0) {
    setTimeout(() => {
      message = "";
    }, time);
  }
}

function shake() {
  shakeRowIndex = currentRowIndex;
  setTimeout(() => {
    shakeRowIndex = -1;
  }, 1000);
}

const icons = {
  [LetterState.CORRECT]: "🟩",
  [LetterState.PRESENT]: "🟨",
  [LetterState.ABSENT]: "⬜",
  [LetterState.INITIAL]: null,
};

function genResultGrid() {
  return board
    .slice(0, currentRowIndex + 1)
    .map((row) => {
      return row.map((tile) => icons[tile.state]).join("");
    })
    .join("\n");
}
function handleNewWord() {
  const word = prompt("Enter a 5 letter word:");
  if (word.length === 5) {
    let url = btoa(word);
    console.log(url);
    alert(url);
    window.open(url, "_blank");
  } else {
    alert("Please ensure you word is 5 letters.");
  }
}
</script>

<template>
  <Transition>
    <div class="message" v-if="message">
      {{ message }}
      <pre v-if="grid">{{ grid }}</pre>
    </div>
  </Transition>
  <header>
    <h1>Custom Wordle</h1>
  </header>
  <div align="center">
    <button class="btn-91" @click="handleNewWord()">
      <span>Create Your Own Worldle</span>
    </button>
  </div>
  <div id="board">
    <div
      v-for="(row, index) in board"
      :class="[
        'row',
        shakeRowIndex === index && 'shake',
        success && currentRowIndex === index && 'jump',
      ]"
    >
      <div
        v-for="(tile, index) in row"
        :class="['tile', tile.letter && 'filled', tile.state && 'revealed']"
      >
        <div class="front" :style="{ transitionDelay: `${index * 300}ms` }">
          {{ tile.letter }}
        </div>
        <div
          :class="['back', tile.state]"
          :style="{
            transitionDelay: `${index * 300}ms`,
            animationDelay: `${index * 100}ms`,
          }"
        >
          {{ tile.letter }}
        </div>
      </div>
    </div>
  </div>
  <Keyboard @key="onKey" :letter-states="letterStates" />
</template>





<style scoped>
#board {
  display: grid;
  grid-template-rows: repeat(6, 1fr);
  grid-gap: 5px;
  padding: 10px;
  box-sizing: border-box;
  --height: min(420px, calc(var(--vh, 100vh) - 310px));
  height: var(--height);
  width: min(350px, calc(var(--height) / 6 * 5));
  margin: 0px auto;
}
.message {
  position: absolute;
  left: 50%;
  top: 80px;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.85);
  padding: 16px 20px;
  z-index: 2;
  border-radius: 4px;
  transform: translateX(-50%);
  transition: opacity 0.3s ease-out;
  font-weight: 600;
}
.message.v-leave-to {
  opacity: 0;
}
.row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  grid-gap: 5px;
}
.tile {
  width: 100%;
  font-size: 2rem;
  line-height: 2rem;
  font-weight: bold;
  vertical-align: middle;
  text-transform: uppercase;
  user-select: none;
  position: relative;
}
.tile.filled {
  animation: zoom 0.2s;
}
.tile .front,
.tile .back {
  box-sizing: border-box;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}
.tile .front {
  border: 2px solid #d3d6da;
}
.tile.filled .front {
  border-color: #999;
}
.tile .back {
  transform: rotateX(180deg);
}
.tile.revealed .front {
  transform: rotateX(180deg);
}
.tile.revealed .back {
  transform: rotateX(0deg);
}

@keyframes zoom {
  0% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
  }
}

.shake {
  animation: shake 0.5s;
}

@keyframes shake {
  0% {
    transform: translate(1px);
  }
  10% {
    transform: translate(-2px);
  }
  20% {
    transform: translate(2px);
  }
  30% {
    transform: translate(-2px);
  }
  40% {
    transform: translate(2px);
  }
  50% {
    transform: translate(-2px);
  }
  60% {
    transform: translate(2px);
  }
  70% {
    transform: translate(-2px);
  }
  80% {
    transform: translate(2px);
  }
  90% {
    transform: translate(-2px);
  }
  100% {
    transform: translate(1px);
  }
}

.jump .tile .back {
  animation: jump 0.5s;
}

@keyframes jump {
  0% {
    transform: translateY(0px);
  }
  20% {
    transform: translateY(5px);
  }
  60% {
    transform: translateY(-25px);
  }
  90% {
    transform: translateY(3px);
  }
  100% {
    transform: translateY(0px);
  }
}

@media (max-height: 680px) {
  .tile {
    font-size: 3vh;
  }
}
.btn-101,
.btn-101 *,
.btn-101 :after,
.btn-101 :before,
.btn-101:after,
.btn-101:before {
  border: 0 solid;
  box-sizing: border-box;
}
.btn-101 {
  -webkit-tap-highlight-color: transparent;
  -webkit-appearance: button;
  background-color: #000;
  background-image: none;
  color: #fff;
  font-family: ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont,
    Segoe UI, Roboto, Helvetica Neue, Arial, Noto Sans, sans-serif,
    Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji;
  font-size: 100%;
  font-weight: 900;
  line-height: 1.5;
  margin: 0;
  -webkit-mask-image: -webkit-radial-gradient(#000, #fff);
  padding: 0;
  text-transform: uppercase;
}
.btn-101:disabled {
  cursor: default;
}
.btn-101:-moz-focusring {
  outline: auto;
}
.btn-101 svg {
  vertical-align: middle;
}
.btn-101 [hidden] {
  display: none;
}
.btn-101 {
  --thickness: 0.3rem;
  --roundness: 1.2rem;
  --color: #eff6ff;
  --opacity: 0.6;
  -webkit-backdrop-filter: blur(100px);
  backdrop-filter: blur(100px);
  background: none;
  background: hsla(0, 0%, 100%, 0.2);
  border: none;
  border-radius: var(--roundness);
  color: var(--color);
  cursor: pointer;
  display: block;
  font-family: Poppins, "sans-serif";
  font-size: 1rem;
  font-weight: 500;
  padding: 0.8rem 3rem;
  position: relative;
}
.btn-101:hover {
  background: hsla(0, 0%, 100%, 0.3);
  filter: brightness(1.2);
}
.btn-101:active {
  --opacity: 0;
  background: hsla(0, 0%, 100%, 0.1);
}
.btn-101 svg {
  border-radius: var(--roundness);
  display: block;
  filter: url(#glow);
  height: 100%;
  left: 0;
  position: absolute;
  top: 0;
  width: 100%;
}
.btn-101 rect {
  fill: none;
  stroke: var(--color);
  stroke-width: var(--thickness);
  rx: var(--roundness);
  stroke-linejoin: round;
  stroke-dasharray: 185%;
  stroke-dashoffset: 80;
  -webkit-animation: snake 2s linear infinite;
  animation: snake 2s linear infinite;
  -webkit-animation-play-state: paused;
  animation-play-state: paused;
  height: 100%;
  opacity: 0;
  transition: opacity 0.2s;
  width: 100%;
}
.btn-101:hover rect {
  -webkit-animation-play-state: running;
  animation-play-state: running;
  opacity: var(--opacity);
}
@-webkit-keyframes snake {
  to {
    stroke-dashoffset: 370%;
  }
}
@keyframes snake {
  to {
    stroke-dashoffset: 370%;
  }
}
.btn-91,
.btn-91 *,
.btn-91 :after,
.btn-91 :before,
.btn-91:after,
.btn-91:before {
  border: 0 solid;
  box-sizing: border-box;
}
.btn-91 {
  -webkit-tap-highlight-color: transparent;
  -webkit-appearance: button;
  background-color: #000;
  background-image: none;
  color: #000;
  cursor: pointer;
  font-family: ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont,
    Segoe UI, Roboto, Helvetica Neue, Arial, Noto Sans, sans-serif,
    Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji;
  font-size: 100%;
  font-weight: 900;
  line-height: 1.5;
  margin: 0;
  -webkit-mask-image: -webkit-radial-gradient(#000, #fff);
  padding: 0;
}
.btn-91:disabled {
  cursor: default;
}
.btn-91:-moz-focusring {
  outline: auto;
}
.btn-91 svg {
  display: block;
  vertical-align: middle;
}
.btn-91 [hidden] {
  display: none;
}
.btn-91 {
  background: none;
  border: 2px solid;
  border-radius: 999px;
  box-sizing: border-box;
  display: block;
  padding: 1.2rem 3rem;
  position: relative;
  text-transform: uppercase;
}
.btn-91 span {
  font-weight: 900;
}
.btn-91:after,
.btn-91:before {
  border: 2px solid red;
  border-radius: 999px;
  content: "";
  inset: -2px;
  position: absolute;
  z-index: -1;
}
.btn-91:after {
  border-color: blue;
}
.btn-91:hover:before {
  -webkit-animation: glitch-border 0.2s infinite;
  animation: glitch-border 0.2s infinite;
}
.btn-91:hover:after {
  animation: glitch-border 0.2s infinite reverse;
}
.btn-91:hover span {
  -webkit-animation: glitch-text 0.2s infinite;
  animation: glitch-text 0.2s infinite;
}
@-webkit-keyframes glitch-text {
  0% {
    text-shadow: 255 255 255 red, 255 255 255 blue;
  }
  to {
    text-shadow: -2px 1px 0 red, 2px -1px 0 blue;
  }
}
@keyframes glitch-text {
  0% {
    text-shadow: 255 255 255 red, 255 255 255 blue;
  }
  to {
    text-shadow: -2px 1px 0 red, 2px -1px 0 blue;
  }
}
@-webkit-keyframes glitch-border {
  0% {
    transform: translate(2px, -1px);
  }
  to {
    transform: translate(-2px, -1px);
  }
}
@keyframes glitch-border {
  0% {
    transform: translate(2px, -1px);
  }
  to {
    transform: translate(-2px, -1px);
  }
}
</style>
