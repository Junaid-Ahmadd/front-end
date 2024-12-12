<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import { spring } from 'svelte/motion';
  
  export let screenshots: Map<string, string>;
  export let crawledLinks: Array<{ url: string; depth: number }>;
  export let isOpen: boolean;
  export let onClose: () => void;

  const dispatch = createEventDispatcher();
  
  // Canvas state
  let canvas: HTMLDivElement;
  let minimap: HTMLDivElement;
  let isDragging = false;
  let isPanning = false;
  let startX = 0;
  let startY = 0;
  let lastX = 0;
  let lastY = 0;
  
  // Viewport state
  const viewport = spring({ x: 0, y: 0, scale: 1 }, {
    stiffness: 0.2,
    damping: 0.6
  });
  
  // Constants
  const MIN_SCALE = 0.1;
  const MAX_SCALE = 5;
  const ZOOM_SPEED = 0.002;
  const GRID_SIZE = 50;
  
  // Screenshot layout
  let layout = new Map<string, { x: number, y: number, width: number, height: number }>();
  
  function initLayout() {
    layout.clear();
    let currentX = 0;
    const padding = 20; // Space between screenshots
    
    crawledLinks.forEach(({ url, depth }) => {
      const screenshot = screenshots.get(url);
      if (screenshot) {
        // Create a temporary image to get the natural dimensions
        const img = new Image();
        img.src = screenshot;
        
        // Calculate dimensions maintaining aspect ratio
        const maxWidth = 400; // Maximum width for a screenshot
        const scaleFactor = maxWidth / img.naturalWidth;
        const width = maxWidth;
        const height = img.naturalHeight * scaleFactor;
        
        // Place screenshots in a single row
        layout.set(url, {
          x: currentX,
          y: 50, // Fixed Y position for single row
          width,
          height
        });
        
        currentX += width + padding;
      }
    });
    
    layout = new Map(layout);
  }
  
  function handleWheel(event: WheelEvent) {
    event.preventDefault();
    event.stopPropagation(); // Prevent page scrolling
    
    if (event.ctrlKey || event.metaKey) {
      // Zoom
      const rect = canvas.getBoundingClientRect();
      const mouseX = event.clientX - rect.left;
      const mouseY = event.clientY - rect.top;
      
      const delta = -event.deltaY * ZOOM_SPEED;
      const oldScale = $viewport.scale;
      const newScale = Math.max(MIN_SCALE, Math.min(MAX_SCALE, oldScale * (1 + delta)));
      
      if (newScale !== oldScale) {
        const scaleRatio = newScale / oldScale;
        const focusX = mouseX - $viewport.x;
        const focusY = mouseY - $viewport.y;
        
        viewport.set({
          scale: newScale,
          x: mouseX - focusX * scaleRatio,
          y: mouseY - focusY * scaleRatio
        });
      }
    } else {
      // Pan
      viewport.set({
        ...$viewport,
        x: $viewport.x - event.deltaX,
        y: $viewport.y - event.deltaY
      });
    }
  }
  
  function startDragging(event: MouseEvent) {
    if (event.button !== 0) return;
    
    isDragging = true;
    startX = event.clientX - $viewport.x;
    startY = event.clientY - $viewport.y;
    canvas.style.cursor = 'grabbing';
  }
  
  function handleDrag(event: MouseEvent) {
    if (!isDragging) return;
    
    viewport.set({
      ...$viewport,
      x: event.clientX - startX,
      y: event.clientY - startY
    });
  }
  
  function stopDragging() {
    isDragging = false;
    canvas.style.cursor = 'grab';
  }
  
  function drawMinimap() {
    if (!minimap) return;
    
    const ctx = minimap.getContext('2d');
    if (!ctx) return;
    
    ctx.clearRect(0, 0, minimap.width, minimap.height);
    
    // Draw background
    ctx.fillStyle = 'var(--surface)';
    ctx.fillRect(0, 0, minimap.width, minimap.height);
    
    // Draw screenshots
    layout.forEach((pos, url) => {
      ctx.fillStyle = 'var(--primary)';
      ctx.fillRect(
        pos.x * 0.1,
        pos.y * 0.1,
        pos.width * 0.1,
        pos.height * 0.1
      );
    });
    
    // Draw viewport
    ctx.strokeStyle = 'var(--text)';
    ctx.strokeRect(
      -$viewport.x * 0.1,
      -$viewport.y * 0.1,
      canvas.clientWidth * 0.1,
      canvas.clientHeight * 0.1
    );
  }
  
  // Initialize
  onMount(() => {
    console.log("Screenshots received:", screenshots);
    console.log("Screenshots base64 data:", Array.from(screenshots.values()));
    initLayout();
    viewport.subscribe(drawMinimap);
  });
  
  $: {
    if (screenshots || crawledLinks) {
      initLayout();
    }
  }

  // Touch event handlers for mobile interactions
  let isTouching = false;
  let lastTouchPosition = { x: 0, y: 0 };
  let scale = 1;

  function handleTouchStart(event) {
      event.preventDefault(); // Prevent default scrolling
      event.stopPropagation(); // Prevent touch events from affecting the page
      isTouching = true;
      lastTouchPosition = { x: event.touches[0].clientX, y: event.touches[0].clientY };
  }

  function handleTouchMove(event) {
      event.preventDefault(); // Prevent default scrolling
      event.stopPropagation(); // Prevent touch events from affecting the page
      if (event.touches.length === 1) {
          // Handle panning
          if (!isTouching) return;
          const dx = event.touches[0].clientX - lastTouchPosition.x;
          const dy = event.touches[0].clientY - lastTouchPosition.y;
          viewport.set({
            ...$viewport,
            x: $viewport.x - dx,
            y: $viewport.y - dy
          });
          lastTouchPosition = { x: event.touches[0].clientX, y: event.touches[0].clientY };
      } else if (event.touches.length === 2) {
          // Handle pinch zoom
          handlePinchZoom(event);
      }
  }

  function handleTouchEnd(event) {
      event.preventDefault(); // Prevent default scrolling
      event.stopPropagation(); // Prevent touch events from affecting the page
      isTouching = false;
  }

  function handlePinchZoom(event) {
      if (event.touches.length === 2) {
          const dx = event.touches[0].clientX - event.touches[1].clientX;
          const dy = event.touches[0].clientY - event.touches[1].clientY;
          const distance = Math.sqrt(dx * dx + dy * dy);

          const deltaScale = distance / scale;
          const newScale = Math.max(MIN_SCALE, Math.min(MAX_SCALE, $viewport.scale * deltaScale));

          if (newScale !== $viewport.scale) {
              const rect = canvas.getBoundingClientRect();
              const centerX = (event.touches[0].clientX + event.touches[1].clientX) / 2 - rect.left;
              const centerY = (event.touches[0].clientY + event.touches[1].clientY) / 2 - rect.top;

              const focusX = centerX - $viewport.x;
              const focusY = centerY - $viewport.y;

              viewport.set({
                  scale: newScale,
                  x: centerX - focusX * (newScale / $viewport.scale),
                  y: centerY - focusY * (newScale / $viewport.scale)
              });
              scale = distance;
          }
      }
  }
</script>

<div class="canvas-container" class:open={isOpen} 
     on:touchstart={handleTouchStart} 
     on:touchmove={handleTouchMove} 
     on:touchend={handleTouchEnd}>
  <div class="toolbar">
    <div class="toolbar-left">
      <button on:click={() => viewport.set({ x: 0, y: 0, scale: 1 })}>
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M15 3h6v6M9 21H3v-6M21 3l-7 7M3 21l7-7"/>
        </svg>
        Reset View
      </button>
      <span class="zoom-level">{Math.round($viewport.scale * 100)}%</span>
    </div>
    <div class="toolbar-right">
      <button on:click={onClose}>
        <svg xmlns="http://www.w3.org/200
