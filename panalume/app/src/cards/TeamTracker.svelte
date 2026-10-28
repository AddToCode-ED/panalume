<script module>
  import { states } from '../lib/ha.svelte.js';
  // Finds a Team Tracker sensor (the integration by vasqued2): a sensor whose
  // attributes carry the team and sport. Prefers one with a game on right now.
  function autofill() {
    const ids = [...states.keys()].filter((id) => id.startsWith('sensor.') && states.get(id)?.attributes?.team_abbr !== undefined && states.get(id)?.attributes?.sport !== undefined).sort();
    return { entity: ids.find((id) => states.get(id)?.state === 'IN') || ids[0] || '' };
  }
  export const meta = {
    type: 'team-tracker', name: 'Team tracker', icon: 'mdi:scoreboard', category: 'Info',
    size: { w: 380, h: 230 }, tap: 'more-info', hold: 'more-info',
    defaults: { entity: '', logos: true, bglogos: true },
    autofill,
    sections: [
      { key: 'league', label: 'League' }, { key: 'records', label: 'Records and rankings' },
      { key: 'live', label: 'Live play details' }, { key: 'prob', label: 'Win probability' },
      { key: 'info', label: 'Venue, TV and odds' },
    ],
    fields: [
      { key: 'entity', label: 'Team Tracker sensor', type: 'entity', domain: 'sensor' },
      { key: 'name', label: 'Title (blank: the league)', type: 'text' },
      { key: 'logos', label: 'Team logos (loaded from the sports service’s website, e.g. ESPN)', type: 'bool' },
      { key: 'bglogos', label: 'Large faded logos and team colours in the background', type: 'bool' },
    ],
  };
</script>

