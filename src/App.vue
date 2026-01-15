<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import * as pdfjsLib from "pdfjs-dist";
import pdfjsWorker from "pdfjs-dist/build/pdf.worker?worker";

type DuplexMode = "simplex" | "duplex";
type FlipMode = "long" | "short";
type OrientationMode = "portrait" | "landscape";
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
const orientation = ref<OrientationMode>("portrait");

const isDuplex = computed(() => duplex.value === "duplex");
const isLandscape = computed(() => orientation.value === "landscape");
const sheetsPerCopy = computed(() => (isDuplex.value ? Math.ceil(pages.value / 2) : pages.value));
const totalSheets = computed(() => sheetsPerCopy.value * copies.value);
const totalSides = computed(() => pages.value * copies.value);
const maxRenderedSheets = 100;
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

const resetCamera = () => {
  if (!camera || !controls) return;
  camera.position.set(6, 6.5, 9.5);
  camera.lookAt(0, 0, 0);
  controls.target.set(0, 0, 0);
  controls.update();
};

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

const createLabelSprite = (text: string) => {
  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d");
  if (!context) return null;

  const fontSize = 64;
  const paddingX = 32;
  const paddingY = 22;
  context.font = `600 ${fontSize}px Inter, system-ui, sans-serif`;
  const textWidth = context.measureText(text).width;
  const width = Math.ceil(textWidth + paddingX * 2);
  const height = fontSize + paddingY * 2;
  canvas.width = width;
  canvas.height = height;

  const ctx = canvas.getContext("2d");
  if (!ctx) return null;
  ctx.font = `600 ${fontSize}px Inter, system-ui, sans-serif`;
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillStyle = "rgba(15, 23, 42, 0.85)";
  ctx.fillText(text, width / 2, height / 2);

  const texture = new THREE.CanvasTexture(canvas);
  texture.colorSpace = THREE.SRGBColorSpace;
  disposable.push(texture);
  const material = new THREE.SpriteMaterial({ map: texture, transparent: true });
  disposable.push(material);
  const sprite = new THREE.Sprite(material);
  return sprite;
};

