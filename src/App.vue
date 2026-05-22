<script setup>
import { computed, onBeforeUnmount, reactive, ref, watch } from "vue";

const tunings = [
  {
    name: "标准调弦",
    strings: [
      ["E2", 82.41],
      ["A2", 110],
      ["D3", 146.83],
      ["G3", 196],
      ["B3", 246.94],
      ["E4", 329.63],
    ],
  },
  {
    name: "降 D",
    strings: [
      ["D2", 73.42],
      ["A2", 110],
      ["D3", 146.83],
      ["G3", 196],
      ["B3", 246.94],
      ["E4", 329.63],
    ],
  },
  {
    name: "全降半音",
    strings: [
      ["Eb2", 77.78],
      ["Ab2", 103.83],
      ["Db3", 138.59],
      ["Gb3", 185],
      ["Bb3", 233.08],
      ["Eb4", 311.13],
    ],
  },
  {
    name: "全降一音",
    strings: [
      ["D2", 73.42],
      ["G2", 98],
      ["C3", 130.81],
      ["F3", 174.61],
      ["A3", 220],
      ["D4", 293.66],
    ],
  },
  {
    name: "开放 G",
    strings: [
      ["D2", 73.42],
      ["G2", 98],
      ["D3", 146.83],
      ["G3", 196],
      ["B3", 246.94],
      ["D4", 293.66],
    ],
  },
  {
    name: "DADGAD",
    strings: [
      ["D2", 73.42],
      ["A2", 110],
      ["D3", 146.83],
      ["G3", 196],
      ["A3", 220],
      ["D4", 293.66],
    ],
  },
];

const noteNames = [
  "C",
  "C#",
  "D",
  "Eb",
  "E",
  "F",
  "F#",
  "G",
  "Ab",
  "A",
  "Bb",
  "B",
];
const meterTicks = [-50, -25, 0, 25, 50];
const githubUrl = "https://github.com/linyanm/anything-tuner";
const minimumSignalRms = 0.0008;
const defaultNoiseFloor = 0.00035;
const comfortableSignalRms = 0.03;
const browserUserAgent = navigator.userAgent.toLowerCase();
const isSafari =
  browserUserAgent.includes("safari") &&
  !/(chrome|chromium|crios|fxios|edg|android)/.test(browserUserAgent);
const headstockLayout = [
  { side: "left", slot: 2, x: -104 },
  { side: "left", slot: 1, x: -104 },
  { side: "left", slot: 0, x: -104 },
  { side: "right", slot: 0, x: 104 },
  { side: "right", slot: 1, x: 104 },
  { side: "right", slot: 2, x: 104 },
];

const activeTuningIndex = ref(0);
const activeStringIndex = ref(0);
const statusLine = ref("选择调弦后点击开始，允许浏览器使用麦克风。");
const detectedFrequency = ref(null);
const inputLevel = ref(0);
const pitchConfidence = ref(0);
const isSupported = Boolean(navigator.mediaDevices?.getUserMedia);
const audioState = reactive({
  context: null,
  analyser: null,
  source: null,
  preamp: null,
  stream: null,
  buffer: null,
  rafId: null,
  lastAnalysisTime: 0,
  missedAnalyses: 0,
  noiseFloor: defaultNoiseFloor,
  running: false,
});

