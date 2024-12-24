<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';

  let searchQuery = '';
  const searchResults = writable([]);
  const pdfUrl = writable(null);
  let proxyUrl = "http://104.129.194.43:10005"; // Default proxy
  let annasArchiveBase = "annas-archive.org/scidb";

  let totalResults = 0;
  let stats = {
    openAccess: 0,
    openAccessCompatible: 0,
    closed: 0,
    closedWithUrl: 0
  };
  let currentPage = 1;

  async function searchPapers() {
    if (!searchQuery) return;

    // Reset stats and results for a new search
    searchResults.set([]);
    stats = {
      openAccess: 0,
      openAccessCompatible: 0,
      closed: 0,
      closedWithUrl: 0
    };
    currentPage = 1;
    await fetchResults();
  }

  async function fetchResults() {
    const response = await fetch(`https://api.openalex.org/works?search=${encodeURIComponent(searchQuery)}&page=${currentPage}`);
    const data = await response.json();

    totalResults = data.meta.count;

    const enrichedResults = await Promise.all(
      (data.results || []).map(async (result) => {
        const pdfUrl = await getPdfUrl(result);
        if (pdfUrl) {
          result.pdfUrl = pdfUrl;
        }
        categorizeResult(result, pdfUrl);
        return result;
      })
    );

    searchResults.update(results => [...results, ...enrichedResults]);
    currentPage++;
  }

  function categorizeResult(result, pdfUrl) {
    if (result.open_access?.is_oa) {
      stats.openAccess++;
      if (pdfUrl) {
        stats.openAccessCompatible++;
      }
    } else {
      stats.closed++;
      if (pdfUrl) {
        stats.closedWithUrl++;
      }
    }
  }

  function openPdf(url) {
    pdfUrl.set(url);
  }

  async function getPdfUrl(result) {
    try {
      if (result.open_access?.pdf_url) {
        return result.open_access.pdf_url;
      } else if (result.open_access?.oa_url?.includes("arxiv")) {
        return result.open_access.oa_url.replace("abs", "pdf");
      } else if (result.open_access?.oa_url?.endsWith(".pdf")) {
        return result.open_access.oa_url;
      } else if (result.doi) {
        const doiPath = result.doi.split("doi.org/")[1];
        const annasUrl = `${annasArchiveBase}/${doiPath}`;

        const proxyResponse = await fetch(`https://api.codetabs.com/v1/proxy?quest=${annasUrl}`);
        const html = await proxyResponse.text();

        const fileParamMatch = html.match(/file=([^&"]+)/);
        if (fileParamMatch) {
          const fileParam = decodeURIComponent(fileParamMatch[1]);
          return fileParam.replace("https%3A//", "https://");
        }
      }
    } catch (err) {
      console.error(`Error fetching PDF for result ${result.display_name}: ${err.message}`);
    }
    return null;
  }
</script>

<style>
  .container {
    display: flex;
    height: 100vh;
  }

  .search-results {
    width: 30%;
    padding: 1rem;
    overflow-y: auto;
    border-right: 1px solid #ccc;
  }

  .pdf-viewer {
    width: 70%;
    height: 100%;
  }

  .search-field {
    margin-bottom: 1rem;
  }

  .result {
    padding: 0.5rem;
    border: 1px solid #ccc;
    margin-bottom: 0.5rem;
    cursor: pointer;
  }

  .result:hover {
    background-color: #f0f0f0;
  }

  .load-more {
    text-align: center;
    padding: 1rem;
  }
</style>

<div class="container">
  <div class="search-results">
    <input
      class="search-field"
      type="text"
      placeholder="Search research papers"
      bind:value={searchQuery}
    />
    <button on:click={searchPapers}>Search</button>

    {#if totalResults > 0}
      <p>Results: {totalResults}, Open Access: {stats.openAccess}, Open Access Compatible: {stats.openAccessCompatible}, Closed: {stats.closed}, Closed but Available: {stats.closedWithUrl}</p>
    {/if}

    {#if $searchResults.length > 0}
      <div>
        {#each $searchResults as result}
          <div class="result">
            <strong>{result.display_name}</strong>
            <br/>
            <a href={result.doi} target="_blank">{result.doi}</a>
            {#if result.open_access?.is_oa}
              <p style="color: green;">Open Access</p>
            {:else}
              <p style="color: red;">Closed Access</p>
            {/if}
            <p>{result.authorships.map(auth => auth.author.display_name).join(', ')}</p>
            {#if result.pdfUrl}
              <button on:click={() => openPdf(result.pdfUrl)}>Open PDF</button>
            {:else if result.open_access?.oa_url}
              <a href={result.open_access.oa_url} target="_blank">Available here</a>
            {/if}
          </div>
        {/each}
      </div>
      <div class="load-more">
        <button on:click={fetchResults}>Load More</button>
      </div>
    {/if}
  </div>

  {#if $pdfUrl}
    <div class="pdf-viewer">
      <iframe src="{$pdfUrl}#view=FitH" width="100%" height="100%" frameborder="0"></iframe>
    </div>
  {/if}
</div>
