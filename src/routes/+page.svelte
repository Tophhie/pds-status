<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">

<script lang="ts">
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import {
    getDidsFromPDS,
    getHealthFromPDS,
    getDescriptionFromPDS,
    getHandleFromDid,
    getTotalPostsThisYear,
    getBlobUsageFromPDS,
    getUptimeForMonth,
    getDidAccessibilityScores,
    formatDuration,
    getMonthNameYear
  } from '$lib/api';
  import type { Repo } from '@atproto/api/dist/client/types/com/atproto/sync/listRepos';

  let showBanner: boolean = false;
  let bannerMsg: string | null = null

  let accounts: Repo[] = [];
  let pdsHealth: any;
  let pdsDescription: any;
  let r2StorageUsage: string | null = null;
  let r2BlobCount: number | null = null;
  let totalPostsThisYear: number | null = null;
  let pdsAccessibilityScore: string | null = null;

  let currentMonthUptime: string | null = null;
  let currentMonthUptimeValue = 0;
  let previousMonthUptime: string | null = null;
  let previousMonthUptimeValue = 0;
  let totalDowntimeThisMonth: string | null = null;

  let handleCache: Record<string, string> = {};
  let blobUsageCache: Record<string, string> = {};
  let accessibilityFetched: boolean = false;
  let accessibilityCache: Record<string, number> = {};
  let accessibilityLastUpdate: string | null = null

  // Sorting state
  let sortColumn: 'did' | 'handle' | 'blobUsage' | 'accessibilityScore' | null = null;
  let sortDirection: 'asc' | 'desc' = 'asc';

  function sortBy(column: 'did' | 'handle' | 'blobUsage' | 'accessibilityScore') {
    if (sortColumn === column) {
      sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      sortColumn = column;
      sortDirection = 'asc';
    }
  }

  function resetSort() {
    sortColumn = null;
    sortDirection = 'asc';
  }

  function copyRepoToClipboard(a: Repo) {
    navigator.clipboard.writeText(JSON.stringify(a)).then(() => { 
      bannerMsg = "Copied to your clipboard!"
    }, function(err) {
      bannerMsg = "Failed to copy to clipboard..."
    })

    showBanner = true
    setTimeout(() => {
      showBanner = false
    }, 5000)    
  }

  function exportTableToCSV() {
    if (!accounts.length) return;

    // CSV headers
    const headers = ['DID', 'Handle', 'Blob Usage', 'Accessibility Score (0-100)', 'PLC Directory'];

    // CSV rows
    const rows = sortedAccounts.map(acc => [
      acc.did,
      handleCache[acc.did] ?? '',
      blobUsageCache[acc.did] ?? '',
      accessibilityCache[acc.did] ?? '',
      `https://plc.directory/${acc.did}`
    ]);

    // Combine headers + rows
    const csvContent =
      [headers, ...rows]
        .map(row => row.map(cell => `"${String(cell).replace(/"/g, '""')}"`).join(','))
        .join('\n');

    // Create blob and trigger download
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);

    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'accounts.csv');
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }


  function parseSize(bytesStr: string, useDecimal = true): string {
    const bytes = parseInt(bytesStr, 10);
    if (isNaN(bytes) || bytes <= 0) return '0 B';

    const base = useDecimal ? 1000 : 1024; // Decimal vs Binary
    const units = ['B', 'KB', 'MB', 'GB', 'TB'];

    let value = bytes;
    let unitIndex = 0;

    while (value >= base && unitIndex < units.length - 1) {
      value /= base;
      unitIndex++;
    }

    return `${value.toFixed(2)} ${units[unitIndex]}`;
  }


  // Reactive sorted list — now tied to handleCache + blobUsageCache
  $: sortedAccounts = [...accounts].sort((a, b) => {
    if (!sortColumn) return 0;

    let valA: string | number = '';
    let valB: string | number = '';

    if (sortColumn === 'did') {
      valA = a.did;
      valB = b.did;
    } else if (sortColumn === 'handle') {
      valA = handleCache[a.did] ?? '';
      valB = handleCache[b.did] ?? '';
    } else if (sortColumn === 'blobUsage') {
      valA = parseSize(blobUsageCache[a.did] ?? '0 KB');
      valB = parseSize(blobUsageCache[b.did] ?? '0 KB');
    } else if (sortColumn === 'accessibilityScore') {
      valA = accessibilityCache[a.did] ?? '';
      valB = accessibilityCache[b.did] ?? '';
    }

    if (typeof valA === 'number' && typeof valB === 'number') {
      return sortDirection === 'asc' ? valA - valB : valB - valA;
    }

    return sortDirection === 'asc'
      ? String(valA).localeCompare(String(valB))
      : String(valB).localeCompare(String(valA));
  });

  onMount(async () => {
    try {
      const [accountsData, pdsHealthData, pdsDescriptionData] = await Promise.all([
        getDidsFromPDS(),
        getHealthFromPDS(),
        getDescriptionFromPDS()
      ]);

      accounts = accountsData;
      pdsHealth = pdsHealthData;
      pdsDescription = pdsDescriptionData;

      getBlobUsageFromPDS().then(data => {
        r2StorageUsage = data.formattedUsage ?? '0 KB';
        r2BlobCount = data.blobCount ?? 0;
      });
      getTotalPostsThisYear().then(posts => totalPostsThisYear = posts ?? 0);

      getUptimeForMonth(0).then(data => {
        currentMonthUptime = `${(Math.floor(data.availability * 100) / 100).toFixed(2)}%`;
        currentMonthUptimeValue = data.availability;
        totalDowntimeThisMonth = formatDuration(data.total_downtime);
      });

      getUptimeForMonth(-1).then(data => {
        previousMonthUptime = `${(Math.floor(data.availability * 100) / 100).toFixed(2)}%`;
        previousMonthUptimeValue = data.availability;
      });

      // Fetch handles + blob usage + accessibility score properly and await them
      await Promise.all(
        accounts.map(async acc => {
          handleCache[acc.did] = await getHandleFromDid(acc.did).catch(() => 'Error');
          blobUsageCache[acc.did] = (await getBlobUsageFromPDS(acc.did).catch(() => ({formattedUsage: '0 KB', blobCount: 0}))).formattedUsage;
        })
      );


      try {
        const scoreData = await getDidAccessibilityScores();

        // Convert individualScores array into a map
        accessibilityCache = Object.fromEntries(
          scoreData.individualScores.map((item: { did: any; score: any; }) => [item.did, item.score])
        );

        accessibilityLastUpdate = new Date(scoreData.lastUpdated).toLocaleString("en-GB", {
          dateStyle: "medium",
          timeStyle: "short"
        });

        pdsAccessibilityScore = scoreData.pdsAccessibilityScore;

        accessibilityFetched = true;

      } catch (err) {
        console.error('Failed to fetch accessibility scores:', err);
        pdsAccessibilityScore = 'Unknown';
        accessibilityFetched = true;
      }

    } catch (error) {
      console.error('Error fetching data:', error);
    }
  });
