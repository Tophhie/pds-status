<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">

<script lang="ts">
  import { onMount } from 'svelte';
  import { getDidsFromPDS, 
  getHealthFromPDS, 
  getDescriptionFromPDS, 
  getHandleFromDid,
  getTotalPostsThisYear,
  getBlobUsageFromPDS,
  getUptimeForMonth,
  formatDuration } from '$lib/api';
	import type { Repo } from '@atproto/api/dist/client/types/com/atproto/sync/listRepos';

  let metrics = {
    load: '0.00 / 0.00 / 0.00',
    cpu: '0.0%',
    mem: '611.8 MB / 3.7 GB',
    net: '3.4 GB / 1.9 GB',
    diskUsed: '2.3 GB',
    diskFree: '28.2 GB'
  };

  let accounts: Repo[] = [];
  let pdsHealth: any;
  let pdsDescription: any;
  let r2StorageUsage: string = 'Loading...';
  let totalPostsThisYear: number = 0;

  let currentMonthUptime: string = 'Loading...';
  let currentMonthUptimeValue: number = 0;
  let previousMonthUptime: string = 'Loading...';
  let previousMonthUptimeValue: number = 0;
  let totalDowntimeThisMonth: string = 'Loading...';

onMount(async () => {
  try {
    const [
      accountsData,
      pdsHealthData,
      pdsDescriptionData,
      totalPostsData,
      r2StorageData,
      currentMonthData,
      previousMonthData
    ] = await Promise.all([
      getDidsFromPDS(),
      getHealthFromPDS(),
      getDescriptionFromPDS(),
      getTotalPostsThisYear(),
      getBlobUsageFromPDS(),
      getUptimeForMonth(0),
      getUptimeForMonth(-1)
    ]);

    // Assign results
    accounts = accountsData;
    pdsHealth = pdsHealthData;
    pdsDescription = pdsDescriptionData;
    totalPostsThisYear = totalPostsData;
    r2StorageUsage = r2StorageData;

    currentMonthUptime = `${currentMonthData.availability.toFixed(2)}%`;
    currentMonthUptimeValue = currentMonthData.availability.toFixed(2);
    previousMonthUptime = `${previousMonthData.availability.toFixed(2)}%`;
    previousMonthUptimeValue = previousMonthData.availability.toFixed(2);
    totalDowntimeThisMonth = formatDuration(currentMonthData.total_downtime);

  } catch (error) {
    console.error('Error fetching data:', error);
  }
});

</script>

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
        <p class="font-semibold text-sm sm:text-base">{pdsHealth?.version != null ? "✅" : "❌"}</p>
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
<!-- Interesting Stats -->
<section class="mb-6 sm:mb-8">
  <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Statistics</h2>
  <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-5 gap-3 sm:gap-4 text-center bg-gray-800 p-3 sm:p-4 rounded-lg">
    
    <!-- Total Posts -->
    <div class="relative group">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Total Bluesky posts created on tophhie.social in the current year. Data may be stale or cached for up to 1 hour.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Total Bluesky Posts for {(new Date()).getFullYear()} 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base">{totalPostsThisYear}</p>
    </div>

    <!-- Cloudflare R2 Usage -->
    <div class="relative group">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Data may be stale or cached for up to 1 hour
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Cloudflare R2 Blob Usage 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base">{r2StorageUsage}</p>
    </div>

    <!-- Uptime Last Month -->
    <div class="relative group">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Uptime for last month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base" class:text-red-500={previousMonthUptimeValue < 99.9} class:text-green-500={previousMonthUptimeValue >= 99.9}>
        {previousMonthUptime}
      </p>
    </div>

    <!-- Uptime This Month -->
    <div class="relative group">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Uptime for this month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base" class:text-red-500={currentMonthUptimeValue < 99.9} class:text-green-500={currentMonthUptimeValue >= 99.9}>
        {currentMonthUptime}
      </p>
    </div>

    <!-- Total Downtime -->
    <div class="relative group">
      <div class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 hidden group-hover:block w-max-48 bg-black text-white text-xs rounded px-2 py-1">
        Uptime may be stale or cached for up to 10 minutes.
        <div class="absolute left-1/2 -translate-x-1/2 top-full h-0 w-0 border-l-4 border-r-4 border-t-4 border-l-transparent border-r-transparent border-t-black"></div>
      </div>
      <p class="text-gray-400 text-xs sm:text-sm mb-1">
        Total Downtime This Month 
        <i class="fa fa-info-circle text-gray-400 cursor-pointer"></i>
      </p>
      <p class="font-semibold text-sm sm:text-base">{totalDowntimeThisMonth}</p>
    </div>

  </div>
</section>

  <!-- Accounts Table -->
  <section class="mb-6 sm:mb-8">
    <h2 class="text-lg sm:text-xl font-semibold mb-3 sm:mb-4">Accounts on tophhie.social</h2>
    <div class="text-xs sm:text-sm text-gray-400 mb-3 sm:mb-4">
      <p class="break-words">
        <strong>Available User Domains:</strong> {pdsDescription?.availableUserDomains.join(", ")}
      </p>
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
            {#await getHandleFromDid(acc.did)}
              <p class="text-sm">Loading...</p>
            {:then handle}
              <p class="text-sm">{handle}</p>
            {:catch error}
              <p class="text-sm text-red-400">Error loading handle</p>
            {/await}
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
            <th class="px-4 py-2 text-left">DID</th>
            <th class="px-4 py-2 text-left">Handle</th>
            <th class="px-4 py-2 text-left">PLC Directory</th>
          </tr>
        </thead>
        <tbody>
          {#each accounts as acc}
            <tr class="border-t border-gray-700 hover:bg-gray-800">
              <td class="px-4 py-2 break-all">{acc.did}</td>
              {#await getHandleFromDid(acc.did)}
                <td class="px-4 py-2">Loading...</td>
              {:then handle}
                <td class="px-4 py-2">{handle}</td>
              {:catch error}
                <td class="px-4 py-2 text-red-400">Error</td>
              {/await}
              <td class="px-4 py-2">
                <a 
                  href="https://plc.directory/{acc.did}" 
                  target="_blank" 
                  rel="noopener noreferrer"
                  class="inline-block p-2 hover:bg-gray-700 rounded transition-colors" 
                  title="Open PLC Directory"
                  aria-label="Open PLC Directory for {acc.did}"
                >
                  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
                    <path d="M14 3h7v7h-2V6.41l-9.29 9.29-1.42-1.42L17.59 5H14V3z"/>
                    <path d="M5 5h7v2H7v10h10v-5h2v7H5V5z"/>
                  </svg>
                </a>
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
</div>
