<script setup lang="ts">
import { computed, ref } from "vue";

type DuplexMode = "simplex" | "duplex";
type FlipMode = "long" | "short";

const file = ref<File | null>(null);
const isDragging = ref(false);
const pages = ref(8);
const copies = ref(1);
const duplex = ref<DuplexMode>("duplex");
const flip = ref<FlipMode>("long");

const isDuplex = computed(() => duplex.value === "duplex");
const sheetsPerCopy = computed(() => (isDuplex.value ? Math.ceil(pages.value / 2) : pages.value));
const totalSheets = computed(() => sheetsPerCopy.value * copies.value);
const totalSides = computed(() => pages.value * copies.value);
const maxRenderedSheets = 24;
const renderedSheets = computed(() => Math.min(totalSheets.value, maxRenderedSheets));
const hiddenSheets = computed(() => Math.max(totalSheets.value - renderedSheets.value, 0));
const sheetIndices = computed(() => Array.from({ length: renderedSheets.value }, (_, i) => i));

const sheetStyle = (index: number) => ({
  transform: `translateZ(${index * 1.2}px) translateY(${-index * 0.8}px) rotateX(${
    6 + index * 0.15
  }deg)`,
  zIndex: 200 - index,
});

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  file.value = target.files?.[0] ?? null;
};

const handleDrop = (event: DragEvent) => {
  event.preventDefault();
  isDragging.value = false;
  const dropped = event.dataTransfer?.files?.[0];
  if (dropped) file.value = dropped;
};

const handleDragOver = (event: DragEvent) => {
  event.preventDefault();
  isDragging.value = true;
};

const handleDragLeave = () => {
  isDragging.value = false;
};

const clearFile = () => {
  file.value = null;
};
</script>

<template>
  <div class="app">
    <header class="hero">
      <div>
        <p class="eyebrow">Print Preview Studio</p>
        <h1>Visualize printed pages before you hit print.</h1>
        <p class="subhead">
          Upload a document, choose duplex or simplex settings, and preview a 3D stack of your
          finished output.
        </p>
      </div>
      <div class="summary">
        <div>
          <span class="label">Total sheets</span>
          <strong>{{ totalSheets }}</strong>
        </div>
        <div>
          <span class="label">Total sides</span>
          <strong>{{ totalSides }}</strong>
        </div>
        <div>
          <span class="label">Mode</span>
          <strong>{{ isDuplex ? "Duplex" : "Simplex" }}</strong>
        </div>
      </div>
    </header>

    <main class="layout">
      <section class="panel upload" aria-label="Upload">
        <h2>Upload</h2>
        <p class="muted">PDF or image files work best.</p>
        <label
          class="dropzone"
          :class="{ dragging: isDragging }"
          @dragover="handleDragOver"
          @dragleave="handleDragLeave"
          @drop="handleDrop"
        >
          <input type="file" accept="application/pdf,image/*" @change="handleFileChange" />
          <div class="dropzone-content">
            <span class="badge">Drag & drop</span>
            <p v-if="!file">Drop a file here or click to browse.</p>
            <div v-else class="file-card">
              <div>
                <strong>{{ file.name }}</strong>
                <p>{{ (file.size / 1024 / 1024).toFixed(2) }} MB</p>
              </div>
              <button type="button" class="ghost" @click="clearFile">Remove</button>
            </div>
          </div>
        </label>
      </section>

      <section class="panel settings" aria-label="Print settings">
        <h2>Print settings</h2>
        <div class="field">
          <label for="pages">Pages per copy</label>
          <input id="pages" type="number" min="1" max="500" v-model.number="pages" />
        </div>
        <div class="field">
          <label for="copies">Copies</label>
          <input id="copies" type="number" min="1" max="200" v-model.number="copies" />
        </div>
        <div class="field">
          <label for="duplex">Print mode</label>
          <select id="duplex" v-model="duplex">
            <option value="duplex">Duplex</option>
            <option value="simplex">Simplex</option>
          </select>
        </div>
        <div class="field">
          <label for="flip">Flip on</label>
          <select id="flip" v-model="flip" :disabled="!isDuplex">
            <option value="long">Long edge</option>
            <option value="short">Short edge</option>
          </select>
          <p class="hint" v-if="!isDuplex">Available for duplex only.</p>
        </div>
        <div class="callout">
          <p>
            Sheets per copy: <strong>{{ sheetsPerCopy }}</strong>
          </p>
          <p>
            Flip:
            <strong>{{ isDuplex ? (flip === "long" ? "Long edge" : "Short edge") : "—" }}</strong>
          </p>
        </div>
      </section>

      <section class="panel preview" aria-label="3D preview">
        <div class="preview-header">
          <h2>3D preview</h2>
          <span class="muted">Showing {{ renderedSheets }} of {{ totalSheets }} sheets</span>
        </div>
        <div class="stack-scene">
          <div class="stack">
            <div
              v-for="index in sheetIndices"
              :key="index"
              class="sheet"
              :style="sheetStyle(index)"
            >
              <span>Sheet {{ renderedSheets - index }}</span>
            </div>
          </div>
        </div>
        <div class="legend">
          <div>
            <span class="dot front"></span>
            <span>Front side</span>
          </div>
          <div>
            <span class="dot back"></span>
            <span>Back side</span>
          </div>
          <div v-if="hiddenSheets > 0" class="muted">+ {{ hiddenSheets }} more sheets</div>
        </div>
      </section>
    </main>
  </div>
</template>