const activeTuning = computed(() => tunings[activeTuningIndex.value]);
const targetString = computed(
  () => activeTuning.value.strings[activeStringIndex.value],
);
const targetNote = computed(() => targetString.value[0]);
const targetFrequency = computed(() => targetString.value[1]);
const frequencyText = computed(() =>
  detectedFrequency.value
    ? `${detectedFrequency.value.toFixed(2)} Hz`
    : "-- Hz",
);
const targetFrequencyText = computed(
  () => `${targetFrequency.value.toFixed(2)} Hz`,
);
const inputLevelText = computed(() => `${Math.round(inputLevel.value * 100)}%`);
const inputLevelStyle = computed(() => ({
  transform: `scaleX(${inputLevel.value})`,
}));
const confidenceText = computed(() =>
  pitchConfidence.value ? `${Math.round(pitchConfidence.value * 100)}%` : "--",
);
const detectedNote = computed(() =>
  detectedFrequency.value ? frequencyToNote(detectedFrequency.value) : "--",
);
const cents = computed(() => {
  if (!detectedFrequency.value) return null;
  return 1200 * Math.log2(detectedFrequency.value / targetFrequency.value);
});
const centsText = computed(() => {
  if (cents.value === null)
    return audioState.running ? "请拨响琴弦" : "等待输入";
  const absCents = Math.abs(cents.value);
  if (absCents <= 5) return "已调准";
  return cents.value < 0
    ? `偏低 ${Math.round(absCents)} cents`
    : `偏高 ${Math.round(absCents)} cents`;
});
const tunerStateClass = computed(() => {
  if (cents.value === null) return "";
  if (Math.abs(cents.value) <= 5) return "is-good";
  return cents.value < 0 ? "is-flat" : "is-sharp";
});
const needleStyle = computed(() => {
  if (cents.value === null)
    return { transform: "translateX(-50%) rotate(0deg)" };
  const clampedCents = Math.max(-50, Math.min(50, cents.value));
  const rotation = (clampedCents / 50) * 42;
  return { transform: `translateX(-50%) rotate(${rotation}deg)` };
});

watch(activeTuningIndex, () => {
  activeStringIndex.value = 0;
  resetPitch();
});

watch(activeStringIndex, resetPitch);

if (!isSupported) {
  statusLine.value = "当前浏览器不支持麦克风输入，请换用现代浏览器。";
}

async function toggleTuner() {
  if (audioState.running) {
    stopTuner();
    return;
  }
  await startTuner();
}

async function startTuner() {
  try {
    audioState.stream = await navigator.mediaDevices.getUserMedia({
      audio: getAudioConstraints(),
    });
    const AudioContextClass = window.AudioContext || window.webkitAudioContext;
    audioState.context = new AudioContextClass();
    if (audioState.context.state === "suspended") {
      await audioState.context.resume();
    }
    audioState.analyser = audioState.context.createAnalyser();
    audioState.analyser.fftSize = 8192;
    audioState.analyser.smoothingTimeConstant = 0;
    audioState.buffer = new Float32Array(audioState.analyser.fftSize);

    audioState.source = audioState.context.createMediaStreamSource(
      audioState.stream,
    );
    audioState.preamp = audioState.context.createGain();
    audioState.preamp.gain.value = getInputGain();
    audioState.source.connect(audioState.preamp);
    audioState.preamp.connect(audioState.analyser);

    audioState.running = true;
    audioState.lastAnalysisTime = 0;
    audioState.missedAnalyses = 0;
    audioState.noiseFloor = defaultNoiseFloor;
    statusLine.value = "正在监听琴弦声音。";
    tick();
  } catch (error) {
    statusLine.value = "无法使用麦克风，请检查浏览器权限或 HTTPS 环境。";
  }
}

function stopTuner() {
  audioState.running = false;
  cancelAnimationFrame(audioState.rafId);
  audioState.stream?.getTracks().forEach((track) => track.stop());
  audioState.context?.close();
  audioState.stream = null;
  audioState.context = null;
  audioState.analyser = null;
  audioState.source = null;
  audioState.preamp = null;
  audioState.lastAnalysisTime = 0;
  audioState.missedAnalyses = 0;
  audioState.noiseFloor = defaultNoiseFloor;
  inputLevel.value = 0;
  statusLine.value = "已停止监听。";
  resetPitch();
}

function tick(timestamp = performance.now()) {
  if (!audioState.running || !audioState.analyser) return;
  if (timestamp - audioState.lastAnalysisTime >= 45) {
    audioState.lastAnalysisTime = timestamp;
    audioState.analyser.getFloatTimeDomainData(audioState.buffer);
    const pitch = detectPitch(
      audioState.buffer,
      audioState.context.sampleRate,
      targetFrequency.value,
    );
    inputLevel.value = pitch.level;
    pitchConfidence.value = pitch.confidence;
    detectedFrequency.value = smoothDetectedFrequency(pitch.frequency);
    updateStatusFromPitch();
  }
  audioState.rafId = requestAnimationFrame(tick);
}

function getAudioConstraints() {
  const supportedConstraints = navigator.mediaDevices.getSupportedConstraints();
  const constraints = {};

  if (supportedConstraints.echoCancellation)
    constraints.echoCancellation = false;
  if (supportedConstraints.noiseSuppression)
    constraints.noiseSuppression = false;
  if (supportedConstraints.autoGainControl) constraints.autoGainControl = true;
  if (supportedConstraints.channelCount) constraints.channelCount = 1;

  return constraints;
}

