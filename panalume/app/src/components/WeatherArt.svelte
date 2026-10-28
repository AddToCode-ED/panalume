<script>
  // A minimalist, flat "weather poster" for a card background: the sky for the
  // time of day, the sun or moon where it is, clouds / rain / snow / fog for the
  // condition, and two quiet hills. Drawn here, no pictures to download.
  let { cond = 'sunny', phase = 'day', az = NaN, el = NaN, hour = 12, animate = true } = $props();
  const uid = 'wa' + Math.random().toString(36).slice(2, 8);

  const hex = (h) => { const n = parseInt(h.slice(1), 16); return [(n >> 16) & 255, (n >> 8) & 255, n & 255]; };
  const mix = (a, b, k) => { const x = hex(a), y = hex(b); return '#' + x.map((v, i) => Math.round(v + (y[i] - v) * k).toString(16).padStart(2, '0')).join(''); };

  const SKY = { day: ['#2c7fd6', '#7fc0f2'], dawn: ['#3a3c80', '#f4a97f'], dusk: ['#342a6c', '#f0805f'], night: ['#060b22', '#1a2b5a'] };
  const GREY = { day: '#8693a6', dawn: '#7c7287', dusk: '#7a6a82', night: '#262c42' };

  const c = $derived(cond === 'sunny' && phase === 'night' ? 'clear-night' : cond);
  const stormy = $derived(['lightning', 'lightning-rainy', 'hail'].includes(c));
  const wet = $derived(['rainy', 'pouring', 'lightning-rainy', 'snowy-rainy'].includes(c));
  const snow = $derived(c === 'snowy' || c === 'snowy-rainy');
  const fog = $derived(c === 'fog');
  // How much of the sky is covered (0 clear … 1 overcast).
  const cover = $derived({ sunny: 0, 'clear-night': 0, partlycloudy: 0.35, windy: 0.1, 'windy-variant': 0.4, cloudy: 0.8, fog: 0.8, rainy: 0.9, pouring: 1, lightning: 1, 'lightning-rainy': 1, hail: 1, snowy: 0.9, 'snowy-rainy': 0.9, exceptional: 0.5 }[c] ?? 0.5);

  const sky = $derived.by(() => {
    let [t, b] = SKY[phase] || SKY.day;
    const g = GREY[phase] || GREY.day;
    const amt = { cloudy: 0.45, partlycloudy: 0.12, rainy: 0.6, pouring: 0.72, lightning: 0.75, 'lightning-rainy': 0.78, hail: 0.65, snowy: 0.4, 'snowy-rainy': 0.55, fog: 0.5, 'windy-variant': 0.2, windy: 0.05, exceptional: 0.3 }[c] || 0;
    t = mix(t, g, amt); b = mix(b, g, amt * 0.85);
    if (stormy || c === 'pouring') { t = mix(t, '#000000', 0.2); b = mix(b, '#000000', 0.12); }
    if (fog || snow) { b = mix(b, phase === 'night' ? '#6f7da0' : '#e4ebf3', 0.35); }
    return [t, b];
  });
  const hillBack = $derived(snow ? mix(sky[1], '#ffffff', 0.55) : mix(sky[1], '#000000', 0.22));
  const hillFront = $derived(snow ? mix(sky[1], '#ffffff', 0.78) : mix(sky[1], '#000000', 0.42));
  const cloudCol = $derived(stormy || c === 'pouring' || c === 'rainy' ? (phase === 'night' ? '#4a5575' : '#66728a') : phase === 'night' ? '#8a9bcc' : phase === 'day' ? '#ffffff' : '#ffe6da');

  // The sun follows the real sun; the moon crosses the sky through the night.
  const clamp = (v, a, b) => Math.min(b, Math.max(a, v));
  const sun = $derived.by(() => {
    if (Number.isFinite(az) && Number.isFinite(el)) return { x: clamp(0.18 + ((az - 80) / 200) * 0.64, 0.14, 0.86), y: clamp(0.62 - Math.max(0, el) / 70 * 0.4, 0.2, 0.6) };
    const p = clamp((hour - 6) / 13, 0, 1);
    return { x: 0.15 + p * 0.7, y: 0.6 - Math.sin(p * Math.PI) * 0.4 };
  });
  const moon = $derived.by(() => { const h = hour >= 12 ? hour - 18 : hour + 6; const p = clamp(h / 12, 0, 1); return { x: 0.2 + p * 0.62, y: 0.5 - Math.sin(p * Math.PI) * 0.28 }; });
  const body = $derived(phase === 'night' ? moon : sun);
  const bodyX = $derived(body.x * 400), bodyY = $derived(body.y * 240);
  const bodyAlpha = $derived(clamp(1 - cover * 0.95, 0.12, 1));
  const celest = $derived(phase === 'night' ? '#f6efc8' : phase === 'day' ? '#fff3b0' : '#ffd9a0');

  const STARS = [[30, 24, 1.4], [78, 60, 1], [120, 18, 1.2], [165, 48, 1], [210, 26, 1.5], [262, 14, 1], [300, 52, 1.2], [345, 28, 1.4], [372, 70, 1], [52, 96, 1], [188, 84, 1], [330, 100, 1]];
  const starOp = $derived(phase === 'night' ? clamp(1 - cover * 1.1, 0, 0.9) : 0);

  // Flat cloud: a pill with two round bumps. [x, y, scale, opacity, drift class]
  const CLOUDS = $derived.by(() => {
    if (cover <= 0.05) return [];
    if (cover < 0.5) return [[70, 70, 1, 0.55, 'a'], [270, 52, 0.8, 0.4, 'b']];
    const o = clamp(cover, 0.5, 1);
    return [[40, 52, 1.5, 0.45 * o, 'a'], [250, 38, 1.7, 0.5 * o, 'b'], [140, 92, 1.8, 0.55 * o, 'b'], [330, 100, 1.4, 0.5 * o, 'a'], [-10, 112, 1.6, 0.5 * o, 'a']];
  });
  const RAIN = Array.from({ length: 26 }, (_, i) => [(i * 37 + 11) % 410, (i * 53 + 7) % 200]);
  const FLAKES = Array.from({ length: 28 }, (_, i) => [(i * 41 + 9) % 400, (i * 59 + 13) % 205, 1.6 + (i % 3) * 0.7]);
