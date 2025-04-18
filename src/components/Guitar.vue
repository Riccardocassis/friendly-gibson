<script setup>
import { ref, onMounted, computed } from 'vue'
import gibson from '../assets/gibson.png'
import hotelPath from '../assets/suono/canzoni/H_california_solo.mp3'
import panamaPath from '../assets/suono/canzoni/panama.mp3'

const corde = ref([
  { file: 'EE.mp3', colore: '#FF4136', attiva: false, tasto: 'q' },
  { file: 'A.mp3',  colore: '#FF851B', attiva: false, tasto: 'w' },
  { file: 'D.mp3',  colore: '#FFDC00', attiva: false, tasto: 'e' },
  { file: 'G.mp3',  colore: '#2ECC40', attiva: false, tasto: 'r' },
  { file: 'B.mp3',  colore: '#0074D9', attiva: false, tasto: 't' },
  { file: 'e.mp3',  colore: '#B10DC9', attiva: false, tasto: 'y' }
])

const posizioniPx = [290, 305, 319, 333, 346, 361]
const spessoriPx = [4, 3.5, 3, 2.5, 2, 1.8]
const context = new (window.AudioContext || window.webkitAudioContext)()
const sourcesAttivi = []

const hotelAudio = new Audio(hotelPath)
const panamaAudio = new Audio(panamaPath)

let inModalitaBrano = ref(false)
let branoAttuale = ref(null)
let branoPartito = ref(false)

const nomeModalita = computed(() => {
  if (branoAttuale.value === 'hotel') return 'Hotel California'
  if (branoAttuale.value === 'panama') return 'Panama'
  return ''
})

function resetBrani() {
  hotelAudio.pause()
  hotelAudio.currentTime = 0
  panamaAudio.pause()
  panamaAudio.currentTime = 0
  branoPartito.value = false
}

function playBranoCorrente() {
  resetBrani()
  if (branoAttuale.value === 'hotel') hotelAudio.play()
  if (branoAttuale.value === 'panama') panamaAudio.play()
  branoPartito.value = true
}

async function suona(corda, index) {
  corde.value[index].attiva = true
  setTimeout(() => (corde.value[index].attiva = false), 150)

  if (inModalitaBrano.value) {
    if (!branoPartito.value) playBranoCorrente()
    return
  }

  try {
    const response = await fetch(`/src/assets/Suoni/${corda.file}`)
    const buffer = await response.arrayBuffer()
    const audioBuffer = await context.decodeAudioData(buffer)

    const source = context.createBufferSource()
    const gainNode = context.createGain()
    gainNode.gain.setValueAtTime(1, context.currentTime)

    source.buffer = audioBuffer
    source.connect(gainNode).connect(context.destination)
    source.start()

    sourcesAttivi.push({ source, gain: gainNode })

    source.onended = () => {
      const i = sourcesAttivi.findIndex(s => s.source === source)
      if (i !== -1) sourcesAttivi.splice(i, 1)
    }
  } catch (error) {
    console.error('Errore audio:', error)
  }
}

function gestisciTastiera(e) {
  const key = e.key.toLowerCase()

  if (key === ' ') {
    if (inModalitaBrano.value) {
      resetBrani()
    } else {
      const now = context.currentTime
      sourcesAttivi.forEach(({ source, gain }) => {
        try {
          gain.gain.linearRampToValueAtTime(0, now + 0.5)
          source.stop(now + 0.5)
        } catch {}
      })
      sourcesAttivi.length = 0
    }
    return
  }

  if (key === 'h') {
    inModalitaBrano.value = true
    branoAttuale.value = 'hotel'
    resetBrani()
    return
  }

  if (key === 'p') {
    inModalitaBrano.value = true
    branoAttuale.value = 'panama'
    resetBrani()
    return
  }

  if (key === 'n') {
    inModalitaBrano.value = false
    branoAttuale.value = null
    resetBrani()
    return
  }

  const index = corde.value.findIndex(c => c.tasto === key)
  if (index !== -1) {
    suona(corde.value[index], index)
  }
}

onMounted(() => {
  window.addEventListener('keydown', gestisciTastiera)
})
</script>

<template>
  <div class="page">
    <p class="info">
      🎸 Premi <strong>Q W E R T Y</strong> per suonare – <strong>SPACE</strong> per fermare<br>
      🎵 Premi <strong>H</strong> per Hotel California, <strong>P</strong> per Panama – <strong>N</strong> per tornare alla modalità normale
    </p>
    <p v-if="inModalitaBrano" class="modalita-attiva">
      Modalità <strong>{{ nomeModalita }}</strong> attivata 🎶
    </p>

    <div class="contenitore">
      <div class="wrapper">
        <img :src="gibson" alt="Chitarra" />
        <div
          v-for="(corda, index) in corde"
          :key="index"
          class="corda"
          :class="{ attiva: corda.attiva }"
          :style="{
            left: posizioniPx[index] + 'px',
            width: spessoriPx[index] + 'px',
            top: '0px',
            height: '380px',
            '--corda-colore': corda.colore
          }"
          @click="suona(corda, index)"
        ></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  width: 100vw;
  height: 100vh;
  background: #111;
  color: white;
  font-family: sans-serif;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.info {
  padding: 10px 10px 4px;
  font-size: 1.1rem;
  text-align: center;
  margin-bottom: 0;
}

.modalita-attiva {
  margin: 0;
  padding: 6px 14px;
  background: #222;
  border-radius: 6px;
  font-size: 1rem;
  color: #f1f1f1;
  box-shadow: 0 0 5px #555;
  text-align: center;
}

.contenitore {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  flex: 1;
}

.wrapper {
  position: relative;
  width: 646px;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: -20px;
}

.wrapper img {
  width: 100%;
  display: block;
  user-select: none;
  pointer-events: none;
}

.corda {
  position: absolute;
  background-color: transparent;
  border-radius: 2px;
  cursor: pointer;
  transition: 0.15s;
  pointer-events: auto;
}

.corda.attiva {
  background-color: var(--corda-colore);
  box-shadow: 0 0 10px var(--corda-colore), 0 0 20px var(--corda-colore);
}
</style>