<script>
  // A scoreboard for a sports team, from the Team Tracker integration
  // (https://github.com/vasqued2/ha-teamtracker). One sensor per team: before the
  // game it shows when and where; during it the score, clock and, per sport,
  // possession, down and distance, the runners on base or the last play, and the
  // win probability; afterwards the final score. It also copes with bye weeks
  // and with no game being found. Away team on the left, home team on the right.
  // Short cards become a single compact row.
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { clock } from '../lib/clock.svelte.js';
  let { props, w = 380, h = 230 } = $props();
  const show = (k) => !props.hide?.[k];

  const e = $derived(ent(props.entity));
  const A = $derived(e?.attributes || {});
  const STATES = ['PRE', 'IN', 'POST', 'BYE', 'NOT_FOUND'];
  const st = $derived(String(e?.state || '').toUpperCase());
  const dead = $derived(!e || !STATES.includes(st));

  const SPORT_ICON = {
    'australian-football': 'mdi:football-australian', baseball: 'mdi:baseball', basketball: 'mdi:basketball', cricket: 'mdi:cricket', football: 'mdi:football',
    golf: 'mdi:golf-tee', hockey: 'mdi:hockey-puck', mma: 'mdi:karate', racing: 'mdi:flag-checkered', rugby: 'mdi:rugby', soccer: 'mdi:soccer',
    tennis: 'mdi:tennis', volleyball: 'mdi:volleyball',
  };
  const sport = $derived(String(A.sport || '').toLowerCase());
  const sportIcon = $derived(SPORT_ICON[sport] || 'mdi:scoreboard');

  // Win probability comes as 0-1 (or a percentage); 0 / missing means "not known".
  const pct = (v) => { const n = Number(v); return Number.isFinite(n) && n > 0 ? (n > 1 ? n / 100 : n) : 0; };
  const rankOf = (v) => { const n = Number(v); return Number.isFinite(n) && n > 0 && n < 99 ? `#${n}` : ''; };
  const side = (p) => ({
    p, id: A[p + '_id'], abbr: A[p + '_abbr'] || '', name: A[p + '_name'] || A[p + '_abbr'] || '', logo: A[p + '_logo'], record: A[p + '_record'],
    rank: rankOf(A[p + '_rank']), score: A[p + '_score'], win: A[p + '_winner'] === true, colors: A[p + '_colors'], ha: A[p + '_homeaway'], prob: pct(A[p + '_win_probability']),
  });
  const me = $derived(side('team'));
  const op = $derived(side('opponent'));
  const hasOpp = $derived(!!(op.name || op.abbr));
  // Away left, home right (without a home / away, your team first).
  const pair = $derived(!hasOpp ? [me] : me.ha === 'home' ? [op, me] : [me, op]);
  const hasWinner = $derived(me.win || op.win);

  // Team colours are often very dark; lighten those so they show on the glass.
  function fix(c, fallback) {
    const m = /^#?([0-9a-f]{6})$/i.exec(String(c || '').trim());
    if (!m) return fallback;
    const n = parseInt(m[1], 16);
    const lum = (0.2126 * ((n >> 16) & 255) + 0.7152 * ((n >> 8) & 255) + 0.0722 * (n & 255)) / 255;
    return lum < 0.28 ? `color-mix(in srgb, #${m[1]} 45%, #fff)` : `#${m[1]}`;
  }
  const colors = $derived(pair.map((s, i) => fix(s.colors?.[0], i === 0 ? '#7aa2ff' : '#ffb547')));
  const accent = $derived(colors[pair.indexOf(me)] || '#7aa2ff');

  // Win probability of each side (if only one is known the other is the rest).
  const probs = $derived.by(() => {
    if (st !== 'IN' || !show('prob') || !hasOpp) return null;
    let a = me.prob, b = op.prob;
    if (!a && !b) return null;
    if (!b) b = 1 - a; else if (!a) a = 1 - b;
    const m = { [me.p]: a, [op.p]: b };
    const l = m[pair[0].p], r = m[pair[1].p];
    return { l, r, share: l / (l + r || 1) };
  });

  const when = $derived.by(() => {
    const d = new Date(A.date);
    if (!A.date || isNaN(d)) return '';
    const day0 = (x) => { const y = new Date(x); y.setHours(0, 0, 0, 0); return y; };
    const diff = Math.round((day0(d) - day0(clock.now)) / 864e5);
    const hm = d.toLocaleTimeString(undefined, { hour: 'numeric', minute: '2-digit' });
    const day = diff === 0 ? 'Today' : diff === 1 ? 'Tomorrow' : diff > 1 && diff < 7 ? d.toLocaleDateString(undefined, { weekday: 'short' }) : d.toLocaleDateString(undefined, { weekday: 'short', day: 'numeric', month: 'short' });
    return { day, hm };
  });

  const period = $derived.by(() => {
    const q = A.quarter;
    if (q == null || q === '') return '';
    const s = String(q);
    if (!/^\d+$/.test(s)) return s;
    return sport === 'hockey' ? `P${s}` : sport === 'volleyball' ? `Set ${s}` : ['football', 'basketball', 'australian-football'].includes(sport) ? `Q${s}` : s;
  });
  const clockText = $derived(A.clock ? String(A.clock) : '');
  const live = $derived(st === 'IN' && show('live'));
  const hasBall = (s) => live && A.possession != null && A.possession !== '' && String(A.possession) === String(s.id);
  const venue = $derived([A.venue, A.location].filter(Boolean).join(', '));
  const info = $derived(show('info') ? [
    A.series_summary,
    st === 'PRE' ? venue : '',
    st !== 'POST' ? A.tv_network : '',
    st === 'PRE' && A.odds ? `${A.odds}${A.overunder ? ` · O/U ${A.overunder}` : ''}` : '',
  ].filter(Boolean) : []);
  const baseball = $derived(sport === 'baseball' && live);
  const num = (v) => (v == null || v === '' || !Number.isFinite(Number(v)) ? null : Number(v));
  const bases = $derived([!!A.on_second, !!A.on_third, !!A.on_first]);

  const compact = $derived(h < 150);
  const wide = $derived(w >= 330);
  const tall = $derived(h >= 215);
  const logo = $derived(compact ? Math.max(28, Math.min(44, h - 36)) : Math.max(36, Math.min(76, h * 0.27, w * 0.17)));
  const logos = $derived(props.logos !== false);
  const bglogos = $derived(logos && props.bglogos !== false);
  let bad = $state({});
  // Each team's logo, huge and faded, behind its half of the card.
  const big = $derived(Math.round(compact ? Math.max(90, Math.min(h * 1.5, w * 0.5)) : Math.max(120, Math.min(h * 1.2, w * 0.62))));

  const statusText = $derived(st === 'IN' ? 'LIVE' : st === 'POST' ? 'Final' : st === 'PRE' ? (A.kickoff_in ? String(A.kickoff_in) : 'Upcoming') : st === 'BYE' ? 'Bye week' : 'No game');
  const title = $derived(t(props.name) || A.league_name || A.league || '');

  // Bye week / no game: the sensor has no team logo then, only the league's.
  const idleLogo = $derived([me.logo, A.league_logo].find((u) => u && !bad[u]) || '');
  const idleSize = $derived(Math.max(36, Math.min(84, h - 90)));
  const idleText = $derived.by(() => {
    if (st === 'BYE') return 'Bye week';
    // "No competition scheduled for 'LAD' between <start> and <end>": say how far it looked.
    const end = /between\s+\S+\s+and\s+(\S+)/i.exec(String(A.api_message || ''))?.[1];
    const d = end ? new Date(end) : null;
    return d && !isNaN(d) ? `No upcoming games through ${d.toLocaleDateString()}` : 'No game found';
  });
