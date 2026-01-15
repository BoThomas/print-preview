<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import * as pdfjsLib from "pdfjs-dist";
import pdfjsWorker from "pdfjs-dist/build/pdf.worker?worker";

type DuplexMode = "simplex" | "duplex";
type FlipMode = "long" | "short";
type PageRender = {
  url: string;
  width: number;
  height: number;
};

pdfjsLib.GlobalWorkerOptions.workerPort = new pdfjsWorker();

const file = ref<File | null>(null);
const isDragging = ref(false);
const isLoading = ref(false);
const loadError = ref<string | null>(null);
const maxPageCount = ref<number | null>(null);
const pageRenders = ref<PageRender[]>([]);
const previewRef = ref<HTMLDivElement | null>(null);
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
const hasRenderedPages = computed(() => pageRenders.value.length > 0);

let renderer: THREE.WebGLRenderer | null = null;
let scene: THREE.Scene | null = null;
let camera: THREE.PerspectiveCamera | null = null;
let controls: OrbitControls | null = null;
let stackGroup: THREE.Group | null = null;
let resizeObserver: ResizeObserver | null = null;
const disposable: Array<THREE.Texture | THREE.Material | THREE.BufferGeometry> = [];

const disposeStack = () => {
  disposable.forEach((item) => item.dispose());
  disposable.length = 0;
  if (!stackGroup) return;
  while (stackGroup.children.length > 0) {
    const child = stackGroup.children.pop();
    if (child) child.removeFromParent();
  }
};

const createTexture = (url: string, rotation = 0) => {
  const texture = new THREE.TextureLoader().load(url);
  texture.colorSpace = THREE.SRGBColorSpace;
  texture.center.set(0.5, 0.5);
  texture.rotation = rotation;
  disposable.push(texture);
  return texture;
};

const createPageMaterial = (url: string | null, rotation = 0) => {
  const material = new THREE.MeshStandardMaterial({
    color: url ? 0xffffff : 0xf1f5f9,
    roughness: 0.85,
    metalness: 0.05,
    side: THREE.DoubleSide,
    map: url ? createTexture(url, rotation) : null,
  });
  disposable.push(material);
  return material;
};

const buildStack = () => {
  if (!scene || !stackGroup) return;
  disposeStack();

  const pageCount = Math.max(pages.value, 1);
  const pageDimensions = pageRenders.value.length
    ? pageRenders.value
    : [{ url: "", width: 1, height: 1.414 }];
  const base = pageDimensions[0]!;
  const planeHeight = 2.6;
  const planeWidth = planeHeight * (base.width / base.height);
  const sheetDepth = 0.03;
  const sheets = totalSheets.value;
  const renderCount = Math.min(sheets, maxRenderedSheets);

  for (let i = 0; i < renderCount; i += 1) {
    const sheetIndex = i % sheetsPerCopy.value;
    const frontPage = isDuplex.value ? sheetIndex * 2 : sheetIndex;
    const backPage = isDuplex.value ? sheetIndex * 2 + 1 : null;
    const frontIndex = frontPage < pageCount ? frontPage : null;
    const backIndex = backPage !== null && backPage < pageCount ? backPage : null;

    const frontUrl = frontIndex !== null ? pageRenders.value[frontIndex]?.url ?? null : null;
    const backUrl = backIndex !== null ? pageRenders.value[backIndex]?.url ?? null : null;
    const backRotation = flip.value === "short" ? Math.PI : 0;

    const geometry = new THREE.PlaneGeometry(planeWidth, planeHeight);
    disposable.push(geometry);

    const frontMaterial = createPageMaterial(frontUrl, 0);
    const backMaterial = createPageMaterial(backUrl, backRotation);

    const frontMesh = new THREE.Mesh(geometry, frontMaterial);
    frontMesh.position.z = i * sheetDepth;
    const backMesh = new THREE.Mesh(geometry, backMaterial);
    backMesh.rotation.y = Math.PI;
    backMesh.position.z = i * sheetDepth - 0.001;

    const sheetGroup = new THREE.Group();
    sheetGroup.add(frontMesh, backMesh);
    sheetGroup.position.y = -i * 0.015;
    sheetGroup.rotation.x = THREE.MathUtils.degToRad(6 + i * 0.2);

    stackGroup.add(sheetGroup);
  }
};