function getInputGain() {
  return isSafari ? 10 : 3;
}

function detectPitch(buffer, sampleRate, targetFrequencyValue) {
  const workingBuffer = new Float32Array(buffer.length);
  let mean = 0;
  for (const sample of buffer) {
    mean += sample;
  }
  mean /= buffer.length;

  let rms = 0;
  for (let index = 0; index < buffer.length; index += 1) {
    const centeredSample = buffer[index] - mean;
    workingBuffer[index] = centeredSample;
    rms += centeredSample * centeredSample;
  }
  rms = Math.sqrt(rms / buffer.length);
  const level = Math.min(1, rms / comfortableSignalRms);
  updateNoiseFloor(rms);

  const signalGate = Math.max(minimumSignalRms, audioState.noiseFloor * 2.2);
  if (rms < signalGate) return { frequency: null, confidence: 0, level };

  for (let index = 0; index < workingBuffer.length; index += 1) {
    workingBuffer[index] /= rms;
  }

  const minFrequency = 55;
  const maxFrequency = 430;
  const minimumCorrelation = getMinimumCorrelation(rms);
  const minLag = Math.floor(sampleRate / maxFrequency);
  const maxLag = Math.min(
    buffer.length - 1,
    Math.ceil(sampleRate / minFrequency),
  );
  const candidates = [];

  for (let lag = minLag; lag <= maxLag; lag += 1) {
    let correlation = 0;
    let energy = 0;
    const compareLength = buffer.length - lag;
    for (let index = 0; index < compareLength; index += 1) {
      const sampleA = workingBuffer[index];
      const sampleB = workingBuffer[index + lag];
      correlation += sampleA * sampleB;
      energy += sampleA * sampleA + sampleB * sampleB;
    }

    const normalizedCorrelation = energy > 0 ? (2 * correlation) / energy : 0;
    if (normalizedCorrelation < minimumCorrelation) continue;

    const previousLag = Math.max(minLag, lag - 1);
    const nextLag = Math.min(maxLag, lag + 1);
    let previousCorrelation = normalizedCorrelation;
    let nextCorrelation = normalizedCorrelation;

    if (previousLag !== lag) {
      previousCorrelation = normalizedCorrelationForLag(
        workingBuffer,
        previousLag,
      );
    }
    if (nextLag !== lag) {
      nextCorrelation = normalizedCorrelationForLag(workingBuffer, nextLag);
    }

    if (
      normalizedCorrelation >= previousCorrelation &&
      normalizedCorrelation >= nextCorrelation
    ) {
      const denominator =
        previousCorrelation - 2 * normalizedCorrelation + nextCorrelation;
      const shift =
        Math.abs(denominator) > 0.000001
          ? (previousCorrelation - nextCorrelation) / (2 * denominator)
          : 0;
      const refinedLag = lag + Math.max(-0.5, Math.min(0.5, shift));
      const frequency = sampleRate / refinedLag;
      candidates.push({
        frequency,
        confidence: normalizedCorrelation,
        centsFromTarget: Math.abs(
          1200 * Math.log2(frequency / targetFrequencyValue),
        ),
      });
    }
  }

  if (!candidates.length) return { frequency: null, confidence: 0, level };

  candidates.sort((candidateA, candidateB) => {
    const targetDifference =
      candidateA.centsFromTarget - candidateB.centsFromTarget;
    if (Math.abs(targetDifference) > 12) return targetDifference;
    return candidateB.confidence - candidateA.confidence;
  });

  const bestCandidate = candidates[0];
  return {
    frequency: bestCandidate.frequency,
    confidence: bestCandidate.confidence,
    level,
  };
}

function updateNoiseFloor(rms) {
  const currentFloor = audioState.noiseFloor || defaultNoiseFloor;
  if (rms < currentFloor * 1.8) {
    audioState.noiseFloor = currentFloor * 0.94 + rms * 0.06;
    return;
  }

  audioState.noiseFloor = Math.max(
    defaultNoiseFloor,
    Math.min(currentFloor * 1.01, currentFloor + 0.00002),
  );
}