</script>

<div class="tt" class:compact style="--c:{accent}">
  <div class="glow"></div>
  {#if !dead && bglogos}
    <div class="bg" aria-hidden="true">
      {#if st === 'BYE' || st === 'NOT_FOUND'}
        {#if idleLogo}<div class="half l" style="--tc:{accent}"><img src={idleLogo} alt="" style="width:{big}px;height:{big}px;margin-left:-{Math.round(big * 0.14)}px" onerror={() => (bad[idleLogo] = true)} /></div>{/if}
      {:else}
        {#each pair as s, i (s.p)}
          <div class="half {i ? 'r' : 'l'}" style="--tc:{colors[i]}">
            {#if s.logo && !bad[s.logo]}<img src={s.logo} alt="" style="width:{big}px;height:{big}px;{i ? 'margin-right' : 'margin-left'}:-{Math.round(big * 0.14)}px" onerror={() => (bad[s.logo] = true)} />{/if}
          </div>
        {/each}
      {/if}
    </div>
  {/if}
  {#if dead}
    <div class="empty"><Icon icon="mdi:scoreboard-outline" size="1.8em" /><span>{e ? 'Team Tracker sensor unavailable' : 'Pick a Team Tracker sensor'}</span></div>
  {:else if st === 'BYE' || st === 'NOT_FOUND'}
    <div class="empty">
      {#if logos && idleLogo}<img class="solo" style="width:{idleSize}px;height:{idleSize}px" src={idleLogo} alt="" onerror={() => (bad[idleLogo] = true)} />{:else}<Icon icon={sportIcon} size="{idleSize * 0.7}px" />{/if}
      <b>{me.name || me.abbr}</b>
      {#if !compact && show('league') && (A.league || A.league_name)}<small>{A.league || A.league_name}</small>{/if}
      <span>{idleText}</span>
    </div>
  {:else}
    {#if !compact}
      <div class="head">
        {#if show('league')}
          {#if logos && A.league_logo && !bad[A.league_logo]}<img class="lgl" src={A.league_logo} alt="" onerror={() => (bad[A.league_logo] = true)} />{/if}
          <span class="league">{title}</span>
        {:else}<span class="league"></span>{/if}
        <span class="chip {st}">{#if st === 'IN'}<i class="dot"></i>{/if}{statusText}</span>
      </div>
    {/if}
    <div class="game" style="grid-template-columns:{pair.length === 2 ? '1fr auto 1fr' : '1fr auto'}">
      {#each pair as s, i (s.p)}
        {#if i === 1}{@render mid()}{/if}
        <div class="team" class:win={st === 'POST' && s.win} class:lose={st === 'POST' && hasWinner && !s.win} style="--tc:{colors[i]}">
          <div class="logo" style="width:{logo}px;height:{logo}px">
            {#if logos && s.logo && !bad[s.logo]}<img src={s.logo} alt="" onerror={() => (bad[s.logo] = true)} />{:else}<Icon icon={sportIcon} size="{logo * 0.6}px" />{/if}
            {#if hasBall(s)}<span class="ball" title="Has the ball"><Icon icon={sportIcon} size="1em" /></span>{/if}
          </div>
          <div class="tn">{#if show('records') && s.rank}<span class="rk">{s.rank}</span> {/if}{wide ? s.name : s.abbr || s.name}</div>
          {#if show('records') && s.record && !compact}<div class="rec">{s.record}</div>{/if}
        </div>
      {/each}
      {#if pair.length === 1}{@render mid()}{/if}
    </div>

    {#if !compact}
      {#if baseball}
        <div class="diamond-row">
          <svg viewBox="0 0 40 34" width="38" height="32" aria-hidden="true">
            {#each [[20, 7], [8, 19], [32, 19]] as [x, y], k}
              <rect x={x - 5} y={y - 5} width="10" height="10" transform="rotate(45 {x} {y})" class="base" class:on={bases[k]} />
            {/each}
          </svg>
          <span>{[num(A.balls) != null && num(A.strikes) != null ? `${A.balls}-${A.strikes}` : '', num(A.outs) != null ? `${A.outs} out` : ''].filter(Boolean).join(' · ')}</span>
        </div>
      {/if}
      {#if live && A.down_distance_text}<div class="dd">{A.down_distance_text}</div>{/if}
      {#if probs}
        <div class="prob" title="Win probability">
          <span>{Math.round(probs.l * 100)}%</span>
          <div class="bar"><i style="width:{probs.share * 100}%;background:{colors[0]}"></i><i style="width:{(1 - probs.share) * 100}%;background:{colors[1]}"></i></div>
          <span>{Math.round(probs.r * 100)}%</span>
        </div>
      {/if}
      {#if live && A.last_play && tall}<div class="play">{A.last_play}</div>{/if}
      {#if info.length && (tall || st !== 'IN')}<div class="info">{info.join(' · ')}</div>{/if}
    {/if}
  {/if}
</div>

{#snippet mid()}
  <div class="mid">
    {#if st === 'PRE'}
      {#if when}<div class="day">{when.day}</div><div class="time">{when.hm}</div>{:else}<div class="time">vs</div>{/if}
    {:else}
      <div class="score"><span class:dim={st === 'POST' && hasWinner && !pair[0].win}>{pair[0].score ?? '–'}</span>{#if pair[1]}<em>–</em><span class:dim={st === 'POST' && hasWinner && !pair[1].win}>{pair[1].score ?? '–'}</span>{/if}</div>
      {#if st === 'IN'}<div class="clk">{[period, clockText].filter(Boolean).join(' · ')}</div>{/if}
    {/if}
    {#if compact}<div class="cs {st}">{#if st === 'IN'}<i class="dot"></i>{/if}{statusText}</div>{/if}
  </div>
{/snippet}

<style>
  .tt { position: relative; height: 100%; display: flex; flex-direction: column; gap: 6px; isolation: isolate; min-width: 0; }
  .glow { position: absolute; inset: calc(var(--pad) * -1); z-index: -1; background: radial-gradient(90% 70% at 50% 35%, color-mix(in srgb, var(--c) 14%, transparent), transparent 70%); pointer-events: none; }
  .bg { position: absolute; inset: calc(var(--pad) * -1); z-index: -1; pointer-events: none; overflow: hidden; border-radius: inherit; }
  .half { position: absolute; top: 0; bottom: 0; width: 62%; display: flex; align-items: center; background: radial-gradient(85% 120% at var(--x) 50%, color-mix(in srgb, var(--tc) 24%, transparent), transparent 72%); }
  .half.l { left: 0; --x: 0%; justify-content: flex-start; }
  .half.r { right: 0; --x: 100%; justify-content: flex-end; }
  .half img { flex: none; object-fit: contain; opacity: .16; filter: saturate(1.15); }
  .half.l img { -webkit-mask-image: linear-gradient(to right, #000 30%, transparent 92%); mask-image: linear-gradient(to right, #000 30%, transparent 92%); }
  .half.r img { -webkit-mask-image: linear-gradient(to left, #000 30%, transparent 92%); mask-image: linear-gradient(to left, #000 30%, transparent 92%); }
  .head { display: flex; align-items: center; gap: 7px; min-width: 0; font-size: .8em; }
  .league { flex: 1; min-width: 0; font-weight: 600; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .lgl { width: 1.5em; height: 1.5em; object-fit: contain; }
  .chip { flex: none; padding: 2px 9px; border-radius: 99px; background: rgba(255,255,255,.07); color: var(--muted); font-weight: 600; display: inline-flex; align-items: center; gap: 5px; }
  .chip.IN, .cs.IN { color: #ff7a7a; background: rgba(255,90,90,.14); }
  .chip.POST { color: inherit; }
  .dot { width: .55em; height: .55em; border-radius: 50%; background: #ff5a5a; animation: pulse 1.4s ease-in-out infinite; }
  @keyframes pulse { 50% { opacity: .35; } }
  .game { flex: 1; min-height: 0; display: grid; align-items: center; gap: 6px; }
  .team { display: flex; flex-direction: column; align-items: center; gap: 2px; min-width: 0; text-align: center; transition: opacity .4s; }
  .team.lose { opacity: .55; }
  .logo { position: relative; flex: none; display: grid; place-items: center; color: var(--tc); }
  .logo img, .solo { width: 100%; height: 100%; object-fit: contain; filter: drop-shadow(0 2px 6px rgba(0,0,0,.35)); }
  .solo { object-fit: contain; }
  .empty small { font-size: .75em; opacity: .8; }
  .ball { position: absolute; right: -4px; bottom: -3px; display: grid; place-items: center; width: 1.5em; height: 1.5em; border-radius: 50%; background: rgba(0,0,0,.6); color: #ffb547; }
  .tn { text-shadow: 0 1px 8px rgba(0,0,0,.35); font-weight: 700; max-width: 100%; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .team.win .tn { color: var(--tc); }
  .rk { font-weight: 500; font-size: .8em; color: var(--muted); margin-right: .3em; }
  .rec { font-size: .75em; color: var(--muted); }
  .mid { text-align: center; display: flex; flex-direction: column; align-items: center; gap: 1px; padding: 0 2px; min-width: 3.2em; }
  .score { text-shadow: 0 1px 10px rgba(0,0,0,.4); font-size: 2em; font-weight: 700; font-variant-numeric: tabular-nums; line-height: 1.1; display: flex; gap: .25em; align-items: baseline; white-space: nowrap; }
  .score em { font-style: normal; color: var(--muted); font-weight: 400; }
  .score .dim { color: var(--muted); font-weight: 500; }
  .clk { font-size: .85em; color: var(--c); font-weight: 600; white-space: nowrap; }
  .day { font-size: .8em; color: var(--muted); }
  .time { font-size: 1.3em; font-weight: 700; white-space: nowrap; }
  .cs { font-size: .7em; color: var(--muted); font-weight: 600; padding: 1px 7px; border-radius: 99px; display: inline-flex; align-items: center; gap: 4px; }
  .dd { text-align: center; font-size: .85em; font-weight: 600; color: var(--c); }
  .diamond-row { display: flex; align-items: center; justify-content: center; gap: 8px; font-size: .85em; font-weight: 600; }
  .base { fill: rgba(255,255,255,.1); stroke: rgba(255,255,255,.35); stroke-width: 1; }
  .base.on { fill: #ffb547; stroke: #ffb547; }
  .prob { display: flex; align-items: center; gap: 7px; font-size: .75em; color: var(--muted); font-variant-numeric: tabular-nums; }
  .bar { flex: 1; height: 5px; border-radius: 3px; overflow: hidden; display: flex; background: rgba(255,255,255,.08); }
  .bar i { display: block; height: 100%; transition: width 1s; }
  .play { font-size: .8em; color: var(--muted); text-align: center; line-height: 1.3; overflow: hidden; display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; line-clamp: 2; }
  .info { font-size: .75em; color: var(--muted); text-align: center; line-height: 1.3; overflow: hidden; display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; line-clamp: 2; }
  .empty { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 4px; color: var(--muted); text-align: center; font-size: .9em; }
  .empty b { color: inherit; font-size: 1.15em; }
  .compact { gap: 0; }
  .compact .score { font-size: 1.5em; }
  .compact .time { font-size: 1.1em; }
  .compact .tn { font-size: .85em; }
</style>
