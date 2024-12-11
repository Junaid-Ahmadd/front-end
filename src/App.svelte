<script lang="ts">
  import { onMount } from 'svelte';
  import InfiniteCanvas from './lib/InfiniteCanvas.svelte';
  
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
  <div class="header">...</div>

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
          <svg class="spinner" viewBox="0 0 24 24">...</svg>
          Processing...
        {:else}
          <span>🚀 Start Crawling</span>
        {/if}
      </button>
    </div>

    {#if totalLinks > 0}
      <div class="stats">...</div>
    {/if}
  </div>

{#if screenshots.size > 0}
      <button
        class="view-canvas toggle-canvas-btn"
        on:click={toggleCanvas}
      >
        <span>{isCanvasOpen ? 'Hide Screenshots' : 'View Screenshots'}</span>
        📸
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
    <InfiniteCanvas {screenshots} {crawledLinks} isOpen={isCanvasOpen} onClose={toggleCanvas} />
  {/if}
</main>

<style>
  :root {
    --primary: #6c63ff;
    --primary-hover: #5048d1;
    --secondary: #f3f4f6;
    --text-primary: #333;
    --text-secondary: #666;
    --background: #f9f9fb;
    --surface: #fff;
    --shadow: rgba(0, 0, 0, 0.1);
    --border-radius: 10px;
    --font-family: 'Roboto', sans-serif;
    --input-padding: 0.75rem;
    --button-padding: 0.8rem;
  }

  .container {
    max-width: 100%;
    margin: 0 auto;
    padding: 1rem 16px;
    box-shadow: 0 4px 6px var(--shadow);
    border-radius: var(--border-radius);
    background: var(--surface);
    display: flex;
    flex-direction: column;
  }

  .url-input {
    display: flex;
    gap: 0.75rem;
    align-items: center;
    flex-wrap: wrap;
  }

  input {
    flex: 1;
    padding: var(--input-padding);
    border: none;
    border-radius: var(--border-radius);
    box-shadow: inset 0 2px 4px var(--shadow);
    width: 100%;
  }

  .submit-btn {
    padding: var(--button-padding);
    border: none;
    border-radius: var(--border-radius);
    background: var(--surface);
    color: var(--primary-hover);
    cursor: pointer;
    font-weight: bold;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    transition: all 0.3s;
    width: 100%;
    justify-content: center;
  }

  .submit-btn:hover:not(:disabled) {
    background: var(--primary-hover);
    color: white;
    transform: translateY(-2px);
  }

  .toggle-canvas-btn {
    margin-top: 0.75rem;
    background: var(--primary);
    color: white;
    padding: 0.75rem 1.25rem;
    border: none;
    border-radius: var(--border-radius);
    font-size: 1.125rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    cursor: pointer;
    transition: transform 0.2s;
    justify-content: center;
  }

  .toggle-canvas-btn:hover {
    background: #4e44c0;
    transform: translateY(-3px);
  }

  .link-item {
    padding: 0.5rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--secondary);
    white-space: nowrap;
    gap: 8px;
  }

  .status {
    background: var(--primary);
    color: white;
    padding: 0.25rem 0.4rem;
    border-radius: var(--border-radius);
    font-size: 0.75rem;
    font-weight: bold;
  }
</style>