function getMinimumCorrelation(rms) {
  if (rms < 0.004) return 0.42;
  if (rms < 0.008) return 0.48;
  if (rms < 0.016) return 0.56;
  return 0.62;
}

function smoothDetectedFrequency(frequency) {
  if (!frequency) {
    audioState.missedAnalyses += 1;
    return audioState.missedAnalyses <= 3 ? detectedFrequency.value : null;
  }

  audioState.missedAnalyses = 0;
  if (!detectedFrequency.value) return frequency;

  const distanceInCents = Math.abs(
    1200 * Math.log2(frequency / detectedFrequency.value),
  );
  if (distanceInCents > 40) return frequency;

  return detectedFrequency.value * 0.55 + frequency * 0.45;
}

function normalizedCorrelationForLag(buffer, lag) {
  let correlation = 0;
  let energy = 0;
  const compareLength = buffer.length - lag;
  for (let index = 0; index < compareLength; index += 1) {
    const sampleA = buffer[index];
    const sampleB = buffer[index + lag];
    correlation += sampleA * sampleB;
    energy += sampleA * sampleA + sampleB * sampleB;
  }
  return energy > 0 ? (2 * correlation) / energy : 0;
}

function updateStatusFromPitch() {
  if (cents.value === null) {
    statusLine.value =
      inputLevel.value > 0.04
        ? "已收到声音，但音高不稳定。请靠近麦克风并单独拨响当前琴弦。"
        : "请拨响琴弦。";
    return;
  }
  if (Math.abs(cents.value) <= 5) {
    statusLine.value = "音准稳定在目标附近。";
    return;
  }
  statusLine.value =
    cents.value < 0 ? "琴弦偏低，稍微拧紧。" : "琴弦偏高，稍微放松。";
}

function resetPitch() {
  detectedFrequency.value = null;
  pitchConfidence.value = 0;
  audioState.missedAnalyses = 0;
}

function frequencyToNote(frequency) {
  const noteNumber = Math.round(12 * Math.log2(frequency / 440) + 69);
  const octave = Math.floor(noteNumber / 12) - 1;
  return `${noteNames[noteNumber % 12]}${octave}`;
}

onBeforeUnmount(() => {
  if (audioState.running) stopTuner();
});
</script>

