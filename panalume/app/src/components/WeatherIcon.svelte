<script>
  // Full-colour weather icons, drawn here (no outside files): Home Assistant's
  // weather conditions, with a night variant for the sun / moon.
  let { c = 'cloudy', night = false, size = '1.6em' } = $props();
  const uid = 'wi' + Math.random().toString(36).slice(2, 8);
  const cond = $derived(String(c || 'cloudy'));
  const isNight = $derived(night || cond === 'clear-night');
  const dark = $derived(['pouring', 'lightning', 'lightning-rainy', 'hail'].includes(cond));
  const CLOUD = 'M20 52C11.7 52 6 46.8 6 40c0-6 4.5-10.4 10.3-11.4C18.3 20.9 25.3 16 33 16c8.5 0 15.300 5.700 16.600 13.400C56 29.900 60 34.400 60 40.600 60 47 55 52 48.500 52Z';
  const rain = $derived(cond === 'rainy' || cond === 'pouring' || cond === 'lightning-rainy' || cond === 'snowy-rainy');
  const drops = $derived(cond === 'pouring' ? [[20, 55, 17, 63], [29, 55, 26, 63], [38, 55, 35, 63], [47, 55, 44, 63]] : [[24, 55, 21, 62], [34, 55, 31, 62], [44, 55, 41, 62]]);
  const solo = $derived(['sunny', 'clear-night'].includes(cond));
  const withSky = $derived(cond === 'partlycloudy');
</script>

<svg viewBox="0 0 64 64" width={size} height={size} aria-hidden="true" style="flex:none;overflow:visible">
  <defs>
    <linearGradient id="{uid}s" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="#ffe680" /><stop offset="1" stop-color="#ffb21e" /></linearGradient>
    <linearGradient id="{uid}m" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#fff3bd" /><stop offset="1" stop-color="#f1bd3f" /></linearGradient>
    <linearGradient id="{uid}c" gradientUnits="userSpaceOnUse" x1="0" y1="16" x2="0" y2="52"><stop offset="0" stop-color="#ffffff" /><stop offset="1" stop-color="#cfd9ea" /></linearGradient>
    <linearGradient id="{uid}d" gradientUnits="userSpaceOnUse" x1="0" y1="16" x2="0" y2="52"><stop offset="0" stop-color="#b3bfd2" /><stop offset="1" stop-color="#7d8aa3" /></linearGradient>
    <linearGradient id="{uid}b" gradientUnits="userSpaceOnUse" x1="0" y1="16" x2="0" y2="52"><stop offset="0" stop-color="#c9d3e3" /><stop offset="1" stop-color="#97a5bd" /></linearGradient>
  </defs>

  {#if solo}
    {#if isNight}
      <path d="M40 8A24 24 0 1 0 56 44A19 19 0 0 1 40 8Z" fill="url(#{uid}m)" />
      <circle cx="49" cy="17" r="2" fill="#fff3bd" /><circle cx="55" cy="27" r="1.3" fill="#fff3bd" />
    {:else}
      <g stroke="#ffc233" stroke-width="3.400" stroke-linecap="round">
        {#each [0, 45, 90, 135, 180, 225, 270, 315] as r}<line x1="32" y1="6" x2="32" y2="11" transform="rotate({r} 32 32)" />{/each}
      </g>
      <circle cx="32" cy="32" r="14" fill="url(#{uid}s)" />
    {/if}
  {:else if withSky}
    {#if isNight}
      <path d="M28 4A15 15 0 1 0 38 30A12 12 0 0 1 28 4Z" fill="url(#{uid}m)" transform="translate(-2 2)" />
    {:else}
      <g transform="translate(-4 -3)">
        <g stroke="#ffc233" stroke-width="3" stroke-linecap="round">
          {#each [0, 45, 90, 135, 180, 225, 270, 315] as r}<line x1="30" y1="9" x2="30" y2="13" transform="rotate({r} 30 30)" />{/each}
        </g>
        <circle cx="30" cy="30" r="11" fill="url(#{uid}s)" />
      </g>
    {/if}
    <path d={CLOUD} fill="url(#{uid}c)" transform="translate(4 4)" />
  {:else if cond === 'windy' || cond === 'windy-variant'}
    {#if cond === 'windy-variant'}<path d={CLOUD} fill="url(#{uid}c)" transform="translate(-2 -10) scale(.9)" />{/if}
    <g fill="none" stroke="#9fb9dd" stroke-width="3.600" stroke-linecap="round">
      <path d="M8 30H38a7 7 0 1 0-7-7" /><path d="M8 40H46a7 7 0 1 1-7 7" /><path d="M8 50H26" />
    </g>
  {:else if cond === 'exceptional'}
    <path d="M32 7L60 55H4Z" fill="#ffb02e" stroke="#ffb02e" stroke-width="4" stroke-linejoin="round" />
    <rect x="30" y="23" width="4" height="17" rx="2" fill="#fff" /><circle cx="32" cy="46" r="2.400" fill="#fff" />
  {:else}
    <!-- a cloud, with weather under it -->
    {#if cond === 'cloudy'}<path d={CLOUD} fill="url(#{uid}b)" transform="translate(-4 -9) scale(.82)" />{/if}
    <path d={CLOUD} fill="url(#{uid}{dark ? 'd' : 'c'})" transform={cond === 'cloudy' ? 'translate(5 3)' : rain || cond === 'snowy' || cond === 'hail' || cond === 'fog' || cond.startsWith('lightning') ? 'translate(0 -6)' : ''} />
    {#if cond === 'fog'}
      <g stroke="#a9b6c9" stroke-width="3.400" stroke-linecap="round"><line x1="12" y1="52" x2="50" y2="52" /><line x1="18" y1="59" x2="56" y2="59" /></g>
    {/if}
    {#if rain}
      <g stroke="#4da6ff" stroke-width="3.200" stroke-linecap="round" transform="translate(0 -6)">
        {#each cond === 'snowy-rainy' ? drops.slice(0, 2) : drops as d}<line x1={d[0]} y1={d[1]} x2={d[2]} y2={d[3]} />{/each}
      </g>
    {/if}
    {#if cond === 'snowy' || cond === 'snowy-rainy'}
      <g fill="#f2f9ff" stroke="#9ccaf5" stroke-width="1.200">
        {#each (cond === 'snowy' ? [[22, 58], [33, 56], [44, 58], [27, 63]] : [[44, 58], [36, 63]]) as p}<circle cx={p[0]} cy={p[1] - 6} r="2.800" />{/each}
      </g>
    {/if}
    {#if cond === 'hail'}
      <g fill="#dcefff" stroke="#8fc3f0" stroke-width="1.200">{#each [[22, 56], [33, 59], [44, 56], [28, 64]] as p}<circle cx={p[0]} cy={p[1] - 6} r="3" />{/each}</g>
    {/if}
    {#if cond === 'lightning' || cond === 'lightning-rainy'}
      <path d="M34 36L26 50h7l-3 11 11-15h-7l4-10Z" fill="#ffd23f" stroke="#ffb300" stroke-width="1.400" stroke-linejoin="round" transform="translate(0 -8)" />
    {/if}
  {/if}
</svg>