</script>

{#if showBanner}
  <div
    class="fixed top-4 right-4 bg-purple-600 text-white px-4 py-2 rounded shadow-lg"
    transition:fade
  >
    {bannerMsg}
  </div>
{/if}

<div class="min-h-screen bg-[#100235] text-gray-100 p-4 sm:p-6 md:p-8 lg:p-12">
  <!-- Page Header -->
   <img
    src="https://blob.tophhie.cloud/tophhiecloud-resources/Logos/tophhiecloud-white.png"
    height="50"
    alt="Tophhie Social Logo"
    id="Logo"
    class="h-8 sm:h-10 md:h-12 w-auto mb-4 sm:mb-6"
  />
  <h1 class="text-xl sm:text-2xl md:text-3xl font-bold mb-6 sm:mb-8">Tophhie Social Server Status</h1>

  <!-- Service Information -->
  <section class="mb-6 sm:mb-8">
    <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Service Information</h2>
    <div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-3 sm:gap-4 bg-gray-800 p-3 sm:p-4 rounded-lg">
      <div class="text-center">
        <p class="text-gray-400 text-xs sm:text-sm mb-1">Service Reachable</p>
        {#if pdsHealth?.version != null}
            <i class="fa fa-solid fa-check fa-lg" style="color: #63E6BE;"></i>
        {:else}
            <i class="fa fa-times fa-lg" style="color: #ff0000;"></i>
        {/if}
      </div>
      <div class="text-center">
        <p class="text-gray-400 text-xs sm:text-sm mb-1">PDS Version</p>
        <p class="font-semibold text-sm sm:text-base break-words">{pdsHealth?.version ?? 'Loading...'}</p>
      </div>
      <div class="text-center col-span-2 sm:col-span-1">
        <p class="text-gray-400 text-xs sm:text-sm mb-1">Server DID</p>
        <p class="font-semibold text-xs sm:text-sm break-all">{pdsDescription?.did ?? 'Loading...'}</p>
      </div>
      <div class="text-center">
        <p class="text-gray-400 text-xs sm:text-sm mb-1">No. of Accounts</p>
        <p class="font-semibold text-sm sm:text-base">{accounts.length}</p>
      </div>
      <div class="text-center">
        <p class="text-gray-400 text-xs sm:text-sm mb-1">Invite Required</p>
        <p class="font-semibold text-sm sm:text-base">{pdsDescription?.inviteCodeRequired ? "Yes" : "No"}</p>
      </div>
    </div>
  </section>

<!-- Interesting Stats -->
<section class="mb-6 sm:mb-8">
  <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Statistics</h2>
  <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-5 gap-3 sm:gap-4 text-center bg-gray-800 p-3 sm:p-4 rounded-lg">
    
    <!-- Total Posts -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Total Bluesky posts created on tophhie.social in the current year. Data may be stale or cached for up to 1 hour.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Total Bluesky Posts for {(new Date()).getFullYear()} 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{totalPostsThisYear === null}">
          {#if totalPostsThisYear === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {totalPostsThisYear}
          {/if}
        </p>
    </div>

    <!-- Cloudflare R2 Usage -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Data may be stale or cached for up to 1 hour
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Cloudflare R2 Blob Usage 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{r2StorageUsage === null}">
          {#if r2StorageUsage === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {r2StorageUsage}
          {/if}
        </p>
    </div>

      <!-- Cloudflare R2 Blob Count -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Data may be stale or cached for up to 1 hour
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Cloudflare R2 Blob Count 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{r2StorageUsage === null}">
          {#if r2BlobCount === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {r2BlobCount}
          {/if}
        </p>
    </div>

    <!-- Uptime Last Month -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        {getMonthNameYear(-1)} • Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Uptime for last month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base" class:text-red-500={previousMonthUptimeValue < 99.9} class:text-green-500={previousMonthUptimeValue >= 99.9} aria-busy="{previousMonthUptime === null}">
        {#if previousMonthUptime === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {previousMonthUptime}
          {/if}
      </p>
    </div>

    <!-- Uptime This Month -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        {getMonthNameYear(0)} • Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Uptime for this month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base" class:text-red-500={currentMonthUptimeValue < 99.9} class:text-green-500={currentMonthUptimeValue >= 99.9} aria-busy="{currentMonthUptime === null}">
        {#if currentMonthUptime === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {currentMonthUptime}
          {/if}
      </p>
    </div>

    <!-- Total Downtime -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Total Downtime This Month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{totalDowntimeThisMonth === null}">
          {#if totalDowntimeThisMonth === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {totalDowntimeThisMonth}
          {/if}
        </p>
    </div>

    <!-- PDS Accessibility Score -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        This is the Accessibility Score for Tophhie Social as a whole.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Combined Accessibility Score
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{pdsAccessibilityScore === null}">
          {#if pdsAccessibilityScore === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {pdsAccessibilityScore}
          {/if}
        </p>
    </div>

    <!-- PDS Accessibility Score (Last Update) -->
    <div class="relative group" tabindex="0" role="link">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block group-focus:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        When the Accessibility Scores for the PDS were last updated.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Accessibility Score (Last Updated)
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
        <p class="font-semibold text-sm sm:text-base" aria-busy="{accessibilityLastUpdate === null}">
          {#if accessibilityLastUpdate === null}
            <i class="fa fa-spinner fa-spin text-gray-400"></i>
          {:else}
            {accessibilityLastUpdate}
          {/if}
        </p>
    </div>

  </div>
</section>

  <!-- Accounts Table -->
  <section class="mb-6 sm:mb-8">
    <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Accounts on tophhie.social</h2>
    <div class="text-xs sm:text-sm text-gray-400 mb-3 sm:mb-4">
      <div class="flex justify-between items-center">
        <p class="break-words">
          <strong>Available User Domains:</strong>
          {pdsDescription?.availableUserDomains.join(", ")}
        </p>

        <!-- Button group -->
        <div class="flex space-x-2 hidden sm:block">
          <button
            class="px-3 py-1.5 text-sm bg-gray-700 hover:bg-gray-600 text-gray-200 rounded transition-colors whitespace-nowrap"
            on:click={exportTableToCSV}
          >
            Export CSV
          </button>
          <button
            class="px-3 py-1.5 text-sm bg-gray-700 hover:bg-gray-600 text-gray-200 rounded transition-colors whitespace-nowrap"
            on:click={resetSort}
          >
            Reset Sort
          </button>
        </div>
      </div>
    </div>

    
    <!-- Mobile Card View (shown on small screens) -->
    <div class="block sm:hidden space-y-3">
      {#each accounts as acc}
        <div class="bg-gray-800 p-4 rounded-lg">
          <div class="mb-2">
            <p class="text-xs text-gray-400 mb-1">DID</p>
            <p class="text-sm break-all">{acc.did}</p>
          </div>
          <div class="mb-2">
            <p class="text-xs text-gray-400 mb-1">Handle</p>
            <p class="text-sm">{handleCache[acc.did] ?? 'Loading...'}</p>
          </div>
          <div class="mb-2">
            <p class="text-xs text-gray-400 mb-1">Blob Usage</p>
            <p class="text-sm">{blobUsageCache[acc.did] ?? 'Loading...'}</p>
          </div>
          <div class="mb-2">
            <p class="text-xs text-gray-400 mb-1">Accessibility Score (0-100)</p>
            <p class="text-sm">
                {#if (accessibilityCache[acc.did] === undefined || accessibilityCache[acc.did] === null) && !accessibilityFetched}
                  <i class="fa fa-spinner fa-spin text-gray-400"></i>
                {:else}
                  {accessibilityCache[acc.did] ?? "Not yet calculated."}
                {/if}
            </p>
          </div>
          <div>
            <a 
              href="https://plc.directory/{acc.did}" 
              target="_blank" 
              rel="noopener noreferrer"
              class="inline-flex items-center gap-2 text-blue-400 hover:text-blue-300 text-sm py-2"
              title="Open PLC Directory"
            >
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
                <path d="M14 3h7v7h-2V6.41l-9.29 9.29-1.42-1.42L17.59 5H14V3z"/>
                <path d="M5 5h7v2H7v10h10v-5h2v7H5V5z"/>
              </svg>
              <span>View in PLC Directory</span>
            </a>
          </div>
        </div>
      {/each}
    </div>

    <!-- Desktop Table View (hidden on small screens) -->
    <div class="hidden sm:block overflow-x-auto">
      <table class="min-w-full text-sm border border-gray-700 rounded-lg">
        <thead class="bg-gray-800 text-gray-300">
          <tr>
            <!-- DID -->
            <th 
              class="px-4 py-2 text-left cursor-pointer select-none"
              on:click={() => sortBy('did')}
            >
              DID 
              {#if sortColumn === 'did'}
                {sortDirection === 'asc' ? '▲' : '▼'}
              {/if}
            </th>

            <!-- Handle -->
            <th 
              class="px-4 py-2 text-left cursor-pointer select-none"
              on:click={() => sortBy('handle')}
            >
              Handle
              {#if sortColumn === 'handle'}
                {sortDirection === 'asc' ? '▲' : '▼'}
              {/if}
            </th>

            <!-- Blob Usage -->
            <th 
              class="px-4 py-2 text-left cursor-pointer select-none"
              on:click={() => sortBy('blobUsage')}
            >
              Blob Usage
              {#if sortColumn === 'blobUsage'}
                {sortDirection === 'asc' ? '▲' : '▼'}
              {/if}
            </th>

            <!-- Accessibility Score -->
            <th 
              class="px-4 py-2 text-left cursor-pointer select-none"
              on:click={() => sortBy('accessibilityScore')}
            >
              Accessibility Score (0-100)
              {#if sortColumn === 'accessibilityScore'}
                {sortDirection === 'asc' ? '▲' : '▼'}
              {/if}
            </th>
            <th class="px-4 py-2 text-left">
              PLC Directory
            </th>
            <th class="px-4 py-2 text-left">
              Copy Repo Info
            </th>
          </tr>
        </thead>

        <tbody>
          {#each sortedAccounts as acc}
            <tr class="border-t border-gray-700 hover:bg-gray-800">
              <td class="px-4 py-2 break-all">{acc.did}</td>
              <td class="px-4 py-2">
                {#if handleCache[acc.did]}
                  {handleCache[acc.did]}
                {:else}
                  <i class="fa fa-spinner fa-spin text-gray-400"></i>
                {/if}
              </td>

              <td class="px-4 py-2">
                {#if blobUsageCache[acc.did]}
                  {blobUsageCache[acc.did]}
                {:else}
                  <i class="fa fa-spinner fa-spin text-gray-400"></i>
                {/if}
              </td>

              <td class="px-4 py-2">
                {#if (accessibilityCache[acc.did] === undefined || accessibilityCache[acc.did] === null) && !accessibilityFetched}
                  <i class="fa fa-spinner fa-spin text-gray-400"></i>
                {:else}
                  {accessibilityCache[acc.did] ?? "Not yet calculated."}
                {/if}
              </td>

              <td class="px-4 py-2">
                <a 
                  href="https://plc.directory/{acc.did}" 
                  target="_blank" 
                  rel="noopener noreferrer"
                  class="inline-block p-2 hover:bg-gray-700 rounded transition-colors"
                  title="Open PLC Directory"
                  aria-label="Open PLC Directory for {acc.did}"
                >
                  <i class="fa fa-solid fa-external-link fa-lg"></i>
                </a>
              </td>
              <td class="px-4 py-2">
                <button on:click={() => copyRepoToClipboard(acc)} aria-label="Copy repo info" style="cursor: pointer;">
                  <i class="fa fa-regular fa-copy fa-lg inline-block p-2 hover:bg-gray-700 rounded transition-colors" style="color: white;"></i>
                </button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  </section>


  <!-- Contact -->
  <section class="mb-6 sm:mb-8">
    <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Contact</h2>
    {#if pdsDescription?.contact.email}
      <p class="text-xs sm:text-sm break-words">
        For support or enquiries, please contact us at 
        <a 
          href="mailto:{pdsDescription?.contact.email}" 
          class="text-blue-400 hover:text-blue-300 underline inline-block py-1"
        >
          {pdsDescription?.contact.email}
        </a>.
      </p>
    {/if}
  </section>

  <!-- Links -->
  <section class="mb-6 sm:mb-8">
    <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Links</h2>
    <ul class="space-y-2">
      <li>
        <a 
          href="{pdsDescription?.links.privacyPolicy}" 
          class="text-blue-400 hover:text-blue-300 underline text-sm sm:text-base inline-block py-1"
        >
          Privacy Policy
        </a>
      </li>
      <li>
        <a 
          href="{pdsDescription?.links.termsOfService}" 
          class="text-blue-400 hover:text-blue-300 underline text-sm sm:text-base inline-block py-1"
        >
          Terms of Service
        </a>
      </li>
    </ul>
   </section>

  <!-- Footer -->
  <footer class="mt-12 pt-8 border-t border-gray-700 text-center">
    <p class="text-sm text-gray-400">
      <a href="https://github.com/Tophhie/pds-status" target="_blank" rel="noopener noreferrer" class="text-blue-400 hover:underline inline-flex items-center gap-2">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
          <path d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"/>
        </svg>
        View Source Code
      </a>
    </p>
  </footer>
</div>