</script>

<svg class="art" class:still={!animate} viewBox="0 0 400 240" preserveAspectRatio="xMidYMid slice" aria-hidden="true">
  <defs>
    <linearGradient id="{uid}k" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color={sky[0]} /><stop offset="1" stop-color={sky[1]} /></linearGradient>
    <radialGradient id="{uid}g"><stop offset="0" stop-color={celest} stop-opacity=".55" /><stop offset="1" stop-color={celest} stop-opacity="0" /></radialGradient>
  </defs>
  <rect width="400" height="240" fill="url(#{uid}k)" />

  {#each STARS as s}<circle cx={s[0]} cy={s[1]} r={s[2]} fill="#fff" opacity={starOp * 0.8} />{/each}

  <g opacity={bodyAlpha}>
    <circle cx={bodyX} cy={bodyY} r="70" fill="url(#{uid}g)" />
    {#if phase === 'night'}
      <circle cx={bodyX} cy={bodyY} r="20" fill={celest} />
      <circle cx={bodyX + 9} cy={bodyY - 6} r="17" fill={sky[0]} opacity=".92" />
    {:else}
      <circle cx={bodyX} cy={bodyY} r="22" fill={celest} />
    {/if}
  </g>

  {#each CLOUDS as k}
    <g class="drift {k[4]}"><g transform="translate({k[0]} {k[1]}) scale({k[2]})" fill={cloudCol} opacity={k[3]}>
      <rect x="0" y="22" width="86" height="22" rx="11" /><circle cx="26" cy="22" r="17" /><circle cx="50" cy="16" r="21" />
    </g></g>
  {/each}

  {#if fog}
    {#each [96, 120, 144, 168] as y, i}<rect class="drift {i % 2 ? 'a' : 'b'}" x={-40 + i * 30} y={y} width="300" height="9" rx="4.500" fill="#fff" opacity={0.2 + i * 0.04} />{/each}
  {/if}
  {#if wet}
    <g stroke="#cfe6ff" stroke-width="1.400" stroke-linecap="round" opacity={c === 'pouring' || stormy ? 0.5 : 0.36}>
      {#each RAIN.slice(0, c === 'pouring' || c === 'lightning-rainy' ? 26 : 16) as p}<line class="rain" x1={p[0]} y1={p[1]} x2={p[0] - 7} y2={p[1] + 20} />{/each}
    </g>
  {/if}
  {#if snow}
    <g fill="#fff" opacity=".8">{#each FLAKES as p}<circle class="flake" cx={p[0]} cy={p[1]} r={p[2]} />{/each}</g>
  {/if}
  {#if c === 'hail'}
    <g fill="#e6f3ff" opacity=".75">{#each FLAKES as p}<circle cx={p[0]} cy={p[1]} r={p[2] + 0.6} />{/each}</g>
  {/if}
  {#if c === 'lightning' || c === 'lightning-rainy'}
    <path class="bolt" d="M236 40L214 98h19l-9 50 38-70h-22l17-38Z" fill="#ffe27a" opacity=".92" />
  {/if}
  {#if c === 'windy' || c === 'windy-variant'}
    <g fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" opacity=".4">
      <path class="drift a" d="M20 100H150a14 14 0 1 0-14-14" /><path class="drift b" d="M120 140H300a16 16 0 1 1-16 16" /><path class="drift a" d="M240 80H350" />
    </g>
  {/if}

  <path d="M0 176Q95 140 200 168T400 152V240H0Z" fill={hillBack} />
  <path d="M0 208Q120 182 240 202T400 196V240H0Z" fill={hillFront} />
  <!-- keeps the text on top readable -->
  <rect width="400" height="240" fill="#000" opacity=".16" />
</svg>

<style>
  .art { position: absolute; inset: 0; width: 100%; height: 100%; display: block; }
  .drift { animation: drift 60s ease-in-out infinite alternate; }
  .drift.b { animation-duration: 85s; animation-direction: alternate-reverse; }
  .rain { animation: fall 1.1s linear infinite; }
  .flake { animation: snowfall 6s ease-in-out infinite alternate; }
  .bolt { animation: flash 6s ease-in-out infinite; }
  @keyframes drift { from { transform: translateX(-14px); } to { transform: translateX(16px); } }
  @keyframes fall { from { transform: translate(0, -6px); } to { transform: translate(-3px, 8px); } }
  @keyframes snowfall { from { transform: translate(-5px, -3px); } to { transform: translate(6px, 7px); } }
  @keyframes flash { 0%, 86%, 92%, 100% { opacity: .2; } 88%, 90% { opacity: 1; } }
  .still *, .still { animation: none !important; }
  @media (prefers-reduced-motion: reduce) { .art * { animation: none !important; } }
</style>
