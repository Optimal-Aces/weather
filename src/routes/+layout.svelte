<script lang="ts">
  import { onMount } from 'svelte';

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
  let timeOfDay: 'dawn' | 'morning' | 'afternoon' | 'evening' | 'night' = 'morning';

  function getTimeOfDay(): 'dawn' | 'morning' | 'afternoon' | 'evening' | 'night' {
    const h = new Date().getHours();
    if (h >= 5  && h < 8)  return 'dawn';
    if (h >= 8  && h < 12) return 'morning';
    if (h >= 12 && h < 17) return 'afternoon';
    if (h >= 17 && h < 20) return 'evening';
    return 'night';
  }

  const themes = {
    dawn: {
      bg: 'linear-gradient(160deg, #1a0533 0%, #6b2d6b 35%, #f4845f 65%, #ffd580 100%)',
      orb1: 'rgba(244,132,95,0.35)', orb2: 'rgba(107,45,107,0.3)', orb3: 'rgba(255,213,128,0.25)',
      textPrimary: '#fff8f0', textSecondary: 'rgba(255,240,220,0.65)', textMuted: 'rgba(255,220,180,0.4)',
      cardBg: 'rgba(255,255,255,0.08)', cardBorder: 'rgba(255,200,150,0.15)', pillBg: 'rgba(255,255,255,0.08)',
      accent: '#ffd580', tempColor: '#fff8f0', label: 'Dawn',
    },
    morning: {
      bg: 'linear-gradient(160deg, #ffecd2 0%, #fcb69f 30%, #ffeaa7 60%, #dff9fb 100%)',
      orb1: 'rgba(255,180,100,0.3)', orb2: 'rgba(255,220,120,0.25)', orb3: 'rgba(255,255,200,0.2)',
      textPrimary: '#2d1b00', textSecondary: 'rgba(60,30,0,0.65)', textMuted: 'rgba(80,40,0,0.45)',
      cardBg: 'rgba(255,255,255,0.35)', cardBorder: 'rgba(255,180,80,0.25)', pillBg: 'rgba(255,255,255,0.35)',
      accent: '#e67e00', tempColor: '#2d1b00', label: 'Morning',
    },
    afternoon: {
      bg: 'linear-gradient(180deg, #74c8f5 0%, #3aaee0 30%, #1a8cc7 60%, #0d5fa0 100%)',
      orb1: 'rgba(255,255,255,0.18)', orb2: 'rgba(100,200,255,0.15)', orb3: 'rgba(20,120,200,0.2)',
      textPrimary: '#ffffff', textSecondary: 'rgba(220,245,255,0.75)', textMuted: 'rgba(180,230,255,0.5)',
      cardBg: 'rgba(255,255,255,0.18)', cardBorder: 'rgba(255,255,255,0.28)', pillBg: 'rgba(255,255,255,0.18)',
      accent: '#ffffff', tempColor: '#ffffff', label: 'Afternoon',
    },
    evening: {
      bg: 'linear-gradient(160deg, #0d0221 0%, #3d1a6e 25%, #c0392b 55%, #f39c12 80%, #f9ca24 100%)',
      orb1: 'rgba(243,156,18,0.35)', orb2: 'rgba(192,57,43,0.3)', orb3: 'rgba(61,26,110,0.4)',
      textPrimary: '#fff8ee', textSecondary: 'rgba(255,235,200,0.65)', textMuted: 'rgba(255,210,150,0.4)',
      cardBg: 'rgba(255,255,255,0.08)', cardBorder: 'rgba(255,160,60,0.2)', pillBg: 'rgba(255,255,255,0.08)',
      accent: '#f9ca24', tempColor: '#fff8ee', label: 'Evening',
    },
    night: {
      bg: 'linear-gradient(160deg, #03070f 0%, #071628 50%, #0a1628 100%)',
      orb1: 'rgba(14,80,140,0.22)', orb2: 'rgba(6,50,100,0.18)', orb3: 'rgba(4,30,60,0.25)',
      textPrimary: '#dce5f0', textSecondary: 'rgba(180,210,240,0.6)', textMuted: 'rgba(120,160,200,0.38)',
      cardBg: 'rgba(255,255,255,0.04)', cardBorder: 'rgba(255,255,255,0.07)', pillBg: 'rgba(255,255,255,0.04)',
      accent: '#4fc3f7', tempColor: '#ffffff', label: 'Night',
    },
  };

  $: theme = themes[timeOfDay];
  $: isLight = timeOfDay === 'morning';
  $: isNight = timeOfDay === 'night';
  $: isDawn = timeOfDay === 'dawn';
  $: isEvening = timeOfDay === 'evening';
  $: isAfternoon = timeOfDay === 'afternoon';

  // Afternoon cloud layers — each has a different size, speed, opacity, y position
  const afternoonClouds = [
    { id: 1, y: 6,  w: 260, h: 70,  speed: 38, opacity: 0.92, delay: 0   },
    { id: 2, y: 18, w: 180, h: 52,  speed: 55, opacity: 0.75, delay: -12 },
    { id: 3, y: 10, w: 320, h: 80,  speed: 62, opacity: 0.6,  delay: -25 },
    { id: 4, y: 28, w: 140, h: 44,  speed: 44, opacity: 0.5,  delay: -8  },
    { id: 5, y: 4,  w: 200, h: 60,  speed: 70, opacity: 0.4,  delay: -35 },
    { id: 6, y: 22, w: 240, h: 65,  speed: 50, opacity: 0.65, delay: -18 },
  ];

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
    return new Date(iso).toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit', hour12: true });
  }
  function sunProgress(sunriseIso: string, sunsetIso: string): number {
    const now = Date.now();
    const rise = new Date(sunriseIso).getTime();
    const set  = new Date(sunsetIso).getTime();
    if (now <= rise) return 0;
    if (now >= set)  return 1;
    return (now - rise) / (set - rise);
  }
  function sparklinePath(points: HourlyPoint[], w: number, h: number): string {
    if (!points.length) return '';
    const temps = points.map(p => p.temp);
    const min = Math.min(...temps), max = Math.max(...temps), range = max - min || 1;
    const xs = points.map((_, i) => (i / (points.length - 1)) * w);
    const ys = points.map(p => h - ((p.temp - min) / range) * (h - 8) - 4);
    let d = `M ${xs[0]} ${ys[0]}`;
    for (let i = 1; i < xs.length; i++) {
      const cpx = (xs[i-1] + xs[i]) / 2;
      d += ` C ${cpx} ${ys[i-1]}, ${cpx} ${ys[i]}, ${xs[i]} ${ys[i]}`;
    }
    return d;
  }
  function sparklineArea(points: HourlyPoint[], w: number, h: number): string {
    const base = sparklinePath(points, w, h);
    if (!base) return '';
    return `${base} L ${w} ${h} L 0 ${h} Z`;
  }

  let searchQuery = '';
  let searchResults: any[] = [];
  let searchLoading = false;
  let showSearch = false;
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

  async function fetchWeatherData() {
    loading = true; error = '';
    timeOfDay = getTimeOfDay();
    try {
      const today = new Date();
      const todayStr = today.toISOString().split('T')[0];
      const startHistory = new Date(today.getTime() - 7 * 86400000).toISOString().split('T')[0];

      const [histRes, foreRes] = await Promise.all([
        fetch(`https://archive-api.open-meteo.com/v1/archive?latitude=${LAT}&longitude=${LON}&start_date=${startHistory}&end_date=${todayStr}&daily=temperature_2m_max,temperature_2m_min,weathercode,windspeed_10m_max,uv_index_max,sunrise,sunset&hourly=relativehumidity_2m&timezone=Asia%2FManila`),
        fetch(`https://api.open-meteo.com/v1/forecast?latitude=${LAT}&longitude=${LON}&daily=temperature_2m_max,temperature_2m_min,weathercode,windspeed_10m_max,uv_index_max,sunrise,sunset&hourly=temperature_2m,relativehumidity_2m,apparent_temperature&current_weather=true&timezone=Asia%2FManila&forecast_days=3`)
      ]);
      if (!histRes.ok) throw new Error(`History API ${histRes.status}`);
      if (!foreRes.ok) throw new Error(`Forecast API ${foreRes.status}`);

      const [hd, fd] = await Promise.all([histRes.json(), foreRes.json()]);
      const ti = Math.max(0, fd.daily.time.indexOf(todayStr));
      const nowHour = today.getHours();
      const hourlyTimes: string[] = fd.hourly.time;
      const curHourIdx = hourlyTimes.findIndex(t => t.startsWith(todayStr + 'T' + String(nowHour).padStart(2,'0')));
      const feelsLikeCurrent = curHourIdx >= 0 ? Math.round(fd.hourly.apparent_temperature[curHourIdx]) : Math.round(fd.current_weather.temperature);

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

      hourlyToday = hourlyTimes
        .map((t, i) => ({ t, temp: Math.round(fd.hourly.temperature_2m[i]) }))
        .filter(({ t }) => t.startsWith(todayStr))
        .map(({ t, temp }) => ({ hour: new Date(t).getHours(), temp }));

      forecast = fd.daily.time.slice(1, 3).map((date: string, i: number) => ({
        date, tempMax: Math.round(fd.daily.temperature_2m_max[i+1]),
        tempMin: Math.round(fd.daily.temperature_2m_min[i+1]),
        temp: Math.round((fd.daily.temperature_2m_max[i+1] + fd.daily.temperature_2m_min[i+1]) / 2),
        feelsLike: 0,
        humidity: avgHumidity(fd.hourly.time, fd.hourly.relativehumidity_2m, date),
        windSpeed: Math.round((fd.daily.windspeed_10m_max[i+1] / 3.6) * 10) / 10,
        weatherCode: fd.daily.weathercode[i+1],
        uvIndex: Math.round(fd.daily.uv_index_max[i+1] ?? 0),
        sunrise: fd.daily.sunrise[i+1], sunset: fd.daily.sunset[i+1], isDay: 1,
      }));

      history = hd.daily.time.slice(0, -1).map((date: string, i: number) => ({
        date, tempMax: Math.round(hd.daily.temperature_2m_max[i]),
        tempMin: Math.round(hd.daily.temperature_2m_min[i]),
        temp: Math.round((hd.daily.temperature_2m_max[i] + hd.daily.temperature_2m_min[i]) / 2),
        feelsLike: 0, humidity: 0, windSpeed: 0,
        weatherCode: hd.daily.weathercode[i],
        uvIndex: Math.round(hd.daily.uv_index_max?.[i] ?? 0),
        sunrise: hd.daily.sunrise[i], sunset: hd.daily.sunset[i], isDay: 1,
      })).reverse();

      lastUpdated = new Date().toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit' });
    } catch (e: any) {
      error = e.message || 'Failed to load weather data.';
    } finally {
      loading = false;
    }
  }

  onMount(fetchWeatherData);

  $: sunProg = current ? sunProgress(current.sunrise, current.sunset) : 0;
  $: sunArcX = 16 + sunProg * 168;
  $: sunArcY = 48 - Math.sin(sunProg * Math.PI) * 38;
