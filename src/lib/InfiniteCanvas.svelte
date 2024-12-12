<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import { spring } from 'svelte/motion';
  import { fabric } from 'fabric'; // Import Fabric.js

  export let screenshots: Map<string, string>;
  export let crawledLinks: Array<{ url: string; depth: number }>;
  export let isOpen: boolean;
  export let onClose: () => void;

  const dispatch = createEventDispatcher();
  
  // Canvas state
  let canvas; // This will be the Fabric.js canvas

  // Viewport state
  const viewport = spring({ x: 0, y: 0, scale: 1 }, {
    stiffness: 0.2,
    damping: 0.6
  });

  onMount(() => {
    const fabricCanvas = new fabric.Canvas('canvas', {
      width: window.innerWidth,
      height: window.innerHeight,
      selection: false,
    });

    // Load screenshots onto the canvas
    crawledLinks.forEach(({ url }) => {
      const screenshot = screenshots.get(url);
      if (screenshot) {
        // Create a Fabric.js image from the base64 screenshot
        fabric.Image.fromURL(`data:image/jpeg;base64,${screenshot}`, (img) => {
          img.set({
            left: Math.random() * 500, // Random position for example
            top: Math.random() * 500,  // Random position for example
            selectable: true,
          });
          fabricCanvas.add(img);
        });
      } else {
        console.warn(`No screenshot found for URL: ${url}`);
      }
    });

    // Enable panning
    let isPanning = false;
    let lastPosX;
    let lastPosY;

    fabricCanvas.on('mouse:down', function (opt) {
      isPanning = true;
      lastPosX = opt.e.clientX;
      lastPosY = opt.e.clientY;
    });

    fabricCanvas.on('mouse:move', function (opt) {
      if (!isPanning) return;
      const deltaX = opt.e.clientX - lastPosX;
      const deltaY = opt.e.clientY - lastPosY;
      fabricCanvas.relativePan({ x: deltaX, y: deltaY });
      lastPosX = opt.e.clientX;
      lastPosY = opt.e.clientY;
    });

    fabricCanvas.on('mouse:up', function () {
      isPanning = false;
    });

    // Enable zooming
    fabricCanvas.on('mouse:wheel', function (opt) {
      const delta = opt.e.deltaY;
      const zoom = fabricCanvas.getZoom();
      fabricCanvas.setZoom(zoom * 0.999 ** delta);
      opt.e.preventDefault();
      opt.e.stopPropagation();
    });
  });
</script>

<canvas id="canvas" class="canvas-container" class:open={isOpen}></canvas>
<div class="toolbar">
  <div class="toolbar-left">
    <button on:click={() => viewport.set({ x: 0, y: 0, scale: 1 })}>
      Reset View
    </button>
  </div>
  <div class="toolbar-right">
    <button on:click={onClose}>
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <line x1="18" y1="6" x2="6" y2="18"/>
        <line x1="6" y1="6" x2="18" y2="18"/>
      </svg>
    </button>
  </div>
</div>

<style>
  .canvas-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: var(--surface);
    z-index: 1000;
  }

  .toolbar {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 48px;
    background: var(--surface-2);
    border-bottom: 1px solid var(--border);
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 16px;
    z-index: 1;
  }

  .toolbar-left,
  .toolbar-right {
    display: flex;
    gap: 8px;
    align-items: center;
  }

  .zoom-level {
    font-size: 14px;
    color: var(--text-2);
  }

  .canvas {
    width: 100%;
    height: 100%;
    cursor: grab;
    overflow: hidden;
  }

  .canvas:active {
    cursor: grabbing;
  }
</style>
