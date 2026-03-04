<script lang="ts">
  import { onMount } from 'svelte';

  // Default: Valencia City, Bukidnon
  let LAT = 7.9064;
  let LON = 125.0941;
  let cityName = 'Valencia City';
  let regionName = 'Bukidnon, Philippines';

  interface DayWeather {
    date: string;
    tempMax: number;
    tempMin: number;
    temp: number;
    feelsLike: number;
    humidity: number;
    windSpeed: number;
    weatherCode: number;
    uvIndex: number;
    sunrise: string;
    sunset: string;
    isDay: number;
  }

  interface HourlyPoint { hour: number; temp: number; }

  let current: DayWeather | null = null;
  let forecast: DayWeather[] = [];
  let history: DayWeather[] = [];
  let hourlyToday: HourlyPoint[] = [];
  let loading = true;
  let error = '';
  let lastUpdated = '';
  let isNight = false;

  // City search
  let searchQuery = '';
  let searchResults: any[] = [];
  let searchLoading = false;
  let showSearch = false;

  // ── Weather helpers ──────────────────────────────────────
  function getWeatherLabel(code: number): string {
    if (code === 0) return 'Clear Sky';
    if (code <= 2) return 'Partly Cloudy';
    if (code === 3) return 'Overcast';
    if (code <= 49) return 'Foggy';
    if (code <= 57) return 'Drizzle';
    if (code <= 67) return 'Rainy';
    if (code <= 77) return 'Snowy';
    if (code <= 82) return 'Rain Showers';
    if (code <= 86) return 'Snow Showers';
    if (code <= 99) return 'Thunderstorm';
    return 'Unknown';
  }
  function getWeatherType(code: number): string {
    if (code === 0) return 'clear';
    if (code <= 2) return 'partly-cloudy';
    if (code === 3) return 'cloudy';
    if (code <= 49) return 'foggy';
    if (code <= 57) return 'drizzle';
    if (code <= 67) return 'rain';
    if (code <= 77) return 'snow';
    if (code <= 82) return 'rain';
    if (code <= 86) return 'snow';
    if (code <= 99) return 'thunder';
    return 'clear';
  }
  function uvLabel(uv: number): string {
    if (uv <= 2) return 'Low';
    if (uv <= 5) return 'Moderate';
    if (uv <= 7) return 'High';
    if (uv <= 10) return 'Very High';
    return 'Extreme';
  }
  function uvColor(uv: number): string {
    if (uv <= 2) return '#4caf50';
    if (uv <= 5) return '#ffeb3b';
    if (uv <= 7) return '#ff9800';
    if (uv <= 10) return '#f44336';
    return '#9c27b0';
  }
  function formatDate(dateStr: string, opts: Intl.DateTimeFormatOptions): string {
    return new Date(dateStr + 'T00:00:00').toLocaleDateString('en-US', opts);
  }
  function avgHumidity(times: string[], values: number[], date: string): number {
    const v = times.map((t, i) => ({ t, h: values[i] })).filter(({ t }) => t.startsWith(date)).map(({ h }) => h);
    return v.length ? Math.round(v.reduce((a, b) => a + b, 0) / v.length) : 0;
  }
  function parseTime(iso: string): string {
    const d = new Date(iso);
    return d.toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit', hour12: true });
  }

  // Sunrise arc: 0–1 progress through the day
  function sunProgress(sunriseIso: string, sunsetIso: string): number {
    const now = Date.now();
    const rise = new Date(sunriseIso).getTime();
    const set  = new Date(sunsetIso).getTime();
    if (now <= rise) return 0;
    if (now >= set)  return 1;
    return (now - rise) / (set - rise);
  }

  // Sparkline SVG path from hourly temps
  function sparklinePath(points: HourlyPoint[], w: number, h: number): string {
    if (!points.length) return '';
    const temps = points.map(p => p.temp);
    const min = Math.min(...temps);
    const max = Math.max(...temps);
    const range = max - min || 1;
    const xs = points.map((_, i) => (i / (points.length - 1)) * w);
    const ys = points.map(p => h - ((p.temp - min) / range) * (h - 8) - 4);
    let d = `M ${xs[0]} ${ys[0]}`;
    for (let i = 1; i < xs.length; i++) {
      const cpx = (xs[i - 1] + xs[i]) / 2;
      d += ` C ${cpx} ${ys[i - 1]}, ${cpx} ${ys[i]}, ${xs[i]} ${ys[i]}`;
    }
    return d;
  }
  function sparklineArea(points: HourlyPoint[], w: number, h: number): string {
    const base = sparklinePath(points, w, h);
    if (!base) return '';
    const temps = points.map(p => p.temp);
    const lastX = w;
    return `${base} L ${lastX} ${h} L 0 ${h} Z`;
  }

  // ── Fetch ────────────────────────────────────────────────
  async function fetchWeatherData() {
    loading = true; error = '';
    try {
      const today = new Date();
      const todayStr = today.toISOString().split('T')[0];
      const startHistory = new Date(today.getTime() - 7 * 86400000).toISOString().split('T')[0];

      const [histRes, foreRes] = await Promise.all([
        fetch(`https://archive-api.open-meteo.com/v1/archive?latitude=${LAT}&longitude=${LON}&start_date=${startHistory}&end_date=${todayStr}&daily=temperature_2m_max,temperature_2m_min,weathercode,windspeed_10m_max,uv_index_max,sunrise,sunset&hourly=relativehumidity_2m,apparent_temperature&timezone=Asia%2FManila`),
        fetch(`https://api.open-meteo.com/v1/forecast?latitude=${LAT}&longitude=${LON}&daily=temperature_2m_max,temperature_2m_min,weathercode,windspeed_10m_max,uv_index_max,sunrise,sunset&hourly=temperature_2m,relativehumidity_2m,apparent_temperature&current_weather=true&timezone=Asia%2FManila&forecast_days=3`)
      ]);

      if (!histRes.ok) throw new Error(`History API ${histRes.status}`);
      if (!foreRes.ok) throw new Error(`Forecast API ${foreRes.status}`);

      const [hd, fd] = await Promise.all([histRes.json(), foreRes.json()]);
      const ti = Math.max(0, fd.daily.time.indexOf(todayStr));

      isNight = fd.current_weather.is_day === 0;

      // Feels like for current hour
      const nowHour = today.getHours();
      const hourlyTimes: string[] = fd.hourly.time;
      const hourlyApparent: number[] = fd.hourly.apparent_temperature;
      const curHourIdx = hourlyTimes.findIndex(t => t.startsWith(todayStr + 'T' + String(nowHour).padStart(2,'0')));
      const feelsLikeCurrent = curHourIdx >= 0 ? Math.round(hourlyApparent[curHourIdx]) : Math.round(fd.current_weather.temperature);

      current = {
        date: todayStr,
        tempMax: Math.round(fd.daily.temperature_2m_max[ti]),
        tempMin: Math.round(fd.daily.temperature_2m_min[ti]),
        temp: Math.round(fd.current_weather.temperature),
        feelsLike: feelsLikeCurrent,
        humidity: avgHumidity(fd.hourly.time, fd.hourly.relativehumidity_2m, todayStr),
        windSpeed: Math.round((fd.current_weather.windspeed / 3.6) * 10) / 10,
        weatherCode: fd.current_weather.weathercode,
        uvIndex: Math.round(fd.daily.uv_index_max[ti] ?? 0),
        sunrise: fd.daily.sunrise[ti],
        sunset: fd.daily.sunset[ti],
        isDay: fd.current_weather.is_day,
      };

      // Hourly sparkline — today's temps
      const hourlyTemps: number[] = fd.hourly.temperature_2m;
      hourlyToday = hourlyTimes
        .map((t, i) => ({ t, temp: hourlyTemps[i] }))
        .filter(({ t }) => t.startsWith(todayStr))
        .map(({ t, temp }) => ({ hour: new Date(t).getHours(), temp: Math.round(temp) }));

      forecast = fd.daily.time.slice(1, 3).map((date: string, i: number) => ({
        date,
        tempMax: Math.round(fd.daily.temperature_2m_max[i + 1]),
        tempMin: Math.round(fd.daily.temperature_2m_min[i + 1]),
        temp: Math.round((fd.daily.temperature_2m_max[i + 1] + fd.daily.temperature_2m_min[i + 1]) / 2),
        feelsLike: 0,
        humidity: avgHumidity(fd.hourly.time, fd.hourly.relativehumidity_2m, date),
        windSpeed: Math.round((fd.daily.windspeed_10m_max[i + 1] / 3.6) * 10) / 10,
        weatherCode: fd.daily.weathercode[i + 1],
        uvIndex: Math.round(fd.daily.uv_index_max[i + 1] ?? 0),
        sunrise: fd.daily.sunrise[i + 1],
        sunset: fd.daily.sunset[i + 1],
        isDay: 1,
      }));

      history = hd.daily.time.slice(0, -1).map((date: string, i: number) => ({
        date,
        tempMax: Math.round(hd.daily.temperature_2m_max[i]),
        tempMin: Math.round(hd.daily.temperature_2m_min[i]),
        temp: Math.round((hd.daily.temperature_2m_max[i] + hd.daily.temperature_2m_min[i]) / 2),
        feelsLike: 0, humidity: 0, windSpeed: 0,
        weatherCode: hd.daily.weathercode[i],
        uvIndex: Math.round(hd.daily.uv_index_max?.[i] ?? 0),
        sunrise: hd.daily.sunrise[i],
        sunset: hd.daily.sunset[i],
        isDay: 1,
      })).reverse();

      lastUpdated = new Date().toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit' });

    } catch (e: any) {
      error = e.message || 'Failed to load weather data.';
    } finally {
      loading = false;
    }
  }

  // ── City Search ──────────────────────────────────────────
  let searchTimer: any;
  async function onSearchInput() {
    clearTimeout(searchTimer);
    if (searchQuery.length < 2) { searchResults = []; return; }
    searchTimer = setTimeout(async () => {
      searchLoading = true;
      try {
        const res = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(searchQuery)}&count=5&language=en&format=json`);
        const data = await res.json();
        searchResults = data.results || [];
      } catch { searchResults = []; }
      finally { searchLoading = false; }
    }, 350);
  }

  async function selectCity(r: any) {
    LAT = r.latitude; LON = r.longitude;
    cityName = r.name;
    regionName = [r.admin1, r.country].filter(Boolean).join(', ');
    showSearch = false; searchQuery = ''; searchResults = [];
    await fetchWeatherData();
  }

  onMount(fetchWeatherData);

  // Reactive sun progress
  $: sunProg = current ? sunProgress(current.sunrise, current.sunset) : 0;
  $: sunArcX = 16 + sunProg * 168;
  $: sunArcY = 48 - Math.sin(sunProg * Math.PI) * 38;
</script>

<svelte:head>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@200;300;400;500;600;700;800&display=swap" rel="stylesheet">
</svelte:head>

<div class="app" class:night={isNight} class:day={!isNight && !!current}>
  <div class="bg-mesh"></div>
  <div class="grain"></div>

  <!-- Night stars -->
  {#if isNight}
    <div class="stars">
      {#each Array(60) as _, i}
        <div class="star" style="
          left:{Math.random()*100}%;top:{Math.random()*60}%;
          width:{Math.random()*2+1}px;height:{Math.random()*2+1}px;
          animation-delay:{Math.random()*4}s;animation-duration:{2+Math.random()*3}s
        "></div>
      {/each}
    </div>
  {/if}

  <!-- Rain particles -->
  {#if current && (getWeatherType(current.weatherCode) === 'rain' || getWeatherType(current.weatherCode) === 'drizzle')}
    <div class="rain-wrap" aria-hidden="true">
      {#each Array(28) as _, i}
        <div class="raindrop" style="left:{i*3.6+Math.random()*3}%;animation-delay:{Math.random()*1.5}s;animation-duration:{0.6+Math.random()*0.5}s;opacity:{0.3+Math.random()*0.35}"></div>
      {/each}
    </div>
  {/if}

  <main class="wrap">
    {#if loading}
      <div class="state-center">
        <div class="pulse-ring"></div>
        <div class="pulse-dot"></div>
        <p class="state-title">Loading weather</p>
        <p class="state-sub">{cityName} · {regionName}</p>
      </div>

    {:else if error}
      <div class="state-center">
        <div class="err-icon">
          <svg viewBox="0 0 24 24" width="34" height="34" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round">
            <path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/>
            <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
          </svg>
        </div>
        <p class="state-title">Something went wrong</p>
        <p class="state-sub">{error}</p>
        <button class="btn-retry" on:click={fetchWeatherData}>Retry</button>
      </div>

    {:else if current}

      <!-- TOP BAR -->
      <div class="topbar">
        <button class="location-btn" on:click={() => { showSearch = !showSearch; searchQuery = ''; searchResults = []; }}>
          <svg viewBox="0 0 24 24" width="13" height="13" fill="currentColor" opacity="0.7">
            <path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5c-1.38 0-2.5-1.12-2.5-2.5s1.12-2.5 2.5-2.5 2.5 1.12 2.5 2.5-1.12 2.5-2.5 2.5z"/>
          </svg>
          <span>{cityName}</span>
          <svg viewBox="0 0 24 24" width="11" height="11" fill="none" stroke="currentColor" stroke-width="2.5" opacity="0.5">
            <polyline points="6 9 12 15 18 9"/>
          </svg>
        </button>
        <div class="topbar-right">
          {#if lastUpdated}<span class="updated-badge">Updated {lastUpdated}</span>{/if}
          <button class="refresh-btn" on:click={fetchWeatherData} title="Refresh">
            <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
              <polyline points="23 4 23 10 17 10"/>
              <path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/>
            </svg>
          </button>
        </div>
      </div>

      <!-- SEARCH PANEL -->
      {#if showSearch}
        <div class="search-panel">
          <div class="search-input-wrap">
            <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" opacity="0.5">
              <circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/>
            </svg>
            <input
              class="search-input"
              type="text"
              placeholder="Search city…"
              bind:value={searchQuery}
              on:input={onSearchInput}
              autofocus
            />
            {#if searchLoading}
              <div class="search-spinner"></div>
            {/if}
          </div>
          {#if searchResults.length > 0}
            <div class="search-results">
              {#each searchResults as r}
                <button class="search-result-item" on:click={() => selectCity(r)}>
                  <span class="sr-name">{r.name}</span>
                  <span class="sr-region">{[r.admin1, r.country].filter(Boolean).join(', ')}</span>
                </button>
              {/each}
            </div>
          {/if}
        </div>
      {/if}

      <!-- HERO -->
      <div class="hero">
        <div class="hero-icon">
          {#if isNight && getWeatherType(current.weatherCode) === 'clear'}
            <!-- Moon -->
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs>
                <radialGradient id="moonG" cx="45%" cy="40%" r="55%">
                  <stop offset="0%" stop-color="#fffde7"/>
                  <stop offset="60%" stop-color="#fff9c4"/>
                  <stop offset="100%" stop-color="#f9a825"/>
                </radialGradient>
                <filter id="mglow"><feGaussianBlur stdDeviation="6" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#mglow)">
                <circle cx="80" cy="80" r="38" fill="url(#moonG)" opacity="0.15"/>
                <path d="M80 44 A36 36 0 1 0 80 116 A28 28 0 1 1 80 44Z" fill="url(#moonG)"/>
              </g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'clear'}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs>
                <radialGradient id="sg" cx="50%" cy="38%" r="55%">
                  <stop offset="0%" stop-color="#fff9c4"/>
                  <stop offset="45%" stop-color="#ffd54f"/>
                  <stop offset="100%" stop-color="#fb8c00"/>
                </radialGradient>
                <filter id="sglow"><feGaussianBlur stdDeviation="5" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#sglow)">
                {#each Array(12) as _, i}
                  <line x1="80" y1="{i%2===0?22:26}" x2="80" y2="10" stroke="#ffd54f" stroke-width="{i%2===0?3:2}" stroke-linecap="round" transform="rotate({i*30} 80 80)" opacity="{i%2===0?1:0.5}"/>
                {/each}
                <circle cx="80" cy="80" r="34" fill="url(#sg)" opacity="0.25"/>
                <circle cx="80" cy="80" r="26" fill="url(#sg)"/>
              </g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'partly-cloudy'}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs>
                <radialGradient id="sg2" cx="40%" cy="40%" r="60%"><stop offset="0%" stop-color="#fff9c4"/><stop offset="100%" stop-color="#ffb300"/></radialGradient>
                <linearGradient id="cg2" x1="0%" y1="0%" x2="20%" y2="100%"><stop offset="0%" stop-color="#e3f2fd"/><stop offset="100%" stop-color="#90caf9"/></linearGradient>
                <filter id="csh"><feDropShadow dx="0" dy="6" stdDeviation="8" flood-color="rgba(0,80,150,0.3)"/></filter>
              </defs>
              <circle cx="55" cy="65" r="22" fill="url(#sg2)"/>
              {#each Array(8) as _,i}
                <line x1="55" y1="{38-(i%2)*3}" x2="55" y2="30" stroke="#ffd54f" stroke-width="2.5" stroke-linecap="round" transform="rotate({i*45} 55 65)" opacity="{i%2===0?1:0.45}"/>
              {/each}
              <g filter="url(#csh)">
                <ellipse cx="100" cy="98" rx="35" ry="22" fill="url(#cg2)"/>
                <ellipse cx="75" cy="104" rx="24" ry="18" fill="url(#cg2)"/>
                <ellipse cx="88" cy="90" rx="28" ry="22" fill="white" opacity="0.9"/>
                <ellipse cx="108" cy="96" rx="22" ry="17" fill="url(#cg2)"/>
              </g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'rain' || getWeatherType(current.weatherCode) === 'drizzle'}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs>
                <linearGradient id="rcg" x1="0%" y1="0%" x2="10%" y2="100%"><stop offset="0%" stop-color="#546e7a"/><stop offset="100%" stop-color="#263238"/></linearGradient>
                <filter id="rsh"><feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="rgba(0,50,100,0.4)"/></filter>
              </defs>
              <g filter="url(#rsh)">
                <ellipse cx="95" cy="72" rx="38" ry="23" fill="url(#rcg)" opacity="0.6"/>
                <ellipse cx="66" cy="78" rx="27" ry="19" fill="url(#rcg)"/>
                <ellipse cx="80" cy="66" rx="32" ry="25" fill="#455a64" opacity="0.95"/>
                <ellipse cx="104" cy="74" rx="24" ry="18" fill="url(#rcg)"/>
              </g>
              {#each [[52,102],[66,110],[80,102],[94,110],[108,102],[66,122],[80,130],[94,122]] as [x,y],i}
                <ellipse cx={x} cy={y} rx="3" ry="6" fill="#64b5f6" opacity="0.85">
                  <animate attributeName="cy" values="{y};{y+14};{y+14}" dur="{0.7+i*0.09}s" repeatCount="indefinite"/>
                  <animate attributeName="opacity" values="0.85;0;0.85" dur="{0.7+i*0.09}s" repeatCount="indefinite"/>
                </ellipse>
              {/each}
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'thunder'}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs><linearGradient id="thcg" x1="0%" y1="0%" x2="0%" y2="100%"><stop offset="0%" stop-color="#37474f"/><stop offset="100%" stop-color="#102027"/></linearGradient><filter id="lglow"><feGaussianBlur stdDeviation="3" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter></defs>
              <ellipse cx="95" cy="68" rx="40" ry="24" fill="url(#thcg)"/>
              <ellipse cx="65" cy="75" rx="28" ry="20" fill="url(#thcg)"/>
              <ellipse cx="80" cy="62" rx="34" ry="26" fill="#455a64"/>
              <ellipse cx="106" cy="72" rx="26" ry="20" fill="url(#thcg)"/>
              <g filter="url(#lglow)">
                <polygon points="85,94 72,118 82,118 68,146 100,108 88,108 98,94" fill="#ffd600">
                  <animate attributeName="opacity" values="1;0.3;1;0.7;1" dur="2.2s" repeatCount="indefinite"/>
                </polygon>
              </g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'snow'}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs><linearGradient id="sncg" x1="0%" y1="0%" x2="10%" y2="100%"><stop offset="0%" stop-color="#e3f2fd"/><stop offset="100%" stop-color="#90caf9"/></linearGradient><filter id="snsh"><feDropShadow dx="0" dy="4" stdDeviation="8" flood-color="rgba(100,180,255,0.3)"/></filter></defs>
              <g filter="url(#snsh)">
                <ellipse cx="95" cy="70" rx="38" ry="22" fill="url(#sncg)" opacity="0.6"/>
                <ellipse cx="66" cy="76" rx="27" ry="18" fill="url(#sncg)"/>
                <ellipse cx="80" cy="64" rx="32" ry="24" fill="white" opacity="0.9"/>
              </g>
              {#each [[52,100],[68,108],[84,100],[100,108],[116,100],[60,122],[80,128],[100,122]] as [x,y],i}
                <g transform="translate({x},{y})">
                  <line x1="-6" y1="0" x2="6" y2="0" stroke="white" stroke-width="1.5" opacity="0.7"/>
                  <line x1="0" y1="-6" x2="0" y2="6" stroke="white" stroke-width="1.5" opacity="0.7"/>
                  <line x1="-4" y1="-4" x2="4" y2="4" stroke="white" stroke-width="1.5" opacity="0.5"/>
                  <line x1="4" y1="-4" x2="-4" y2="4" stroke="white" stroke-width="1.5" opacity="0.5"/>
                  <animate attributeName="opacity" values="0.7;0.2;0.7" dur="{1.4+i*0.18}s" repeatCount="indefinite"/>
                </g>
              {/each}
            </svg>

          {:else}
            <svg viewBox="0 0 160 160" width="140" height="140" class="anim-float">
              <defs><linearGradient id="cldg" x1="0%" y1="0%" x2="15%" y2="100%"><stop offset="0%" stop-color="#cfd8dc"/><stop offset="100%" stop-color="#78909c"/></linearGradient><filter id="csh2"><feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="rgba(0,0,0,0.25)"/></filter></defs>
              <g filter="url(#csh2)">
                <ellipse cx="95" cy="88" rx="38" ry="24" fill="url(#cldg)" opacity="0.5"/>
                <ellipse cx="66" cy="95" rx="28" ry="20" fill="url(#cldg)"/>
                <ellipse cx="80" cy="82" rx="32" ry="26" fill="#eceff1" opacity="0.95"/>
                <ellipse cx="104" cy="92" rx="25" ry="19" fill="url(#cldg)"/>
              </g>
            </svg>
          {/if}
        </div>

        <div class="hero-info">
          <div class="hero-temp">
            <span class="temp-num">{current.temp}</span>
            <span class="temp-deg">°C</span>
          </div>
          <div class="hero-cond">{getWeatherLabel(current.weatherCode)}</div>
          <div class="feels-like">Feels like {current.feelsLike}°C</div>
          <div class="hero-range">
            <span class="rh">H:{current.tempMax}°</span>
            <span class="rdiv">·</span>
            <span class="rl">L:{current.tempMin}°</span>
          </div>
        </div>
      </div>

      <!-- STAT PILLS -->
      <div class="pills">
        <div class="pill">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M12 22a7 7 0 007-7c0-2-1-3.9-3-5.5S12 4 12 2 7 8.5 5 9.5A7 7 0 005 15a7 7 0 007 7z"/></svg>
          <span class="pill-val">{current.humidity}%</span>
          <span class="pill-lbl">Humidity</span>
        </div>
        <div class="pill">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M9.59 4.59A2 2 0 1 1 11 8H2m10.59 11.41A2 2 0 1 0 14 16H2m15.73-8.27A2.5 2.5 0 1 1 19.5 12H2"/></svg>
          <span class="pill-val">{current.windSpeed}</span>
          <span class="pill-lbl">m/s Wind</span>
        </div>
        <div class="pill">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41"/></svg>
          <span class="pill-val" style="color:{uvColor(current.uvIndex)}">{current.uvIndex}</span>
          <span class="pill-lbl">UV · {uvLabel(current.uvIndex)}</span>
        </div>
      </div>

      <!-- HOURLY SPARKLINE -->
      {#if hourlyToday.length > 1}
        <div class="card sparkline-card">
          <div class="card-header">
            <span class="card-title">Today's Temperature</span>
            <span class="spark-range">{Math.min(...hourlyToday.map(h=>h.temp))}° – {Math.max(...hourlyToday.map(h=>h.temp))}°C</span>
          </div>
          <div class="spark-wrap">
            <svg viewBox="0 0 320 64" preserveAspectRatio="none" width="100%" height="64">
              <defs>
                <linearGradient id="sparkGrad" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stop-color="rgba(100,200,255,0.35)"/>
                  <stop offset="100%" stop-color="rgba(100,200,255,0)"/>
                </linearGradient>
              </defs>
              <path d={sparklineArea(hourlyToday, 320, 64)} fill="url(#sparkGrad)"/>
              <path d={sparklinePath(hourlyToday, 320, 64)} fill="none" stroke="#64c8ff" stroke-width="2" stroke-linecap="round"/>
            </svg>
            <div class="spark-labels">
              {#each hourlyToday.filter((_,i) => i % 4 === 0) as pt}
                <span>{pt.hour === 0 ? '12a' : pt.hour < 12 ? pt.hour+'a' : pt.hour === 12 ? '12p' : (pt.hour-12)+'p'}</span>
              {/each}
            </div>
          </div>
        </div>
      {/if}

      <!-- SUNRISE / SUNSET ARC -->
      <div class="card sun-card">
        <div class="card-header">
          <span class="card-title">Sun · {isNight ? 'Night' : 'Daylight'}</span>
          <span class="sun-pct">{Math.round(sunProg * 100)}% of day passed</span>
        </div>
        <div class="sun-arc-wrap">
          <svg viewBox="0 0 200 80" width="100%" height="90">
            <defs>
              <linearGradient id="arcGrad" x1="0%" y1="0%" x2="100%" y2="0%">
                <stop offset="0%" stop-color="#fb8c00" stop-opacity="0.6"/>
                <stop offset="50%" stop-color="#ffd54f" stop-opacity="0.9"/>
                <stop offset="100%" stop-color="#ff7043" stop-opacity="0.6"/>
              </linearGradient>
            </defs>
            <!-- Track -->
            <path d="M16 68 Q100 4 184 68" fill="none" stroke="rgba(255,255,255,0.07)" stroke-width="2"/>
            <!-- Progress -->
            <path d="M16 68 Q100 4 184 68" fill="none" stroke="url(#arcGrad)" stroke-width="2"
              stroke-dasharray="220"
              stroke-dashoffset="{220 - sunProg * 220}"
              stroke-linecap="round"/>
            <!-- Sun dot -->
            <circle cx={sunArcX} cy={sunArcY} r="7" fill="#ffd54f" opacity="0.95">
              <animate attributeName="r" values="7;8.5;7" dur="2.5s" repeatCount="indefinite"/>
            </circle>
            <circle cx={sunArcX} cy={sunArcY} r="12" fill="#ffd54f" opacity="0.18"/>
            <!-- Horizon line -->
            <line x1="10" y1="70" x2="190" y2="70" stroke="rgba(255,255,255,0.08)" stroke-width="1"/>
          </svg>
          <div class="sun-times">
            <div class="sun-time">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="#ffd54f" stroke-width="2" stroke-linecap="round"><path d="M17 18a5 5 0 00-10 0"/><line x1="12" y1="2" x2="12" y2="9"/><line x1="4.22" y1="10.22" x2="5.64" y2="11.64"/><line x1="18.36" y1="11.64" x2="19.78" y2="10.22"/></svg>
              <div>
                <span class="sun-label">Sunrise</span>
                <span class="sun-val">{parseTime(current.sunrise)}</span>
              </div>
            </div>
            <div class="sun-time">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="#ff9800" stroke-width="2" stroke-linecap="round"><path d="M17 18a5 5 0 00-10 0"/><line x1="12" y1="9" x2="12" y2="2"/><line x1="4.22" y1="10.22" x2="5.64" y2="11.64"/><line x1="18.36" y1="11.64" x2="19.78" y2="10.22"/></svg>
              <div>
                <span class="sun-label">Sunset</span>
                <span class="sun-val">{parseTime(current.sunset)}</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- FORECAST -->
      <div class="block-label">2-Day Forecast</div>
      <div class="forecast-row">
        {#each forecast as day, i}
          <div class="fc-card" style="animation-delay:{i*0.08}s">
            <span class="fc-day">{formatDate(day.date, { weekday: 'long' })}</span>
            <span class="fc-dt">{formatDate(day.date, { month: 'short', day: 'numeric' })}</span>
            <div class="fc-icon">
              {#if getWeatherType(day.weatherCode) === 'clear'}
                <svg viewBox="0 0 56 56" width="44" height="44"><defs><radialGradient id="fcs{i}"><stop offset="0%" stop-color="#fff9c4"/><stop offset="100%" stop-color="#fb8c00"/></radialGradient></defs>{#each Array(8) as _,j}<line x1="28" y1="9" x2="28" y2="5" stroke="#ffd54f" stroke-width="2" stroke-linecap="round" transform="rotate({j*45} 28 28)"/>{/each}<circle cx="28" cy="28" r="11" fill="url(#fcs{i})"/></svg>
              {:else if getWeatherType(day.weatherCode) === 'partly-cloudy'}
                <svg viewBox="0 0 56 56" width="44" height="44"><circle cx="20" cy="24" r="10" fill="#ffb300"/>{#each Array(6) as _,j}<line x1="20" y1="10" x2="20" y2="7" stroke="#ffd54f" stroke-width="1.8" stroke-linecap="round" transform="rotate({j*60} 20 24)"/>{/each}<ellipse cx="36" cy="34" rx="14" ry="9" fill="#cfd8dc"/><ellipse cx="26" cy="37" rx="10" ry="7" fill="#eceff1"/><ellipse cx="32" cy="30" rx="12" ry="9" fill="white" opacity="0.9"/></svg>
              {:else if getWeatherType(day.weatherCode) === 'rain' || getWeatherType(day.weatherCode) === 'drizzle'}
                <svg viewBox="0 0 56 56" width="44" height="44"><ellipse cx="34" cy="22" rx="16" ry="10" fill="#546e7a" opacity="0.7"/><ellipse cx="22" cy="26" rx="12" ry="8" fill="#455a64"/><ellipse cx="28" cy="20" rx="14" ry="10" fill="#607d8b"/>{#each [[18,38],[26,42],[34,38],[42,42]] as [x,y]}<ellipse cx={x} cy={y} rx="2.5" ry="5" fill="#64b5f6" opacity="0.85"/>{/each}</svg>
              {:else if getWeatherType(day.weatherCode) === 'thunder'}
                <svg viewBox="0 0 56 56" width="44" height="44"><ellipse cx="34" cy="20" rx="16" ry="10" fill="#37474f"/><ellipse cx="22" cy="25" rx="12" ry="8" fill="#263238"/><ellipse cx="28" cy="18" rx="14" ry="10" fill="#455a64"/><polygon points="28,32 22,44 27,44 20,56 36,40 30,40 34,32" fill="#ffd600"/></svg>
              {:else}
                <svg viewBox="0 0 56 56" width="44" height="44"><ellipse cx="34" cy="26" rx="16" ry="10" fill="#78909c" opacity="0.6"/><ellipse cx="22" cy="30" rx="12" ry="8" fill="#90a4ae"/><ellipse cx="28" cy="24" rx="14" ry="10" fill="#cfd8dc" opacity="0.9"/></svg>
              {/if}
            </div>
            <span class="fc-label">{getWeatherLabel(day.weatherCode)}</span>
            <div class="fc-uv" style="color:{uvColor(day.uvIndex)}">UV {day.uvIndex} · {uvLabel(day.uvIndex)}</div>
            <div class="fc-temps">
              <span class="fc-hi">{day.tempMax}°</span>
              <div class="fc-bar-track"><div class="fc-bar-fill" style="width:{Math.min(100,Math.max(15,((day.tempMax-15)/20)*100))}%"></div></div>
              <span class="fc-lo">{day.tempMin}°</span>
            </div>
          </div>
        {/each}
      </div>

      <!-- HISTORY -->
      <div class="block-label">Past 7 Days</div>
      <div class="hist-list">
        {#each history as d, i}
          <div class="hist-row" style="animation-delay:{i*0.05}s">
            <div class="hist-date">
              <span class="hist-wd">{formatDate(d.date, { weekday: 'short' })}</span>
              <span class="hist-md">{formatDate(d.date, { month: 'short', day: 'numeric' })}</span>
            </div>
            <div class="hist-ico">
              {#if getWeatherType(d.weatherCode) === 'clear'}
                <svg viewBox="0 0 28 28" width="26" height="26"><defs><radialGradient id="hic{i}"><stop offset="0%" stop-color="#fff176"/><stop offset="100%" stop-color="#ffb300"/></radialGradient></defs>{#each Array(6) as _,j}<line x1="14" y1="4" x2="14" y2="2" stroke="#ffd54f" stroke-width="1.5" stroke-linecap="round" transform="rotate({j*60} 14 14)"/>{/each}<circle cx="14" cy="14" r="6" fill="url(#hic{i})"/></svg>
              {:else if getWeatherType(d.weatherCode) === 'partly-cloudy'}
                <svg viewBox="0 0 28 28" width="26" height="26"><circle cx="10" cy="12" r="6" fill="#ffb300" opacity="0.9"/><ellipse cx="19" cy="16" rx="7" ry="5" fill="#cfd8dc"/><ellipse cx="13" cy="18" rx="5" ry="4" fill="#eceff1"/></svg>
              {:else if getWeatherType(d.weatherCode) === 'rain' || getWeatherType(d.weatherCode) === 'drizzle'}
                <svg viewBox="0 0 28 28" width="26" height="26"><ellipse cx="17" cy="11" rx="9" ry="6" fill="#546e7a" opacity="0.7"/><ellipse cx="11" cy="14" rx="6" ry="4" fill="#607d8b"/>{#each [[8,20],[14,23],[20,20]] as [x,y]}<ellipse cx={x} cy={y} rx="2" ry="4" fill="#64b5f6" opacity="0.85"/>{/each}</svg>
              {:else if getWeatherType(d.weatherCode) === 'thunder'}
                <svg viewBox="0 0 28 28" width="26" height="26"><ellipse cx="17" cy="11" rx="9" ry="6" fill="#37474f"/><ellipse cx="10" cy="14" rx="6" ry="4" fill="#263238"/><polygon points="14,17 10,24 13,24 9,30 19,22 15,22 17,17" fill="#ffd600"/></svg>
              {:else}
                <svg viewBox="0 0 28 28" width="26" height="26"><ellipse cx="17" cy="13" rx="9" ry="6" fill="#78909c" opacity="0.6"/><ellipse cx="11" cy="16" rx="6" ry="4" fill="#90a4ae"/></svg>
              {/if}
            </div>
            <span class="hist-cond">{getWeatherLabel(d.weatherCode)}</span>
            <div class="hist-bar-group">
              <span class="hist-lo-val">{d.tempMin}°</span>
              <div class="hist-track">
                <div class="hist-fill" style="margin-left:{Math.max(0,((d.tempMin-15)/20)*100)}%;width:{Math.min(100,Math.max(8,((d.tempMax-d.tempMin)/20)*100))}%"></div>
              </div>
              <span class="hist-hi-val">{d.tempMax}°</span>
            </div>
          </div>
        {/each}
      </div>

      <div class="credit">Open-Meteo · No API key · Free & open source</div>
    {/if}
  </main>
</div>

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) {
    font-family: 'Outfit', sans-serif;
    background: #080c14;
    color: #dce5f0;
    -webkit-font-smoothing: antialiased;
  }

  /* ── App Shell ── */
  .app {
    min-height: 100vh;
    position: relative;
    overflow: hidden;
    background: #080c14;
    transition: background 1.4s ease;
  }
  .app.day   { background: #071628; }
  .app.night { background: #03070f; }

  .bg-mesh {
    position: fixed; inset: 0; z-index: 0; pointer-events: none;
    background:
      radial-gradient(ellipse 70% 50% at 85% 5%,  rgba(14,80,140,0.2) 0%, transparent 65%),
      radial-gradient(ellipse 55% 45% at 5% 75%,   rgba(6,50,100,0.16) 0%, transparent 65%),
      radial-gradient(ellipse 40% 35% at 50% 100%, rgba(4,30,60,0.22)  0%, transparent 65%);
    animation: meshDrift 20s ease-in-out infinite alternate;
  }
  .app.night .bg-mesh {
    background:
      radial-gradient(ellipse 60% 40% at 80% 5%,  rgba(30,20,80,0.3)  0%, transparent 65%),
      radial-gradient(ellipse 50% 40% at 10% 80%, rgba(10,10,50,0.2)  0%, transparent 65%);
  }
  @keyframes meshDrift {
    from { opacity:.85; transform: scale(1); }
    to   { opacity:1;   transform: scale(1.04) translateY(-10px); }
  }
  .grain {
    position: fixed; inset: 0; z-index: 0; pointer-events: none; opacity: 0.028;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='300' height='300'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='300' height='300' filter='url(%23n)'/%3E%3C/svg%3E");
    background-size: 250px;
  }

  /* ── Stars ── */
  .stars { position: fixed; inset: 0; z-index: 0; pointer-events: none; }
  .star {
    position: absolute; background: white; border-radius: 50%;
    animation: twinkle 3s ease-in-out infinite;
  }
  @keyframes twinkle { 0%,100%{opacity:.15} 50%{opacity:.9} }

  /* ── Rain particles ── */
  .rain-wrap {
    position: fixed; inset: 0; z-index: 0; pointer-events: none; overflow: hidden;
  }
  .raindrop {
    position: absolute; top: -10px; width: 1.5px; height: 18px;
    background: linear-gradient(to bottom, transparent, rgba(120,200,255,0.5));
    border-radius: 2px;
    animation: rainFall 0.9s linear infinite;
  }
  @keyframes rainFall { to { transform: translateY(110vh) translateX(-10px); } }

  /* ── Layout ── */
  .wrap {
    position: relative; z-index: 1;
    max-width: 480px; margin: 0 auto;
    padding: 2rem 1.25rem 4rem;
  }

  /* ── States ── */
  .state-center {
    display: flex; flex-direction: column; align-items: center;
    justify-content: center; min-height: 85vh; gap: 1rem; text-align: center;
  }
  .pulse-ring { width: 64px; height: 64px; border-radius: 50%; border: 2px solid rgba(255,255,255,0.08); animation: pulseRing 1.8s ease-out infinite; position: absolute; }
  .pulse-dot  { width: 18px; height: 18px; border-radius: 50%; background: rgba(100,180,255,0.6); animation: pulseDot 1.8s ease-in-out infinite; margin-bottom: 1.5rem; }
  @keyframes pulseRing { 0%{transform:scale(.6);opacity:.8} 100%{transform:scale(2.4);opacity:0} }
  @keyframes pulseDot  { 0%,100%{transform:scale(1);opacity:.6} 50%{transform:scale(1.3);opacity:1} }
  .state-title { font-size: 1.1rem; font-weight: 500; color: rgba(255,255,255,0.65); }
  .state-sub   { font-size: 0.78rem; font-weight: 300; color: rgba(255,255,255,0.28); letter-spacing: 0.06em; }
  .err-icon { color: #ef9a9a; margin-bottom: 0.25rem; }
  .btn-retry { margin-top: 0.5rem; background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); color: rgba(255,255,255,0.7); padding: 0.55rem 1.8rem; border-radius: 100px; cursor: pointer; font-family: 'Outfit', sans-serif; font-size: 0.85rem; font-weight: 400; letter-spacing: 0.04em; transition: all 0.2s; }
  .btn-retry:hover { background: rgba(255,255,255,0.1); color: white; }

  /* ── Top Bar ── */
  .topbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; animation: fadeUp 0.5s ease both; }
  .location-btn { display: flex; align-items: center; gap: 0.35rem; background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.1); border-radius: 100px; padding: 0.38rem 0.75rem; cursor: pointer; font-family: 'Outfit', sans-serif; font-size: 0.78rem; font-weight: 400; color: rgba(255,255,255,0.55); letter-spacing: 0.03em; transition: all 0.2s; }
  .location-btn:hover { background: rgba(255,255,255,0.1); color: rgba(255,255,255,0.8); }
  .topbar-right { display: flex; align-items: center; gap: 0.6rem; }
  .updated-badge { font-size: 0.68rem; font-weight: 300; color: rgba(255,255,255,0.22); letter-spacing: 0.04em; }
  .refresh-btn { background: none; border: none; cursor: pointer; color: rgba(255,255,255,0.3); padding: 0.2rem; display: flex; transition: color 0.2s, transform 0.4s; }
  .refresh-btn:hover { color: rgba(255,255,255,0.7); transform: rotate(180deg); }

  /* ── Search Panel ── */
  .search-panel { margin-bottom: 1.25rem; animation: fadeUp 0.3s ease both; }
  .search-input-wrap { display: flex; align-items: center; gap: 0.6rem; background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 14px; padding: 0.7rem 1rem; }
  .search-input { background: none; border: none; outline: none; font-family: 'Outfit', sans-serif; font-size: 0.9rem; font-weight: 300; color: rgba(255,255,255,0.8); flex: 1; }
  .search-input::placeholder { color: rgba(255,255,255,0.25); }
  .search-spinner { width: 14px; height: 14px; border: 2px solid rgba(255,255,255,0.1); border-top-color: rgba(100,200,255,0.7); border-radius: 50%; animation: spin 0.7s linear infinite; flex-shrink: 0; }
  @keyframes spin { to { transform: rotate(360deg); } }
  .search-results { margin-top: 0.4rem; background: rgba(10,16,28,0.95); border: 1px solid rgba(255,255,255,0.1); border-radius: 14px; overflow: hidden; backdrop-filter: blur(20px); }
  .search-result-item { display: flex; justify-content: space-between; align-items: center; width: 100%; padding: 0.75rem 1rem; background: none; border: none; cursor: pointer; font-family: 'Outfit', sans-serif; color: rgba(255,255,255,0.7); transition: background 0.15s; border-bottom: 1px solid rgba(255,255,255,0.05); }
  .search-result-item:last-child { border-bottom: none; }
  .search-result-item:hover { background: rgba(255,255,255,0.06); }
  .sr-name   { font-size: 0.88rem; font-weight: 500; color: rgba(255,255,255,0.85); }
  .sr-region { font-size: 0.74rem; font-weight: 300; color: rgba(255,255,255,0.35); }

  /* ── Hero ── */
  .hero { display: flex; align-items: center; justify-content: space-between; margin-bottom: 1.5rem; gap: 1rem; animation: fadeUp 0.55s 0.05s ease both; }
  .hero-icon { flex-shrink: 0; filter: drop-shadow(0 12px 32px rgba(0,80,180,0.35)); }
  .anim-float { animation: floatIcon 5s ease-in-out infinite; }
  @keyframes floatIcon { 0%,100%{transform:translateY(0) rotate(-1deg)} 50%{transform:translateY(-10px) rotate(1deg)} }
  .hero-info { text-align: right; }
  .hero-temp { display: flex; align-items: flex-start; justify-content: flex-end; line-height: 1; gap: 2px; }
  .temp-num  { font-size: 6.5rem; font-weight: 800; color: #ffffff; letter-spacing: -0.04em; line-height: 1; }
  .temp-deg  { font-size: 2rem; font-weight: 200; color: rgba(255,255,255,0.38); padding-top: 0.65rem; }
  .hero-cond { font-size: 1rem; font-weight: 400; color: rgba(255,255,255,0.52); margin-top: 0.25rem; letter-spacing: 0.02em; }
  .feels-like { font-size: 0.78rem; font-weight: 300; color: rgba(255,255,255,0.32); margin-top: 0.2rem; letter-spacing: 0.02em; }
  .hero-range { display: flex; justify-content: flex-end; align-items: center; gap: 0.4rem; margin-top: 0.4rem; font-size: 0.8rem; font-weight: 300; }
  .rh  { color: rgba(255,180,80,0.7); }
  .rl  { color: rgba(80,180,255,0.7); }
  .rdiv { color: rgba(255,255,255,0.18); }

  /* ── Pills ── */
  .pills { display: flex; gap: 0.55rem; margin-bottom: 1.25rem; animation: fadeUp 0.55s 0.1s ease both; }
  .pill { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 0.28rem; padding: 0.8rem 0.4rem; background: rgba(255,255,255,0.04); border: 1px solid rgba(255,255,255,0.07); border-radius: 16px; color: rgba(255,255,255,0.35); transition: background 0.2s; }
  .pill:hover { background: rgba(255,255,255,0.07); }
  .pill-val  { font-size: 0.92rem; font-weight: 600; color: rgba(255,255,255,0.85); }
  .pill-lbl  { font-size: 0.6rem; font-weight: 300; text-transform: uppercase; letter-spacing: 0.08em; text-align: center; }

  /* ── Cards ── */
  .card { background: rgba(255,255,255,0.04); border: 1px solid rgba(255,255,255,0.07); border-radius: 20px; padding: 1.1rem 1.15rem; margin-bottom: 1rem; animation: fadeUp 0.5s 0.12s ease both; }
  .card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.85rem; }
  .card-title  { font-size: 0.68rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.13em; color: rgba(255,255,255,0.28); }

  /* ── Sparkline ── */
  .spark-range { font-size: 0.78rem; font-weight: 500; color: rgba(100,200,255,0.7); }
  .spark-wrap  { position: relative; }
  .spark-labels { display: flex; justify-content: space-between; margin-top: 0.35rem; padding: 0 2px; }
  .spark-labels span { font-size: 0.62rem; font-weight: 300; color: rgba(255,255,255,0.22); }


  .sun-card {}
  .sun-pct { font-size: 0.72rem; font-weight: 300; color: rgba(255,210,80,0.5); }
  .sun-arc-wrap { position: relative; }
  .sun-times { display: flex; justify-content: space-between; padding: 0 0.25rem; margin-top: -0.25rem; }
  .sun-time  { display: flex; align-items: center; gap: 0.45rem; }
  .sun-time > div { display: flex; flex-direction: column; }
  .sun-label { font-size: 0.62rem; font-weight: 300; text-transform: uppercase; letter-spacing: 0.08em; color: rgba(255,255,255,0.25); }
  .sun-val   { font-size: 0.85rem; font-weight: 500; color: rgba(255,220,100,0.85); }

  .block-label { font-size: 0.67rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.14em; color: rgba(255,255,255,0.22); margin-bottom: 0.8rem; padding-left: 2px; }


  .forecast-row { display: grid; grid-template-columns: 1fr 1fr; gap: 0.65rem; margin-bottom: 1.75rem; }
  .fc-card { background: rgba(255,255,255,0.04); border: 1px solid rgba(255,255,255,0.07); border-radius: 20px; padding: 1.1rem 0.95rem; display: flex; flex-direction: column; gap: 0.18rem; animation: fadeUp 0.5s ease both; transition: transform 0.2s, background 0.2s; }
  .fc-card:hover { transform: translateY(-3px); background: rgba(255,255,255,0.07); }
  .fc-day   { font-size: 0.9rem; font-weight: 600; color: rgba(255,255,255,0.78); }
  .fc-dt    { font-size: 0.7rem; font-weight: 300; color: rgba(255,255,255,0.28); }
  .fc-icon  { margin: 0.55rem 0 0.2rem; filter: drop-shadow(0 4px 10px rgba(0,0,0,0.4)); }
  .fc-label { font-size: 0.74rem; font-weight: 300; color: rgba(255,255,255,0.38); }
  .fc-uv    { font-size: 0.68rem; font-weight: 400; margin-top: 0.1rem; }
  .fc-temps { display: flex; align-items: center; gap: 0.4rem; margin-top: 0.55rem; }
  .fc-hi    { font-size: 0.82rem; font-weight: 600; color: rgba(255,180,80,0.85); }
  .fc-lo    { font-size: 0.82rem; font-weight: 300; color: rgba(80,180,255,0.65); }
  .fc-bar-track { flex: 1; height: 3px; border-radius: 2px; background: rgba(255,255,255,0.08); overflow: hidden; }
  .fc-bar-fill  { height: 100%; border-radius: 2px; background: linear-gradient(90deg, #4fc3f7, #ffb74d); transition: width 0.9s ease; }

  .hist-list { display: flex; flex-direction: column; gap: 0.38rem; }
  .hist-row  { display: grid; grid-template-columns: 56px 28px 1fr auto; align-items: center; gap: 0.7rem; background: rgba(255,255,255,0.025); border: 1px solid rgba(255,255,255,0.05); border-radius: 14px; padding: 0.7rem 0.95rem; animation: fadeUp 0.45s ease both; transition: background 0.2s; }
  .hist-row:hover { background: rgba(255,255,255,0.05); }
  .hist-date { display: flex; flex-direction: column; }
  .hist-wd   { font-size: 0.62rem; font-weight: 400; text-transform: uppercase; letter-spacing: 0.07em; color: rgba(255,255,255,0.28); }
  .hist-md   { font-size: 0.8rem; font-weight: 500; color: rgba(255,255,255,0.6); }
  .hist-ico  { filter: drop-shadow(0 2px 5px rgba(0,0,0,0.35)); }
  .hist-cond { font-size: 0.76rem; font-weight: 300; color: rgba(255,255,255,0.38); }
  .hist-bar-group { display: flex; align-items: center; gap: 0.38rem; }
  .hist-lo-val { font-size: 0.7rem; font-weight: 400; color: rgba(80,180,255,0.6); width: 22px; text-align: right; }
  .hist-hi-val { font-size: 0.7rem; font-weight: 400; color: rgba(255,180,80,0.6); width: 22px; }
  .hist-track { width: 50px; height: 3px; border-radius: 2px; background: rgba(255,255,255,0.07); position: relative; overflow: hidden; }
  .hist-fill  { position: absolute; top: 0; height: 100%; background: linear-gradient(90deg,#4fc3f7,#ffb74d); border-radius: 2px; transition: all 0.9s ease; }


  .credit { text-align: center; margin-top: 2rem; font-size: 0.64rem; font-weight: 300; color: rgba(255,255,255,0.13); letter-spacing: 0.06em; }

  @keyframes fadeUp { from{opacity:0;transform:translateY(14px)} to{opacity:1;transform:translateY(0)} }
</style>