const initScene = () => {
  if (!previewRef.value) return;

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setClearColor(0x000000, 0);
  previewRef.value.appendChild(renderer.domElement);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100);
  camera.position.set(0, 1.3, 6.5);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.minDistance = 3.5;
  controls.maxDistance = 12;

  const ambient = new THREE.AmbientLight(0xffffff, 0.7);
  const key = new THREE.DirectionalLight(0xffffff, 0.9);
  key.position.set(4, 6, 6);
  scene.add(ambient, key);

  stackGroup = new THREE.Group();
  scene.add(stackGroup);

  const resize = () => {
    if (!renderer || !camera || !previewRef.value) return;
    const { width, height } = previewRef.value.getBoundingClientRect();
    if (width === 0 || height === 0) return;
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
  };

  resizeObserver = new ResizeObserver(resize);
  resizeObserver.observe(previewRef.value);
  resize();

  const animate = () => {
    if (!renderer || !scene || !camera || !controls) return;
    controls.update();
    renderer.render(scene, camera);
    requestAnimationFrame(animate);
  };

  animate();
};

const loadPdf = async (selected: File) => {
  const data = await selected.arrayBuffer();
  const loadingTask = pdfjsLib.getDocument({ data });
  const pdf = await loadingTask.promise;
  const count = Math.min(pdf.numPages, 60);
  maxPageCount.value = pdf.numPages;
  pages.value = count;

  const renders: PageRender[] = [];
  for (let i = 1; i <= count; i += 1) {
    const page = await pdf.getPage(i);
    const viewport = page.getViewport({ scale: 1.35 });
    const canvas = document.createElement("canvas");
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    const context = canvas.getContext("2d");
    if (!context) continue;
    await page.render({ canvasContext: context, viewport, canvas }).promise;
    renders.push({
      url: canvas.toDataURL("image/png"),
      width: viewport.width,
      height: viewport.height,
    });
  }
  pageRenders.value = renders;
};

const loadImage = async (selected: File) => {
  const url = URL.createObjectURL(selected);
  const img = new Image();
  img.src = url;
  await new Promise((resolve, reject) => {
    img.onload = () => resolve(true);
    img.onerror = reject;
  });
  const canvas = document.createElement("canvas");
  canvas.width = img.width;
  canvas.height = img.height;
  const context = canvas.getContext("2d");
  if (context) {
    context.drawImage(img, 0, 0);
  }
  URL.revokeObjectURL(url);
  pageRenders.value = [
    { url: canvas.toDataURL("image/png"), width: img.width, height: img.height },
  ];
  maxPageCount.value = 1;
  pages.value = 1;
};

const loadFile = async (selected: File) => {
  file.value = selected;
  isLoading.value = true;
  loadError.value = null;
  pageRenders.value = [];

  try {
    if (selected.type === "application/pdf") {
      await loadPdf(selected);
    } else if (selected.type.startsWith("image/")) {
      await loadImage(selected);
    } else {
      throw new Error("Unsupported file format.");
    }
  } catch (error) {
    loadError.value = error instanceof Error ? error.message : "Failed to read file.";
  } finally {
    isLoading.value = false;
  }
};

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const selected = target.files?.[0] ?? null;
  if (selected) {
    loadFile(selected);
  }
};

const handleDrop = (event: DragEvent) => {
  event.preventDefault();
  isDragging.value = false;
  const dropped = event.dataTransfer?.files?.[0];
  if (dropped) loadFile(dropped);
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
  pageRenders.value = [];
  maxPageCount.value = null;
};

watch([pages, copies, duplex, flip, pageRenders], () => {
  buildStack();
});

onMounted(() => {
  initScene();
  buildStack();
});

onBeforeUnmount(() => {
  resizeObserver?.disconnect();
  controls?.dispose();
  disposeStack();
  renderer?.dispose();
  renderer?.domElement?.remove();
});
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
          <input
            id="pages"
            type="number"
            min="1"
            :max="maxPageCount ?? 500"
            v-model.number="pages"
          />
          <p class="hint" v-if="maxPageCount">Detected {{ maxPageCount }} pages.</p>
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
        <div class="stack-scene" ref="previewRef">
          <div v-if="!hasRenderedPages && !isLoading" class="empty-state">
            Upload a file to preview each page.
          </div>
          <div v-if="isLoading" class="empty-state">Rendering pages…</div>
          <div v-if="loadError" class="empty-state error">{{ loadError }}</div>
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