<template>
  <main class="app-shell">
    <section
      class="tuner-panel"
      :class="tunerStateClass"
      aria-label="guitar tuner"
    >
      <header class="app-header">
        <div class="brand-row">
          <p class="eyebrow">Anything Tuner</p>
          <a
            class="source-link"
            :href="githubUrl"
            target="_blank"
            rel="noreferrer"
            aria-label="打开 GitHub 项目"
            title="GitHub"
          >
            <svg viewBox="0 0 24 24" aria-hidden="true">
              <path
                d="M12 2C6.48 2 2 6.58 2 12.24c0 4.52 2.87 8.35 6.84 9.71.5.1.68-.22.68-.49v-1.74c-2.78.62-3.37-1.37-3.37-1.37-.45-1.19-1.11-1.5-1.11-1.5-.91-.64.07-.63.07-.63 1 .07 1.53 1.06 1.53 1.06.9 1.57 2.35 1.12 2.92.86.09-.67.35-1.12.63-1.38-2.22-.26-4.56-1.14-4.56-5.05 0-1.12.39-2.03 1.03-2.74-.1-.26-.45-1.3.1-2.7 0 0 .84-.28 2.75 1.05A9.34 9.34 0 0 1 12 6.99c.85 0 1.71.12 2.51.34 1.91-1.33 2.75-1.05 2.75-1.05.55 1.4.2 2.44.1 2.7.64.71 1.03 1.62 1.03 2.74 0 3.92-2.34 4.79-4.57 5.04.36.32.68.94.68 1.89v2.81c0 .27.18.59.69.49A10.2 10.2 0 0 0 22 12.24C22 6.58 17.52 2 12 2Z"
              />
            </svg>
          </a>
        </div>
        <div class="header-actions">
          <button
            class="icon-button"
            type="button"
            :aria-pressed="audioState.running"
            :disabled="!isSupported"
            @click="toggleTuner"
          >
            <span class="mic-icon" aria-hidden="true"></span>
            <span>{{ audioState.running ? "停止调音" : "开始调音" }}</span>
          </button>
        </div>
      </header>

      <div class="status-line" role="status">{{ statusLine }}</div>
      <div class="input-monitor" aria-label="麦克风输入电平">
        <span>输入 {{ inputLevelText }}</span>
        <div class="input-meter" aria-hidden="true">
          <span :style="inputLevelStyle"></span>
        </div>
        <span>置信度 {{ confidenceText }}</span>
      </div>

      <section class="headstock-preview" aria-label="琴头预览">
        <div class="headstock">
          <div class="headstock-cap"></div>
          <div class="headstock-neck"></div>
          <button
            v-for="([note, frequency], index) in activeTuning.strings"
            :key="`headstock-${note}-${frequency}`"
            class="headstock-string"
            :class="{
              active: index === activeStringIndex,
              left: headstockLayout[index].side === 'left',
              right: headstockLayout[index].side === 'right',
            }"
            type="button"
            :style="{
              '--string-index': index,
              '--peg-slot': headstockLayout[index].slot,
              '--peg-x': `${headstockLayout[index].x}px`,
            }"
            :aria-label="`选择第 ${activeTuning.strings.length - index} 弦 ${note}`"
            @click="activeStringIndex = index"
          >
            <span class="string-line"></span>
            <span class="tuning-peg">
              <span class="peg-label">{{ note }}</span>
              <span class="peg-number">
                {{ activeTuning.strings.length - index }}
              </span>
            </span>
          </button>
        </div>
      </section>

      <section class="readout" aria-live="polite">
        <div class="meter" aria-hidden="true">
          <div class="meter-status">{{ centsText }}</div>
          <div class="meter-arc"></div>
          <div class="meter-safe-zone"></div>
          <div class="meter-ticks">
            <span
              v-for="tick in meterTicks"
              :key="tick"
              :style="{ '--tick-angle': `${(tick / 50) * 50}deg` }"
            ></span>
          </div>
          <div class="meter-band">
            <span
              v-for="tick in meterTicks"
              :key="`label-${tick}`"
              :style="{ '--label-angle': `${(tick / 50) * 50}deg` }"
            >
              {{ tick > 0 ? `+${tick}` : tick }}
            </span>
          </div>
          <div class="meter-center">
            <span>目标 {{ targetNote }}</span>
            <strong>{{ detectedNote }}</strong>
          </div>
          <div class="needle" :style="needleStyle"></div>
          <div class="needle-pivot"></div>
        </div>
        <div class="note-card">
          <div class="target-note">{{ targetNote }}</div>
          <div class="detected-note">{{ detectedNote }}</div>
          <div class="cents">{{ centsText }}</div>
        </div>
        <div class="mobile-frequency-strip" aria-label="频率信息">
          <div>
            <span>当前</span>
            <strong>{{ frequencyText }}</strong>
          </div>
          <div>
            <span>目标</span>
            <strong>{{ targetFrequencyText }}</strong>
          </div>
        </div>
      </section>

      <section class="controls" aria-label="调音设置">
        <label class="select-label" for="tuningSelect">调弦</label>
        <select id="tuningSelect" v-model.number="activeTuningIndex">
          <option
            v-for="(tuning, index) in tunings"
            :key="tuning.name"
            :value="index"
          >
            {{ tuning.name }}
          </option>
        </select>

        <div class="string-list" aria-label="琴弦">
          <button
            v-for="([note, frequency], index) in activeTuning.strings"
            :key="`${note}-${frequency}`"
            class="string-button"
            :class="{ active: index === activeStringIndex }"
            type="button"
            @click="activeStringIndex = index"
          >
            <strong>{{ note }}</strong>
            <span>
              {{ activeTuning.strings.length - index }} 弦 ·
              {{ frequency.toFixed(2) }} Hz
            </span>
          </button>
        </div>
      </section>
    </section>

    <aside class="side-panel" aria-label="使用状态">
      <div class="frequency-card">
        <span>当前频率</span>
        <strong>{{ frequencyText }}</strong>
      </div>
      <div class="frequency-card">
        <span>目标频率</span>
        <strong>{{ targetFrequencyText }}</strong>
      </div>
      <div class="tips">
        <h2>常用调弦</h2>
        <p>
          内置标准、降 D、全降半音、全降一音、开放
          G、DADGAD，适合电吉他和木吉他。
        </p>
      </div>
    </aside>
  </main>
</template>