const createPageMaterial = (url: string | null, rotation = 0) => {
  const material = new THREE.MeshStandardMaterial({
    color: url ? 0xffffff : 0xf1f5f9,
    roughness: 0.6,
    metalness: 0.02,
    side: THREE.DoubleSide,
    map: url ? createTexture(url, rotation) : null,
    polygonOffset: true,
    polygonOffsetFactor: 1,
    polygonOffsetUnits: 1,
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
  const aspect = base.width / base.height;
  const isSourceLandscape = aspect >= 1;
  const targetAspect = isLandscape.value
    ? Math.max(aspect, 1 / aspect)
    : Math.min(aspect, 1 / aspect);
  const planeWidth = planeHeight * targetAspect;
  const sheets = totalSheets.value;
  const renderCount = Math.min(sheets, maxRenderedSheets);
  const spacingX = planeWidth * 1.25;
  const spacingZ = planeHeight * 1.5;
  const columns = Math.min(10, Math.max(1, renderCount));
  const rows = Math.ceil(renderCount / columns);
  const totalWidth = (columns - 1) * spacingX;
  const totalDepth = (rows - 1) * spacingZ;

  for (let i = 0; i < renderCount; i += 1) {
    const sheetIndex = i % sheetsPerCopy.value;
    const frontPage = isDuplex.value ? sheetIndex * 2 : sheetIndex;
    const backPage = isDuplex.value ? sheetIndex * 2 + 1 : null;
    const frontIndex = frontPage < pageCount ? frontPage : null;
    const backIndex = backPage !== null && backPage < pageCount ? backPage : null;

    const frontUrl = frontIndex !== null ? pageRenders.value[frontIndex]?.url ?? null : null;
    const backUrl = backIndex !== null ? pageRenders.value[backIndex]?.url ?? null : null;
    const effectiveFlip: FlipMode = isLandscape.value
      ? flip.value === "long"
        ? "short"
        : "long"
      : flip.value;
    const orientationRotation = isLandscape.value === isSourceLandscape ? 0 : Math.PI / 2;
    const backRotation = (effectiveFlip === "short" ? Math.PI : 0) + orientationRotation;

    const geometry = new THREE.PlaneGeometry(planeWidth, planeHeight);
    disposable.push(geometry);

    const frontMaterial = createPageMaterial(frontUrl, orientationRotation);
    const backMaterial = createPageMaterial(backUrl, backRotation);

    const frontMesh = new THREE.Mesh(geometry, frontMaterial);
    frontMesh.position.z = 0.01;
    const backMesh = new THREE.Mesh(geometry, backMaterial);
    backMesh.rotation.y = Math.PI;
    backMesh.position.z = -0.01;

    const sheetGroup = new THREE.Group();
    sheetGroup.add(frontMesh, backMesh);

    const col = i % columns;
    const row = Math.floor(i / columns);
    const x = col * spacingX - totalWidth / 2;
    const z = totalDepth / 2 - row * spacingZ;
    sheetGroup.position.set(x, 0, z);
    sheetGroup.rotation.x = THREE.MathUtils.degToRad(-18);

    const label = createLabelSprite(String(i + 1));
    if (label) {
      label.position.set(0, -planeHeight * 0.62, 0.05);
      label.scale.set(0.9, 0.35, 1);
      label.renderOrder = 5;
      sheetGroup.add(label);
    }

    stackGroup.add(sheetGroup);
  }
};

const initScene = () => {
  if (!previewRef.value) return;

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setClearColor(0x000000, 0);
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.15;
  previewRef.value.appendChild(renderer.domElement);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 120);
  camera.position.set(6, 6.5, 9.5);
  camera.lookAt(0, 0, 0);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.minDistance = 2.4;
  controls.maxDistance = 24;
  controls.zoomSpeed = 1.1;
  controls.maxPolarAngle = THREE.MathUtils.degToRad(80);
  controls.enablePan = true;
  controls.panSpeed = 0.9;
  controls.screenSpacePanning = true;
  controls.target.set(0, 0, 0);
  controls.update();

  const ambient = new THREE.AmbientLight(0xffffff, 0.9);
  const hemi = new THREE.HemisphereLight(0xffffff, 0xe2e8f0, 0.45);
  const key = new THREE.DirectionalLight(0xffffff, 1.1);
  key.position.set(4, 6, 6);
  const fill = new THREE.DirectionalLight(0xffffff, 0.55);
  fill.position.set(-5, 3, 2);
  const rim = new THREE.DirectionalLight(0xffffff, 0.35);
  rim.position.set(0, 6, -6);
  scene.add(ambient, hemi, key, fill, rim);

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

watch(orientation, () => {
  buildStack();
  resetCamera();
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
          Upload a document, choose your settings, and preview a 3D stack of your output.
        </p>
      </div>
    </header>

    <main class="layout">
      <div class="top-row">
        <section class="panel upload compact" aria-label="Upload">
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

        <section class="panel summary-panel" aria-label="Summary">
          <h2>Summary</h2>
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
          <div class="summary">
            <div>
              <span class="label">Sheets per copy</span>
              <strong>{{ sheetsPerCopy }}</strong>
            </div>
            <div>
              <span class="label">Flip</span>
              <strong>{{ isDuplex ? (flip === "long" ? "Long edge" : "Short edge") : "—" }}</strong>
            </div>
            <div>
              <span class="label">Orientation</span>
              <strong>{{ isLandscape ? "Landscape" : "Portrait" }}</strong>
            </div>
          </div>
        </section>
      </div>

      <section class="panel settings compact" aria-label="Print settings">
        <h2>Print settings</h2>
        <div class="settings-grid">
          <div class="field">
            <label for="pages">Pages per copy</label>
            <input
              id="pages"
              type="number"
              min="1"
              :max="maxPageCount ?? 500"
              v-model.number="pages"
            />
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
          </div>
          <div class="field">
            <label for="orientation">Orientation</label>
            <select id="orientation" v-model="orientation">
              <option value="portrait">Portrait</option>
              <option value="landscape">Landscape</option>
            </select>
          </div>
        </div>
      </section>

      <section class="panel preview" aria-label="3D preview">
        <div class="preview-header">
          <h2>3D preview</h2>
          <div class="preview-actions">
            <span class="muted">Showing {{ renderedSheets }} of {{ totalSheets }} sheets</span>
            <button type="button" class="ghost" @click="resetCamera">Reset view</button>
          </div>
        </div>
        <div class="stack-scene" ref="previewRef">
          <div v-if="!hasRenderedPages && !isLoading" class="empty-state">
            Upload a file to preview each page.
          </div>
          <div v-if="isLoading" class="empty-state">Rendering pages…</div>
          <div v-if="loadError" class="empty-state error">{{ loadError }}</div>
        </div>
        <div v-if="hiddenSheets > 0" class="muted">+ {{ hiddenSheets }} more sheets</div>
      </section>
    </main>
  </div>
</template>
