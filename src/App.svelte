<script lang="ts">
  import { onMount } from 'svelte';
  import InfiniteCanvas from './lib/InfiniteCanvas.svelte';
  import { ImageIcon } from 'lucide-svelte';
  
  let url = "";
  let crawledLinks: Array<{ url: string; depth: number }> = [];
  let screenshots: Map<string, string> = new Map();
  let isProcessing = false;
  let isCanvasOpen = false;
  let totalLinks = 0;
  let uniqueLinks = new Set<string>();

  async function submitUrl() {
    if (!url) return;

    try {
      // Reset all state
      crawledLinks = [];
      screenshots = new Map();
      uniqueLinks.clear();
      totalLinks = 0;
      isProcessing = true;

      // Directly fetch from Azure Function
      const response = await fetch(import.meta.env.VITE_AZURE_FUNCTION_URL, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ url })
      });

      if (!response.ok) {
        throw new Error('Crawling failed');
      }

      const data = await response.json();

      // Process links
      crawledLinks = data.links.map(link => ({ url: link, depth: 0 }));
      uniqueLinks = new Set(crawledLinks.map(link => link.url));
      totalLinks = uniqueLinks.size;

      // Process screenshots
      screenshots = new Map(
        data.screenshots.map(screenshot => [
          screenshot.url, 
          screenshot.data // This should be the base64 encoded data
        ])
      );

      // Log the contents of the screenshots Map
      console.log("Screenshots Map:", Array.from(screenshots.entries()));

      isProcessing = false;

    } catch (error) {
      console.error(`Crawling error: ${error.message}`);
      isProcessing = false;
    }
  }

  function toggleCanvas() {
    isCanvasOpen = !isCanvasOpen;
  }

  onMount(() => {
    return () => {
    };
  });
</script>


<main class="container">
  <div class="header">
    <h1>Web Crawler & Screenshot Tool</h1>
    <p class="subtitle">Uncover website structures and capture visual representations effortlessly.</p>
  </div>

  <div class="input-section">
    <div class="url-input">
      <input
        type="text"
        bind:value={url}
        placeholder="Enter website URL (e.g., https://example.com)"
        on:keydown={(e) => e.key === 'Enter' && submitUrl()}
        disabled={isProcessing}
      />
      <button
        class="submit-btn"
        on:click={submitUrl}
        disabled={!url || isProcessing}
      >
        {#if isProcessing}
          <svg class="spinner" viewBox="0 0 24 24">
            <circle cx="12" cy="12" r="10" fill="none" stroke="currentColor" stroke-width="4" />
          </svg>
          Processing...
        {:else}
          <span class="btn-text">Start Crawling</span>
        {/if}
      </button>
    </div>

    {#if totalLinks > 0}
      <div class="stats">
        <div class="stat">
          <span class="label">Unique Pages Found</span>
          <span class="value">{totalLinks}</span>
        </div>
        <div class="stat">
          <span class="label">Screenshots Captured</span>
          <span class="value">{screenshots.size}</span>
        </div>
      </div>
    {/if}
  </div>

  {#if screenshots.size > 0}
    <button
      class="view-canvas toggle-canvas-btn"
      on:click={toggleCanvas}
    >
      <ImageIcon class="btn-icon" />
      <span class="btn-text">{isCanvasOpen ? 'Hide Screenshots' : 'View Screenshots'}</span>
    </button>
  {/if}

  <div class="content-section">
    {#if totalLinks > 0}
      <div class="links-list">
        <h2>Discovered Links</h2>
        {#each Array.from(uniqueLinks) as link}
          <div class="link-item">
            <span class="link-url">{link}</span>
            {#if screenshots.has(link)}
              <span class="status captured">Captured</span>
            {/if}
          </div>
        {/each}
      </div>
    {/if}
  </div>

  {#if isCanvasOpen}
    <InfiniteCanvas
      {screenshots}
      {crawledLinks}
      isOpen={isCanvasOpen}
      onClose={toggleCanvas}
    />
  {/if}
</main>

<style>
  /* Global variables for colors and fonts */
  :root {
    --primary: #6c63ff;
    --primary-hover: #584fd1;
    --secondary: #f3f4f6;
    --text-primary: #333;
    --text-secondary: #666;
    --background: #f9f9fb;
    --surface: #fff;
    --shadow: rgba(0, 0, 0, 0.1);
    --border-radius: 10px;
    --font-family: 'Roboto', sans-serif;
  }

  /* Responsive Base Styles */
  body {
    font-family: var(--font-family);
    color: var(--text-primary);
    background: var(--background);
    margin: 0;
    padding: 0;
    line-height: 1.6;
    display: flex;
    justify-content: center;
  }

  .container {
    width: 100%;
    max-width: 800px; /* Reduced max-width for better centering */
    margin: 0 auto;
    padding: 1rem;
    box-shadow: 0 4px 6px var(--shadow);
    border-radius: var(--border-radius);
    background: var(--surface);
  }

  /* Overflow prevention and input fixes */
  input, .submit-btn, .stats, .stat {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* Button Text Centering */
  .btn-text {
    text-align: center;
    display: inline-block;
    width: 100%;
  }

  .submit-btn, .toggle-canvas-btn {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5rem;
  }

  .btn-icon {
    width: 18px;
    height: 18px;
  }

  /* Responsive Adjustments */
  @media screen and (max-width: 768px) {
    .submit-btn, .toggle-canvas-btn {
      width: auto; /* Reduced width */
      max-width: 250px;
      margin: 0 auto;
    }
  }

  /* Link URL Left Alignment */
  .link-url {
    text-align: left;
    display: block;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* Remaining styles from previous version */
  /* ... (rest of the styles remain the same) */
</style>
