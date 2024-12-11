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
          <span>🚀 Start Crawling</span>
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
        {isCanvasOpen ? 'Hide Screenshots' : 'View Screenshots'}
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
    --primary: #111827;
    --secondary: white;
    --text-primary: #111827;
    --text-secondary: gray;
    --background: #111827;
    --surface: #111827;
    --shadow: rgba(255, 255, 255, 0.1);
    --font-family: 'Roboto', sans-serif;
  }

  /* General styles */
  body {
    font-family: var(--font-family);
    color: var(--text-primary);
    background: var(--background);
    margin: 0;
    padding: 0;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
    background: none;
    box-shadow: none;
    border-radius: 0;
    padding: 0;
  }

  .header {
    text-align: center;
    margin-bottom: 2rem;
  }

  h1 {
    font-size: 2.5rem;
    margin: 0;
    color: white;
  }

  .subtitle {
    color: var(--text-secondary);
    font-size: 1rem;
  }

  .input-section {
    margin-bottom: 2rem;
    color: var(--secondary);
  }

  .url-input {
    display: flex;
    gap: 1rem;
    align-items: center;
  }

  input {
    flex: 1;
    padding: 10px;
    border: 2px solid white;
    border-radius: 10px;
    background: transparent;
    color: white;
    font-size: 1rem;
  }

  input:disabled {
    background: var(--secondary);
    color: gray;
  }

  .submit-btn {
    padding: 0.5rem 1rem;
    border: 2px solid white;
    border-radius: 10px;
    background: #111827;
    color: white;
    cursor: pointer;
    font-weight: bold;
    transition: all 0.3s;
    font-size: 1rem;
    text-align: center;
  }

  .submit-btn:disabled {
    background: gray;
    color: var(--text-secondary);
    cursor: not-allowed;
  }

  .submit-btn:hover:not(:disabled) {
    background: white;
    color: #111827;
  }

  .stats {
    display: flex;
    gap: 1rem;
    margin-top: 1rem;
    justify-content: space-between;
  }

  .stat {
    padding: 1rem;
    background: #111827;
    color: white;
    text-align: center;
    box-shadow: none;
  }

  .stat .label {
    color: var(--text-secondary);
    font-size: 0.875rem;
  }

  .stat .value {
    font-size: 1.5rem;
    font-weight: bold;
    color: white;
  }

  .content-section {
    margin-top: 2rem;
  }

  .links-list {
    padding: 0;
    color: white;
  }

  .link-item {
    padding: 0.5rem 0;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    border-bottom: 1px solid gray;
  }

  .link-item:last-child {
    border-bottom: none;
  }

  .link-url {
    color: white;
    word-break: break-word;
    text-align: left;
  }

  .status {
    background: gray;
    color: black;
    padding: 0.25rem 0.5rem;
    font-size: 0.875rem;
    font-weight: bold;
  }

  .toggle-canvas-btn {
    margin-top: 1rem;
    background: #111827;
    color: white;
    padding: 0.5rem 1rem;
    border: 2px solid white;
    border-radius: 10px;
    font-size: 1rem;
    cursor: pointer;
  }

  /* Responsive Design */
  @media (max-width: 768px) {
    .container {
      padding: 0;
    }

    h1 {
      font-size: 2rem;
    }

    .url-input {
      flex-direction: column;
      gap: 0.5rem;
    }

    input {
      width: 100%;
    }

    .stats {
      flex-direction: column;
      gap: 0.5rem;
    }

    .stat {
      padding: 0.5rem;
    }

    .links-list {
      font-size: 0.9rem;
    }

    .toggle-canvas-btn {
      font-size: 0.9rem;
    }

    #app {
      padding: 0;
    }
  }

  @media (max-width: 480px) {
    h1 {
      font-size: 2.5rem;
    }

    .input-section {
      padding: 0;
    }

    .url-input {
      width: 100%;
    }

    input {
      width: 100%;
    }

    .submit-btn {
      width: 100%;
    }

    .link-url {
      font-size: 0.85rem;
      text-align: left;
    }

    .status {
      font-size: 0.7rem;
    }
  }
</style>