</script>

<svelte:head>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@200;300;400;500;600;700;800&display=swap" rel="stylesheet">
</svelte:head>

<div class="app"
  class:is-afternoon={isAfternoon}
  class:is-morning={isLight}
  class:is-night={isNight}
  class:is-dawn={isDawn}
  class:is-evening={isEvening}
  style="background:{theme.bg}; --text-primary:{theme.textPrimary}; --text-secondary:{theme.textSecondary}; --text-muted:{theme.textMuted}; --card-bg:{theme.cardBg}; --card-border:{theme.cardBorder}; --pill-bg:{theme.pillBg}; --accent:{theme.accent}; --temp-color:{theme.tempColor}; --orb1:{theme.orb1}; --orb2:{theme.orb2}; --orb3:{theme.orb3};">

  <!-- Orbs (non-afternoon) -->
  {#if !isAfternoon}
    <div class="orb orb-1"></div>
    <div class="orb orb-2"></div>
    <div class="orb orb-3"></div>
  {/if}

  <div class="grain"></div>

  <!-- ══ AFTERNOON: sky + animated clouds ══ -->
  {#if isAfternoon}
    <!-- Top sun halo -->
    <div class="aft-sun-halo"></div>

    <!-- Cloud layers -->
    <div class="clouds-stage" aria-hidden="true">
      {#each afternoonClouds as cloud}
        <div
          class="cloud-wrap"
          style="
            top:{cloud.y}%;
            animation-duration:{cloud.speed}s;
            animation-delay:{cloud.delay}s;
            opacity:{cloud.opacity};
          "
        >
          <!-- Each cloud is built from overlapping ellipses for a fluffy look -->
          <svg
            viewBox="0 0 {cloud.w} {cloud.h}"
            width="{cloud.w}"
            height="{cloud.h}"
            style="overflow:visible"
          >
            <defs>
              <linearGradient id="cg{cloud.id}" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" stop-color="white" stop-opacity="1"/>
                <stop offset="100%" stop-color="#d6eeff" stop-opacity="0.85"/>
              </linearGradient>
              <filter id="cshadow{cloud.id}">
                <feDropShadow dx="0" dy="4" stdDeviation="6" flood-color="rgba(30,100,180,0.15)"/>
              </filter>
            </defs>
            <g filter="url(#cshadow{cloud.id})">
              <!-- Base puff -->
              <ellipse cx="{cloud.w*0.5}" cy="{cloud.h*0.72}" rx="{cloud.w*0.45}" ry="{cloud.h*0.3}" fill="url(#cg{cloud.id})"/>
              <!-- Left bump -->
              <ellipse cx="{cloud.w*0.25}" cy="{cloud.h*0.55}" rx="{cloud.w*0.2}" ry="{cloud.h*0.32}" fill="url(#cg{cloud.id})"/>
              <!-- Center top bump -->
              <ellipse cx="{cloud.w*0.48}" cy="{cloud.h*0.38}" rx="{cloud.w*0.26}" ry="{cloud.h*0.38}" fill="url(#cg{cloud.id})"/>
              <!-- Right bump -->
              <ellipse cx="{cloud.w*0.72}" cy="{cloud.h*0.52}" rx="{cloud.w*0.22}" ry="{cloud.h*0.3}" fill="url(#cg{cloud.id})"/>
              <!-- Far right small -->
              <ellipse cx="{cloud.w*0.88}" cy="{cloud.h*0.65}" rx="{cloud.w*0.14}" ry="{cloud.h*0.24}" fill="url(#cg{cloud.id})"/>
            </g>
          </svg>
        </div>
      {/each}
    </div>

    <!-- Bottom sky fade -->
    <div class="aft-bottom-fade"></div>
  {/if}

  <!-- Morning sun rays -->
  {#if isLight || isDawn}
    <div class="sun-rays" aria-hidden="true">
      {#each Array(12) as _, i}
        <div class="ray" style="transform:rotate({i*30}deg);animation-delay:{i*0.15}s"></div>
      {/each}
    </div>
  {/if}

  <!-- Night stars -->
  {#if isNight || isDawn}
    <div class="stars" aria-hidden="true">
      {#each Array(55) as _, i}
        <div class="star" style="left:{Math.random()*100}%;top:{Math.random()*55}%;width:{Math.random()*2+0.8}px;height:{Math.random()*2+0.8}px;animation-delay:{Math.random()*5}s;animation-duration:{2+Math.random()*3}s"></div>
      {/each}
    </div>
  {/if}

  <!-- Evening horizon glow -->
  {#if isEvening}
    <div class="horizon-glow" aria-hidden="true"></div>
  {/if}

  <!-- Rain -->
  {#if current && (getWeatherType(current.weatherCode) === 'rain' || getWeatherType(current.weatherCode) === 'drizzle')}
    <div class="rain-wrap" aria-hidden="true">
      {#each Array(28) as _, i}
        <div class="raindrop" style="left:{i*3.6+Math.random()*3}%;animation-delay:{Math.random()*1.5}s;animation-duration:{0.6+Math.random()*0.5}s;opacity:{0.25+Math.random()*0.3}"></div>
      {/each}
    </div>
  {/if}

  <main class="wrap">
    {#if loading}
      <div class="state-center">
        <div class="pulse-ring"></div>
        <div class="pulse-dot"></div>
        <p class="state-title">Loading weather</p>
        <p class="state-sub">{cityName}</p>
      </div>

    {:else if error}
      <div class="state-center">
        <div class="err-icon">
          <svg viewBox="0 0 24 24" width="34" height="34" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round">
            <path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/>
            <line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/>
          </svg>
        </div>
        <p class="state-title">{error}</p>
        <button class="btn-retry" on:click={fetchWeatherData}>Retry</button>
      </div>

    {:else if current}

      <!-- TOP BAR -->
      <div class="topbar">
        <button class="location-btn" on:click={() => { showSearch = !showSearch; searchQuery=''; searchResults=[]; }}>
          <svg viewBox="0 0 24 24" width="13" height="13" fill="currentColor">
            <path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5c-1.38 0-2.5-1.12-2.5-2.5s1.12-2.5 2.5-2.5 2.5 1.12 2.5 2.5-1.12 2.5-2.5 2.5z"/>
          </svg>
          <span>{cityName}</span>
          <svg viewBox="0 0 24 24" width="11" height="11" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="6 9 12 15 18 9"/></svg>
        </button>
        <div class="topbar-right">
          <span class="time-badge">{theme.label}</span>
          {#if lastUpdated}<span class="updated-badge">{lastUpdated}</span>{/if}
          <button class="refresh-btn" on:click={fetchWeatherData} title="Refresh">
            <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
              <polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/>
            </svg>
          </button>
        </div>
      </div>

      <!-- SEARCH -->
      {#if showSearch}
        <div class="search-panel">
          <div class="search-input-wrap">
            <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" opacity="0.5"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
            <input class="search-input" type="text" placeholder="Search city…" bind:value={searchQuery} on:input={onSearchInput} autofocus/>
            {#if searchLoading}<div class="search-spinner"></div>{/if}
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
          {#if (timeOfDay === 'morning' || timeOfDay === 'dawn') && getWeatherType(current.weatherCode) === 'clear'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs>
                <radialGradient id="mornSun" cx="50%" cy="45%" r="55%">
                  <stop offset="0%" stop-color="#fffde7"/><stop offset="40%" stop-color="#ffd54f"/>
                  <stop offset="100%" stop-color="{timeOfDay==='dawn'?'#f4845f':'#fb8c00'}"/>
                </radialGradient>
                <filter id="mornGlow"><feGaussianBlur stdDeviation="7" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#mornGlow)">
                {#each Array(16) as _, i}
                  <line x1="80" y1="{i%2===0?18:23}" x2="80" y2="6" stroke="{timeOfDay==='dawn'?'#f4845f':'#ffd54f'}" stroke-width="{i%2===0?3.5:2}" stroke-linecap="round" transform="rotate({i*22.5} 80 80)" opacity="{i%2===0?1:0.5}"/>
                {/each}
                <circle cx="80" cy="80" r="40" fill="url(#mornSun)" opacity="0.2"/>
                <circle cx="80" cy="80" r="30" fill="url(#mornSun)"/>
              </g>
            </svg>

          {:else if timeOfDay === 'afternoon' && getWeatherType(current.weatherCode) === 'clear'}
            <!-- Afternoon: crisp white-gold high sun -->
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs>
                <radialGradient id="aftSun" cx="50%" cy="50%" r="50%">
                  <stop offset="0%" stop-color="#ffffff"/>
                  <stop offset="25%" stop-color="#fffde7"/>
                  <stop offset="100%" stop-color="#ffe082"/>
                </radialGradient>
                <filter id="aftGlow"><feGaussianBlur stdDeviation="10" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#aftGlow)">
                <!-- Outer halo -->
                <circle cx="80" cy="80" r="52" fill="white" opacity="0.08"/>
                <circle cx="80" cy="80" r="42" fill="white" opacity="0.1"/>
                {#each Array(16) as _, i}
                  <line x1="80" y1="{i%2===0?16:21}" x2="80" y2="6"
                    stroke="white" stroke-width="{i%2===0?3.5:2}"
                    stroke-linecap="round" transform="rotate({i*22.5} 80 80)"
                    opacity="{i%2===0?0.95:0.5}"/>
                {/each}
                <circle cx="80" cy="80" r="30" fill="url(#aftSun)"/>
              </g>
            </svg>

          {:else if timeOfDay === 'afternoon' && getWeatherType(current.weatherCode) === 'partly-cloudy'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs>
                <radialGradient id="pcAft"><stop offset="0%" stop-color="#ffffff"/><stop offset="100%" stop-color="#ffe082"/></radialGradient>
                <filter id="pcAftGlow"><feGaussianBlur stdDeviation="6" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#pcAftGlow)">
                <circle cx="55" cy="65" r="24" fill="url(#pcAft)" opacity="0.9"/>
                {#each Array(8) as _,i}<line x1="55" y1="{37-(i%2)*3}" x2="55" y2="28" stroke="white" stroke-width="2.5" stroke-linecap="round" transform="rotate({i*45} 55 65)" opacity="{i%2===0?0.9:0.4}"/>{/each}
              </g>
              <ellipse cx="100" cy="98" rx="35" ry="22" fill="rgba(255,255,255,0.7)"/>
              <ellipse cx="75" cy="104" rx="24" ry="18" fill="rgba(255,255,255,0.75)"/>
              <ellipse cx="88" cy="90" rx="28" ry="22" fill="white" opacity="0.95"/>
              <ellipse cx="108" cy="96" rx="22" ry="17" fill="rgba(255,255,255,0.7)"/>
            </svg>

          {:else if timeOfDay === 'evening' && getWeatherType(current.weatherCode) === 'clear'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs>
                <radialGradient id="eveSun" cx="50%" cy="60%" r="55%"><stop offset="0%" stop-color="#fff3e0"/><stop offset="40%" stop-color="#ff8f00"/><stop offset="100%" stop-color="#e64a19"/></radialGradient>
                <filter id="eveGlow"><feGaussianBlur stdDeviation="9" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter>
              </defs>
              <g filter="url(#eveGlow)">
                {#each Array(10) as _, i}<line x1="80" y1="22" x2="80" y2="8" stroke="#ff8f00" stroke-width="3.5" stroke-linecap="round" transform="rotate({i*36} 80 80)" opacity="0.6"/>{/each}
                <circle cx="80" cy="80" r="46" fill="url(#eveSun)" opacity="0.2"/>
                <circle cx="80" cy="80" r="32" fill="url(#eveSun)"/>
              </g>
            </svg>

          {:else if isNight && getWeatherType(current.weatherCode) === 'clear'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs><radialGradient id="moonG" cx="45%" cy="40%" r="55%"><stop offset="0%" stop-color="#fffde7"/><stop offset="60%" stop-color="#fff9c4"/><stop offset="100%" stop-color="#f9a825"/></radialGradient><filter id="moonGlow"><feGaussianBlur stdDeviation="6" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter></defs>
              <g filter="url(#moonGlow)"><circle cx="80" cy="80" r="40" fill="url(#moonG)" opacity="0.15"/><path d="M80 44 A36 36 0 1 0 80 116 A28 28 0 1 1 80 44Z" fill="url(#moonG)"/></g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'partly-cloudy'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs><radialGradient id="sg2" cx="40%" cy="40%" r="60%"><stop offset="0%" stop-color="#fff9c4"/><stop offset="100%" stop-color="#ffb300"/></radialGradient><linearGradient id="cg2" x1="0%" y1="0%" x2="20%" y2="100%"><stop offset="0%" stop-color="#e3f2fd"/><stop offset="100%" stop-color="#90caf9"/></linearGradient><filter id="csh"><feDropShadow dx="0" dy="6" stdDeviation="8" flood-color="rgba(0,80,150,0.3)"/></filter></defs>
              <circle cx="55" cy="65" r="22" fill="url(#sg2)"/>
              {#each Array(8) as _,i}<line x1="55" y1="{38-(i%2)*3}" x2="55" y2="30" stroke="#ffd54f" stroke-width="2.5" stroke-linecap="round" transform="rotate({i*45} 55 65)" opacity="{i%2===0?1:0.45}"/>{/each}
              <g filter="url(#csh)"><ellipse cx="100" cy="98" rx="35" ry="22" fill="url(#cg2)"/><ellipse cx="75" cy="104" rx="24" ry="18" fill="url(#cg2)"/><ellipse cx="88" cy="90" rx="28" ry="22" fill="white" opacity="0.9"/><ellipse cx="108" cy="96" rx="22" ry="17" fill="url(#cg2)"/></g>
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'rain' || getWeatherType(current.weatherCode) === 'drizzle'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs><linearGradient id="rcg" x1="0%" y1="0%" x2="10%" y2="100%"><stop offset="0%" stop-color="#546e7a"/><stop offset="100%" stop-color="#263238"/></linearGradient><filter id="rsh"><feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="rgba(0,50,100,0.35)"/></filter></defs>
              <g filter="url(#rsh)"><ellipse cx="95" cy="72" rx="38" ry="23" fill="url(#rcg)" opacity="0.6"/><ellipse cx="66" cy="78" rx="27" ry="19" fill="url(#rcg)"/><ellipse cx="80" cy="66" rx="32" ry="25" fill="#455a64" opacity="0.95"/><ellipse cx="104" cy="74" rx="24" ry="18" fill="url(#rcg)"/></g>
              {#each [[52,102],[66,110],[80,102],[94,110],[108,102],[66,122],[80,130],[94,122]] as [x,y],i}
                <ellipse cx={x} cy={y} rx="3" ry="6" fill="#64b5f6" opacity="0.85">
                  <animate attributeName="cy" values="{y};{y+14};{y+14}" dur="{0.7+i*0.09}s" repeatCount="indefinite"/>
                  <animate attributeName="opacity" values="0.85;0;0.85" dur="{0.7+i*0.09}s" repeatCount="indefinite"/>
                </ellipse>
              {/each}
            </svg>

          {:else if getWeatherType(current.weatherCode) === 'thunder'}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs><linearGradient id="thcg" x1="0%" y1="0%" x2="0%" y2="100%"><stop offset="0%" stop-color="#37474f"/><stop offset="100%" stop-color="#102027"/></linearGradient><filter id="lglow"><feGaussianBlur stdDeviation="3" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter></defs>
              <ellipse cx="95" cy="68" rx="40" ry="24" fill="url(#thcg)"/><ellipse cx="65" cy="75" rx="28" ry="20" fill="url(#thcg)"/><ellipse cx="80" cy="62" rx="34" ry="26" fill="#455a64"/><ellipse cx="106" cy="72" rx="26" ry="20" fill="url(#thcg)"/>
              <g filter="url(#lglow)"><polygon points="85,94 72,118 82,118 68,146 100,108 88,108 98,94" fill="#ffd600"><animate attributeName="opacity" values="1;0.3;1;0.7;1" dur="2.2s" repeatCount="indefinite"/></polygon></g>
            </svg>

          {:else}
            <svg viewBox="0 0 160 160" width="145" height="145" class="anim-float">
              <defs><linearGradient id="cldg" x1="0%" y1="0%" x2="15%" y2="100%"><stop offset="0%" stop-color="{isLight||isAfternoon?'#e3f2fd':'#cfd8dc'}"/><stop offset="100%" stop-color="{isLight||isAfternoon?'#90caf9':'#78909c'}"/></linearGradient><filter id="csh2"><feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="rgba(0,0,0,0.2)"/></filter></defs>
              <g filter="url(#csh2)"><ellipse cx="95" cy="88" rx="38" ry="24" fill="url(#cldg)" opacity="0.5"/><ellipse cx="66" cy="95" rx="28" ry="20" fill="url(#cldg)"/><ellipse cx="80" cy="82" rx="32" ry="26" fill="white" opacity="0.95"/><ellipse cx="104" cy="92" rx="25" ry="19" fill="url(#cldg)"/></g>
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

      <!-- PILLS -->
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

      <!-- SPARKLINE -->
      {#if hourlyToday.length > 1}
        <div class="card">
          <div class="card-header">
            <span class="card-title">Today's Temperature</span>
            <span class="spark-range">{Math.min(...hourlyToday.map(h=>h.temp))}° – {Math.max(...hourlyToday.map(h=>h.temp))}°C</span>
          </div>
          <svg viewBox="0 0 320 60" preserveAspectRatio="none" width="100%" height="60">
            <defs>
              <linearGradient id="sparkG" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stop-color="{isAfternoon?'rgba(255,255,255,0.5)':theme.accent}" stop-opacity="0.4"/>
                <stop offset="100%" stop-color="{isAfternoon?'rgba(255,255,255,0)':theme.accent}" stop-opacity="0"/>
              </linearGradient>
            </defs>
            <path d={sparklineArea(hourlyToday, 320, 60)} fill="url(#sparkG)"/>
            <path d={sparklinePath(hourlyToday, 320, 60)} fill="none" stroke="{isAfternoon?'rgba(255,255,255,0.9)':theme.accent}" stroke-width="2" stroke-linecap="round"/>
          </svg>
          <div class="spark-labels">
            {#each hourlyToday.filter((_,i) => i % 4 === 0) as pt}
              <span>{pt.hour===0?'12a':pt.hour<12?pt.hour+'a':pt.hour===12?'12p':(pt.hour-12)+'p'}</span>
            {/each}
          </div>
        </div>
      {/if}

      <!-- SUN ARC -->
      <div class="card">
        <div class="card-header">
          <span class="card-title">Daylight</span>
          <span class="sun-pct">{Math.round(sunProg * 100)}% passed</span>
        </div>
        <svg viewBox="0 0 200 78" width="100%" height="82">
          <defs>
            <linearGradient id="arcG" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#fb8c00" stop-opacity="0.5"/>
              <stop offset="50%" stop-color="#ffd54f" stop-opacity="0.95"/>
              <stop offset="100%" stop-color="#ff7043" stop-opacity="0.5"/>
            </linearGradient>
          </defs>
          <path d="M16 68 Q100 4 184 68" fill="none" stroke="rgba(128,128,128,0.15)" stroke-width="2"/>
          <path d="M16 68 Q100 4 184 68" fill="none" stroke="url(#arcG)" stroke-width="2.5"
            stroke-dasharray="220" stroke-dashoffset="{220 - sunProg * 220}" stroke-linecap="round"/>
          <circle cx={sunArcX} cy={sunArcY} r="7" fill="#ffd54f" opacity="0.95">
            <animate attributeName="r" values="7;8.5;7" dur="2.5s" repeatCount="indefinite"/>
          </circle>
          <circle cx={sunArcX} cy={sunArcY} r="13" fill="#ffd54f" opacity="0.15"/>
          <line x1="10" y1="70" x2="190" y2="70" stroke="rgba(128,128,128,0.12)" stroke-width="1"/>
        </svg>
        <div class="sun-times">
          <div class="sun-time">
            <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="#ffd54f" stroke-width="2" stroke-linecap="round"><path d="M17 18a5 5 0 00-10 0"/><line x1="12" y1="2" x2="12" y2="9"/><line x1="4.22" y1="10.22" x2="5.64" y2="11.64"/><line x1="18.36" y1="11.64" x2="19.78" y2="10.22"/></svg>
            <div><span class="sun-label">Sunrise</span><span class="sun-val">{parseTime(current.sunrise)}</span></div>
          </div>
          <div class="sun-time">
            <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="#ff9800" stroke-width="2" stroke-linecap="round"><path d="M17 18a5 5 0 00-10 0"/><line x1="12" y1="9" x2="12" y2="2"/><line x1="4.22" y1="10.22" x2="5.64" y2="11.64"/><line x1="18.36" y1="11.64" x2="19.78" y2="10.22"/></svg>
            <div><span class="sun-label">Sunset</span><span class="sun-val">{parseTime(current.sunset)}</span></div>
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
              <div class="hist-track"><div class="hist-fill" style="margin-left:{Math.max(0,((d.tempMin-15)/20)*100)}%;width:{Math.min(100,Math.max(8,((d.tempMax-d.tempMin)/20)*100))}%"></div></div>
              <span class="hist-hi-val">{d.tempMax}°</span>
            </div>
          </div>
        {/each}
      </div>

      <div class="credit">Open-Meteo · Free & open source</div>
    {/if}
  </main>
</div>

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) { font-family: 'Outfit', sans-serif; background: #080c14; -webkit-font-smoothing: antialiased; }

  .app { min-height: 100vh; position: relative; overflow: hidden; transition: background 2s ease; color: var(--text-primary); }

  /* ── Orbs ── */
  .orb { position: fixed; border-radius: 50%; filter: blur(90px); pointer-events: none; z-index: 0; animation: floatOrb 18s ease-in-out infinite; }
  .orb-1 { width: 500px; height: 500px; background: radial-gradient(circle, var(--orb1) 0%, transparent 70%); top: -150px; right: -100px; animation-duration: 14s; }
  .orb-2 { width: 400px; height: 400px; background: radial-gradient(circle, var(--orb2) 0%, transparent 70%); bottom: 5%; left: -100px; animation-duration: 18s; animation-direction: reverse; }
  .orb-3 { width: 300px; height: 300px; background: radial-gradient(circle, var(--orb3) 0%, transparent 70%); top: 40%; right: 5%; animation-duration: 22s; }
  @keyframes floatOrb { 0%,100%{transform:translate(0,0) scale(1)} 33%{transform:translate(25px,-35px) scale(1.05)} 66%{transform:translate(-20px,25px) scale(0.95)} }

  .grain { position: fixed; inset: 0; z-index: 0; pointer-events: none; opacity: 0.018; background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='300' height='300'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='300' height='300' filter='url(%23n)'/%3E%3C/svg%3E"); background-size: 250px; }

  /* ════════════════════════════════
     AFTERNOON: sky + clouds
  ════════════════════════════════ */
  .aft-sun-halo {
    position: fixed;
    top: -30vh; left: 50%;
    transform: translateX(-50%);
    width: 80vw; height: 80vw;
    max-width: 600px; max-height: 600px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(255,255,240,0.55) 0%, rgba(255,240,180,0.2) 35%, transparent 70%);
    z-index: 0; pointer-events: none;
    animation: haloPulse 8s ease-in-out infinite;
  }
  @keyframes haloPulse { 0%,100%{opacity:.8;transform:translateX(-50%) scale(1)} 50%{opacity:1;transform:translateX(-50%) scale(1.06)} }

  .clouds-stage {
    position: fixed;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    overflow: hidden;
  }

  .cloud-wrap {
    position: absolute;
    /* start off-screen to the right, animate leftward */
    animation: cloudDrift linear infinite;
    /* initial x set via JS animation-delay trick — clouds start at varied positions */
    right: -400px;
    will-change: transform;
  }

  /* Each cloud drifts right-to-left across the full screen + cloud width */
  @keyframes cloudDrift {
    0%   { transform: translateX(calc(100vw + 400px)); }
    100% { transform: translateX(-500px); }
  }

  .aft-bottom-fade {
    position: fixed;
    bottom: 0; left: 0; right: 0;
    height: 40%;
    z-index: 0;
    pointer-events: none;
    background: linear-gradient(to top, rgba(13,95,160,0.7) 0%, rgba(26,140,199,0.3) 50%, transparent 100%);
  }

  /* ── Morning rays ── */
  .sun-rays { position: fixed; top: -40%; left: 50%; transform: translateX(-50%); width: 100vw; height: 180%; z-index: 0; pointer-events: none; }
  .ray { position: absolute; top: 0; left: 50%; width: 2px; height: 100%; background: linear-gradient(to bottom, rgba(255,220,100,0.12), transparent 60%); transform-origin: top center; animation: rayPulse 6s ease-in-out infinite; }
  @keyframes rayPulse { 0%,100%{opacity:.5} 50%{opacity:1} }

  /* ── Stars ── */
  .stars { position: fixed; inset: 0; z-index: 0; pointer-events: none; }
  .star { position: absolute; background: white; border-radius: 50%; animation: twinkle 3s ease-in-out infinite; }
  @keyframes twinkle { 0%,100%{opacity:.1} 50%{opacity:.85} }

  /* ── Evening ── */
  .horizon-glow { position: fixed; bottom: 0; left: 0; right: 0; height: 35vh; z-index: 0; pointer-events: none; background: linear-gradient(to top, rgba(249,202,36,0.18), rgba(230,81,0,0.1), transparent); }

  /* ── Rain ── */
  .rain-wrap { position: fixed; inset: 0; z-index: 0; pointer-events: none; overflow: hidden; }
  .raindrop { position: absolute; top: -10px; width: 1.5px; height: 18px; background: linear-gradient(to bottom, transparent, rgba(120,200,255,0.45)); border-radius: 2px; animation: rainFall 0.9s linear infinite; }
  @keyframes rainFall { to { transform: translateY(110vh) translateX(-10px); } }

  /* ── Layout ── */
  .wrap { position: relative; z-index: 1; max-width: 480px; margin: 0 auto; padding: 2rem 1.25rem 4rem; }

  /* ── States ── */
  .state-center { display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 85vh; gap: 1rem; text-align: center; }
  .pulse-ring { width: 64px; height: 64px; border-radius: 50%; border: 2px solid rgba(128,128,128,0.2); animation: pulseRing 1.8s ease-out infinite; position: absolute; }
  .pulse-dot  { width: 18px; height: 18px; border-radius: 50%; background: var(--accent); opacity: 0.7; animation: pulseDot 1.8s ease-in-out infinite; margin-bottom: 1.5rem; }
  @keyframes pulseRing { 0%{transform:scale(.6);opacity:.8} 100%{transform:scale(2.4);opacity:0} }
  @keyframes pulseDot  { 0%,100%{transform:scale(1);opacity:.6} 50%{transform:scale(1.3);opacity:1} }
  .state-title { font-size: 1.05rem; font-weight: 500; color: var(--text-secondary); }
  .state-sub   { font-size: 0.78rem; font-weight: 300; color: var(--text-muted); letter-spacing: 0.05em; }
  .err-icon { color: #ef9a9a; margin-bottom: 0.25rem; }
  .btn-retry { margin-top: 0.5rem; background: var(--card-bg); border: 1px solid var(--card-border); color: var(--text-secondary); padding: 0.55rem 1.8rem; border-radius: 100px; cursor: pointer; font-family: 'Outfit', sans-serif; font-size: 0.85rem; letter-spacing: 0.04em; transition: all 0.2s; }
  .btn-retry:hover { opacity: 0.8; }

  /* ── Topbar ── */
  .topbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; animation: fadeUp 0.5s ease both; }
  .location-btn { display: flex; align-items: center; gap: 0.35rem; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 100px; padding: 0.38rem 0.8rem; cursor: pointer; font-family: 'Outfit', sans-serif; font-size: 0.78rem; color: var(--text-secondary); transition: opacity 0.2s; }
  .location-btn:hover { opacity: 0.75; }
  .topbar-right { display: flex; align-items: center; gap: 0.55rem; }
  .time-badge   { font-size: 0.68rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.1em; color: var(--accent); opacity: 0.9; }
  .updated-badge { font-size: 0.66rem; font-weight: 300; color: var(--text-muted); letter-spacing: 0.04em; }
  .refresh-btn  { background: none; border: none; cursor: pointer; color: var(--text-muted); padding: 0.2rem; display: flex; transition: color 0.2s, transform 0.4s; }
  .refresh-btn:hover { color: var(--text-primary); transform: rotate(180deg); }

  /* ── Search ── */
  .search-panel { margin-bottom: 1.25rem; animation: fadeUp 0.3s ease both; }
  .search-input-wrap { display: flex; align-items: center; gap: 0.6rem; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 14px; padding: 0.7rem 1rem; }
  .search-input { background: none; border: none; outline: none; font-family: 'Outfit', sans-serif; font-size: 0.9rem; font-weight: 300; color: var(--text-primary); flex: 1; }
  .search-input::placeholder { color: var(--text-muted); }
  .search-spinner { width: 14px; height: 14px; border: 2px solid rgba(128,128,128,0.2); border-top-color: var(--accent); border-radius: 50%; animation: spin 0.7s linear infinite; flex-shrink: 0; }
  @keyframes spin { to { transform: rotate(360deg); } }
  .search-results { margin-top: 0.4rem; background: rgba(10,16,28,0.96); border: 1px solid rgba(255,255,255,0.1); border-radius: 14px; overflow: hidden; backdrop-filter: blur(20px); }
  .search-result-item { display: flex; justify-content: space-between; align-items: center; width: 100%; padding: 0.75rem 1rem; background: none; border: none; cursor: pointer; font-family: 'Outfit', sans-serif; color: rgba(255,255,255,0.7); transition: background 0.15s; border-bottom: 1px solid rgba(255,255,255,0.05); }
  .search-result-item:last-child { border-bottom: none; }
  .search-result-item:hover { background: rgba(255,255,255,0.06); }
  .sr-name   { font-size: 0.88rem; font-weight: 500; color: rgba(255,255,255,0.85); }
  .sr-region { font-size: 0.74rem; font-weight: 300; color: rgba(255,255,255,0.35); }

  /* ── Hero ── */
  .hero { display: flex; align-items: center; justify-content: space-between; margin-bottom: 1.5rem; gap: 1rem; animation: fadeUp 0.55s 0.05s ease both; }
  .hero-icon { flex-shrink: 0; filter: drop-shadow(0 12px 32px rgba(0,0,0,0.15)); }
  .anim-float { animation: floatIcon 5s ease-in-out infinite; }
  @keyframes floatIcon { 0%,100%{transform:translateY(0) rotate(-1deg)} 50%{transform:translateY(-10px) rotate(1deg)} }
  .hero-info { text-align: right; }
  .hero-temp  { display: flex; align-items: flex-start; justify-content: flex-end; line-height: 1; gap: 2px; }
  .temp-num   { font-size: 6.5rem; font-weight: 800; color: var(--temp-color); letter-spacing: -0.04em; line-height: 1; }
  .temp-deg   { font-size: 2rem; font-weight: 200; color: var(--text-muted); padding-top: 0.65rem; }
  .hero-cond  { font-size: 1rem; font-weight: 400; color: var(--text-secondary); margin-top: 0.25rem; }
  .feels-like { font-size: 0.78rem; font-weight: 300; color: var(--text-muted); margin-top: 0.2rem; }
  .hero-range { display: flex; justify-content: flex-end; gap: 0.4rem; margin-top: 0.4rem; font-size: 0.8rem; font-weight: 300; }
  .rh  { color: rgba(255,160,60,0.85); }
  .rl  { color: rgba(80,160,255,0.85); }
  .rdiv { color: var(--text-muted); }

  /* ── Pills ── */
  .pills { display: flex; gap: 0.55rem; margin-bottom: 1.25rem; animation: fadeUp 0.55s 0.1s ease both; }
  .pill  { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 0.28rem; padding: 0.8rem 0.4rem; background: var(--pill-bg); border: 1px solid var(--card-border); border-radius: 16px; color: var(--text-muted); transition: opacity 0.2s; }
  .pill:hover { opacity: 0.8; }
  .pill-val  { font-size: 0.92rem; font-weight: 600; color: var(--text-primary); }
  .pill-lbl  { font-size: 0.6rem; font-weight: 300; text-transform: uppercase; letter-spacing: 0.08em; text-align: center; }

  /* ── Cards ── */
  .card { background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 20px; padding: 1.1rem 1.15rem; margin-bottom: 1rem; animation: fadeUp 0.5s 0.12s ease both; }
  .card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.8rem; }
  .card-title  { font-size: 0.67rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.13em; color: var(--text-muted); }
  .spark-range { font-size: 0.78rem; font-weight: 500; color: var(--accent); }
  .spark-labels { display: flex; justify-content: space-between; margin-top: 0.35rem; }
  .spark-labels span { font-size: 0.62rem; font-weight: 300; color: var(--text-muted); }

  /* ── Sun arc ── */
  .sun-pct  { font-size: 0.72rem; font-weight: 300; color: rgba(255,200,80,0.65); }
  .sun-times { display: flex; justify-content: space-between; padding: 0 0.25rem; }
  .sun-time  { display: flex; align-items: center; gap: 0.4rem; }
  .sun-time > div { display: flex; flex-direction: column; }
  .sun-label { font-size: 0.6rem; font-weight: 300; text-transform: uppercase; letter-spacing: 0.08em; color: var(--text-muted); }
  .sun-val   { font-size: 0.85rem; font-weight: 500; color: rgba(255,215,80,0.9); }

  /* ── Block label ── */
  .block-label { font-size: 0.67rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.14em; color: var(--text-muted); margin-bottom: 0.8rem; padding-left: 2px; }

  /* ── Forecast ── */
  .forecast-row { display: grid; grid-template-columns: 1fr 1fr; gap: 0.65rem; margin-bottom: 1.75rem; }
  .fc-card  { background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 20px; padding: 1.1rem 0.95rem; display: flex; flex-direction: column; gap: 0.18rem; animation: fadeUp 0.5s ease both; transition: opacity 0.2s, transform 0.2s; }
  .fc-card:hover { opacity: 0.85; transform: translateY(-2px); }
  .fc-day   { font-size: 0.9rem; font-weight: 600; color: var(--text-primary); }
  .fc-dt    { font-size: 0.7rem; font-weight: 300; color: var(--text-muted); }
  .fc-icon  { margin: 0.55rem 0 0.2rem; filter: drop-shadow(0 4px 10px rgba(0,0,0,0.15)); }
  .fc-label { font-size: 0.74rem; font-weight: 300; color: var(--text-secondary); }
  .fc-uv    { font-size: 0.68rem; font-weight: 400; margin-top: 0.1rem; }
  .fc-temps { display: flex; align-items: center; gap: 0.4rem; margin-top: 0.55rem; }
  .fc-hi    { font-size: 0.82rem; font-weight: 600; color: rgba(255,160,60,0.9); }
  .fc-lo    { font-size: 0.82rem; font-weight: 300; color: rgba(80,160,255,0.8); }
  .fc-bar-track { flex: 1; height: 3px; border-radius: 2px; background: rgba(128,128,128,0.15); overflow: hidden; }
  .fc-bar-fill  { height: 100%; border-radius: 2px; background: linear-gradient(90deg, #4fc3f7, #ffb74d); transition: width 0.9s ease; }

  /* ── History ── */
  .hist-list { display: flex; flex-direction: column; gap: 0.38rem; }
  .hist-row  { display: grid; grid-template-columns: 56px 28px 1fr auto; align-items: center; gap: 0.7rem; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 14px; padding: 0.7rem 0.95rem; animation: fadeUp 0.45s ease both; transition: opacity 0.2s; }
  .hist-row:hover { opacity: 0.8; }
  .hist-date { display: flex; flex-direction: column; }
  .hist-wd   { font-size: 0.62rem; font-weight: 400; text-transform: uppercase; letter-spacing: 0.07em; color: var(--text-muted); }
  .hist-md   { font-size: 0.8rem; font-weight: 500; color: var(--text-secondary); }
  .hist-ico  { filter: drop-shadow(0 2px 5px rgba(0,0,0,0.15)); }
  .hist-cond { font-size: 0.76rem; font-weight: 300; color: var(--text-muted); }
  .hist-bar-group { display: flex; align-items: center; gap: 0.38rem; }
  .hist-lo-val { font-size: 0.7rem; font-weight: 400; color: rgba(80,160,255,0.7); width: 22px; text-align: right; }
  .hist-hi-val { font-size: 0.7rem; font-weight: 400; color: rgba(255,160,60,0.7); width: 22px; }
  .hist-track { width: 50px; height: 3px; border-radius: 2px; background: rgba(128,128,128,0.15); position: relative; overflow: hidden; }
  .hist-fill  { position: absolute; top: 0; height: 100%; background: linear-gradient(90deg, #4fc3f7, #ffb74d); border-radius: 2px; transition: all 0.9s ease; }

  /* ── Credit ── */
  .credit { text-align: center; margin-top: 2rem; font-size: 0.64rem; font-weight: 300; color: var(--text-muted); letter-spacing: 0.06em; opacity: 0.6; }

  @keyframes fadeUp { from{opacity:0;transform:translateY(14px)} to{opacity:1;transform:translateY(0)} }
</style>