<script lang="ts">
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  import { Input } from '$lib/components/ui/input';
  import { Button } from '$lib/components/ui/button';
  import { Card } from '$lib/components/ui/card';
  import { ArrowRight } from 'lucide-svelte';
  import 'leaflet/dist/leaflet.css';

  let map: any;
  let marker: any;
  let L: any;
  let searchInput = '';
  let isLoading = false;
  let error = '';
  let lastFetchTime = 0; // for rate limiting
  const RATE_LIMIT_INTERVAL = 5000; // 5 seconds
  const API_KEY = 'at_hl9uCgMURaHwfVvVB0FcIVYZcl3lF';

  type IpApiResponse = {
    ip: string;
    location: {
      city: string;
      region: string;
      postalCode: string;
      lat: number;
      lng: number;
      timezone: string;
    };
    isp: string;
  };

  let ipData: { ip: string; location: string; timezone: string; isp: string } = {
    ip: '',
    location: '',
    timezone: '',
    isp: ''
  };

  async function getIpInfo(ip = '') {
    const now = Date.now();
    if (now - lastFetchTime < RATE_LIMIT_INTERVAL) {
      error = 'Please wait before making another request.';
      return;
    }

    lastFetchTime = now;
    isLoading = true;
    error = '';

    try {
      const response = await fetch(
        `https://geo.ipify.org/api/v2/country,city?apiKey=${API_KEY}&ipAddress=${ip}`
      );

      if (!response.ok) {
        throw new Error('Failed to fetch IP data');
      }

      const data: IpApiResponse = await response.json();
      console.log("API response:", data); // Debugging line to check the API response

      if (data && data.location) {
        ipData = {
          ip: data.ip,
          location: `${data.location.city}, ${data.location.region} ${data.location.postalCode}`,
          timezone: `UTC ${data.location.timezone}`,
          isp: data.isp
        };

        if (data.location.lat && data.location.lng) {
          updateMap(data.location.lat, data.location.lng); // Update map only if valid coordinates are present
        } else {
          error = 'Location data unavailable for this IP address.';
          console.warn('Invalid coordinates:', data.location);
        }
      } else {
        error = 'No data available for the provided IP address.';
      }
    } catch (err) {
      error = 'Error fetching IP data. Please try again later.';
      console.error('Error:', err);
    } finally {
      isLoading = false;
    }
  }

  function updateMap(lat: number, lng: number) {
    if (map) {
      map.setView([lat, lng], 13); // Move map to new latitude and longitude
      marker.setLatLng([lat, lng]); // Move marker to new location
    }
  }

  function handleSubmit() {
    if (searchInput.trim()) {
      getIpInfo(searchInput); // Fetch IP information for the new input
    } else {
      error = 'Please enter a valid IP address or domain.';
    }
  }

  onMount(async () => {
    if (browser) {
      L = await import('leaflet');
      map = L.map('map').setView([0, 0], 13);
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors'
      }).addTo(map);

      const icon = L.icon({
        iconUrl: 'https://maps.gstatic.com/mapfiles/api-3/images/spotlight-poi.png',
        iconSize: [25, 41],
        iconAnchor: [12, 41]
      });

      marker = L.marker([0, 0], { icon }).addTo(map);

      getIpInfo(); // Fetch initial IP information on mount
    }
  });
</script>

<div class="min-h-screen h-[120vh] md:h-max flex flex-col">
  <div class="bg-[url('./assets/patternbg.png')] bg-cover bg-center p-8 text-white text-center relative z-10 h-96 md:h-auto"> 
    <h1 class="text-3xl font-bold mb-8">IP Address Tracker</h1>
    
    <div class="max-w-xl mx-auto flex mb-8 rounded-lg h-14">
      <Input
        type="text"
        placeholder="Search for any IP address or domain"
        class="border-none h-full bg-white rounded-lg rounded-r-none text-black text-base"
        bind:value={searchInput}
        on:keydown={(e) => e.key === 'Enter' && handleSubmit()}
      />
      <Button
        class="bg-black hover:bg-gray-800 rounded-l-none px-6 h-full"
        on:click={handleSubmit}
        disabled={isLoading}
      >
        {#if isLoading}
          <span>Loading...</span>
        {:else}
          <ArrowRight class="h-5 w-5" />
        {/if}
      </Button>
    </div>

    {#if error}
      <p class="text-red-500 mb-4">{error}</p>
    {/if}

    <Card class="bg-white text-black p-8 rounded-lg shadow-lg max-w-4xl mx-auto -mb-20">
      <div class="flex flex-col md:flex-row justify-between text-center md:text-left gap-4">
        <div>
          <p class="text-gray-500 text-sm uppercase tracking-wider mb-2">IP ADDRESS</p>
          <p class="font-bold text-xl">{ipData.ip}</p>
        </div>
        <div class="md:border-l md:pl-8">
          <p class="text-gray-500 text-sm uppercase tracking-wider mb-2">LOCATION</p>
          <p class="font-bold text-xl">{ipData.location}</p>
        </div>
        <div class="md:border-l md:pl-8">
          <p class="text-gray-500 text-sm uppercase tracking-wider mb-2">TIMEZONE</p>
          <p class="font-bold text-xl">{ipData.timezone}</p>
        </div>
        <div class="md:border-l md:pl-8">
          <p class="text-gray-500 text-sm uppercase tracking-wider mb-2">ISP</p>
          <p class="font-bold text-xl">{ipData.isp}</p>
        </div>
      </div>
    </Card>
  </div>

  <div id="map" class="flex-1 z-0"></div>
</div>

<style>
  :global(#map) {
    height: 65vh;
    width: 100%;
  }
</style>
