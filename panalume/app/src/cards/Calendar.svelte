<script module>
  import { states } from '../lib/ha.svelte.js';
  export const meta = {
    type: 'calendar', name: 'Calendar', icon: 'mdi:calendar-month-outline', category: 'Info',
    size: { w: 360, h: 320 }, tap: 'none',
    defaults: { entities: [], days: 7, max: 20 },
    // Every calendar in Home Assistant, up to six.
    autofill: () => ({ entities: [...states.keys()].filter((id) => id.startsWith('calendar.')).sort().slice(0, 6) }),
    sections: [{ key: 'head', label: 'Title & date' }, { key: 'location', label: 'Locations' }, { key: 'legend', label: 'Calendar names' }],
    fields: [
      { key: 'title', label: 'Title (blank = today’s date)', type: 'text' },
      { key: 'entities', label: 'Calendars', type: 'entities', domain: 'calendar' },
      { key: 'days', label: 'Days ahead', type: 'number' },
      { key: 'max', label: 'Most events shown', type: 'number' },
    ],
  };
</script>

<script>
  // Today's and upcoming events from Home Assistant's calendars (Google, CalDAV,
  // Local calendar, holidays…), grouped by day. Each calendar has its own colour;
  // events on now are highlighted, and tapping one shows its details.
  import Icon from '../components/Icon.svelte';
  import { t } from '../lib/tpl.js';
  import { clock } from '../lib/clock.svelte.js';
  let { props, w = 360, h = 320 } = $props();
  const show = (k) => !props.hide?.[k];
  const COLORS = ['#4fb4ff', '#ff8fb1', '#5bd88f', '#ffc861', '#b48cff', '#ff9a3c', '#2ed3c4', '#ff6b6b'];

  const cals = $derived((props.entities || []).filter((id) => id?.startsWith('calendar.')));
  const days = $derived(Math.min(31, Math.max(1, Number(props.days) || 7)));
  // Refetch when a calendar's state changes (its current / next event moved),
  // and every 5 minutes anyway.
  const stamp = $derived(cals.map((id) => states.get(id)?.last_updated || '').join('|'));
  let events = $state([]);
  let error = $state('');
  let refresh = $state(0);
  $effect(() => {
    const i = setInterval(() => refresh++, 5 * 60e3);
    return () => clearInterval(i);
  });
  $effect(() => {
    refresh; stamp;
    const list = cals, n = days;
    if (!list.length) return void (events = []);
    const from = new Date(); from.setHours(0, 0, 0, 0);
    const to = new Date(from); to.setDate(to.getDate() + n);
    const q = `?start=${encodeURIComponent(from.toISOString())}&end=${encodeURIComponent(to.toISOString())}`;
    let alive = true;
    Promise.all(list.map((id, i) => fetch(`ha/api/calendars/${id}${q}`).then((r) => (r.ok ? r.json() : Promise.reject(new Error(`${id}: ${r.status}`)))).then((evs) => evs.map((ev) => norm(ev, id, i)))))
      .then((all) => { if (alive) { events = all.flat().filter(Boolean).sort((a, b) => a.start - b.start || b.allDay - a.allDay); error = ''; } })
      .catch((e) => { if (alive) error = e.message || 'Could not load the calendars'; });
    return () => (alive = false);
  });

  // All-day events come as dates (end exclusive); timed ones as date-times.
  const day0 = (d) => { const x = new Date(d); x.setHours(0, 0, 0, 0); return x; };
  const fromDate = (s) => { const [y, m, d] = s.split('-').map(Number); return new Date(y, m - 1, d); };
  function norm(ev, cal, i) {
    const allDay = !!ev.start?.date && !ev.start?.dateTime;
    const start = allDay ? fromDate(ev.start.date) : new Date(ev.start?.dateTime);
    const end = allDay ? fromDate(ev.end?.date || ev.start.date) : new Date(ev.end?.dateTime || ev.start?.dateTime);
    if (isNaN(start)) return null;
    return { key: `${cal}|${ev.uid || ''}|${start.getTime()}|${ev.summary}`, cal, color: COLORS[i % COLORS.length], title: ev.summary || '(No title)', location: ev.location || '', description: ev.description || '', allDay, start, end };
  }

  // Grouped by day; an event spanning several days shows on each of them.
  const now = $derived(clock.now);
  const groups = $derived.by(() => {
    const out = [];
    const today = day0(now);
    let left = Math.max(1, Number(props.max) || 20);
    for (let d = 0; d < days && left > 0; d++) {
      const date = new Date(today); date.setDate(date.getDate() + d);
      const next = new Date(date); next.setDate(next.getDate() + 1);
      // On this day (end is exclusive; a zero-length event counts at its start)…
      const onDay = (e) => e.start < next && (e.end > date || (+e.end === +e.start && e.start >= date));
      // …and not already over (finished events drop off during the day).
      const notOver = (e) => e.allDay || e.end > now || (+e.end === +e.start && e.start >= now - 60e3);
      const evs = events.filter((e) => onDay(e) && notOver(e));
      if (!evs.length) continue;
      const take = evs.slice(0, left);
      left -= take.length;
      out.push({ date, evs: take });
    }
    return out;
  });
  const empty = $derived(cals.length && !error && !groups.length);

  const lang = undefined;
  const dayName = (d) => {
    const diff = Math.round((day0(d) - day0(now)) / 864e5);
    return diff === 0 ? 'Today' : diff === 1 ? 'Tomorrow' : d.toLocaleDateString(lang, { weekday: diff < 7 ? 'long' : 'short', ...(diff < 7 ? {} : { day: 'numeric', month: 'short' }) });
  };
  const hm = (d) => d.toLocaleTimeString(lang, { hour: '2-digit', minute: '2-digit', hour12: false });
  const when = (e, date) => {
    if (e.allDay) return 'All day';
    const next = new Date(date); next.setDate(next.getDate() + 1);
    const startsToday = e.start >= date, endsToday = e.end <= next;
    if (startsToday && (endsToday || small)) return +e.start === +e.end || small ? hm(e.start) : `${hm(e.start)}–${hm(e.end)}`;
    if (startsToday) return `From ${hm(e.start)}`;
    if (endsToday) return `Until ${hm(e.end)}`;
    return 'All day';
  };
  const isNow = (e) => !e.allDay && e.start <= now && e.end > now;
  const calName = (id) => (states.get(id)?.attributes.friendly_name || id.slice(9)).replace(/\s*calendar$/i, '');
  let open = $state(null);
  const small = $derived(h < 170);
  const title = $derived(t(props.title) || new Date(now).toLocaleDateString(lang, { weekday: 'long', day: 'numeric', month: 'long' }));
</script>

<div class="cal" class:small>
  {#if show('head') && !small}
    <div class="head">
      <Icon icon="mdi:calendar-month-outline" size="1.2em" />
      <span class="t">{title}</span>
      {#if show('legend') && cals.length > 1 && w >= 300}
        <span class="legend">{#each cals as id, i}<i style="background:{COLORS[i % COLORS.length]}" title={calName(id)}></i>{/each}</span>
      {/if}
    </div>
  {/if}

  {#if !cals.length}
    <div class="msg">Pick one or more calendars in the card's settings.</div>
  {:else if error}
    <div class="msg">{error}</div>
  {:else if empty}
    <div class="msg">Nothing on in the next {days === 1 ? 'day' : `${days} days`}</div>
  {:else}
    <div class="list" data-stop>
      {#each groups as g (g.date.getTime())}
        <div class="day" class:today={dayName(g.date) === 'Today'}>{dayName(g.date)}</div>
        {#each g.evs as e (e.key)}
          <button class="ev" class:now={isNow(e)} class:open={open === e.key} style="--c:{e.color}" onclick={() => (open = open === e.key ? null : e.key)}>
            <span class="bar"></span>
            <span class="when">{when(e, g.date)}</span>
            <span class="what">
              <span class="ti">{e.title}</span>
              {#if open === e.key}
                {#if e.location}<span class="sub"><Icon icon="mdi:map-marker-outline" size="1em" />{e.location}</span>{/if}
                {#if e.description}<span class="desc">{e.description}</span>{/if}
                {#if show('legend') && cals.length > 1}<span class="sub"><Icon icon="mdi:calendar" size="1em" />{calName(e.cal)}</span>{/if}
              {:else if e.location && show('location') && !small}<span class="sub">{e.location}</span>{/if}
            </span>
          </button>
        {/each}
      {/each}
    </div>
  {/if}
</div>

<style>
  .cal { height: 100%; display: flex; flex-direction: column; gap: 8px; min-height: 0; }
  .head { display: flex; align-items: center; gap: 8px; font-weight: 600; }
  .head :global(svg) { color: var(--accent); flex: none; }
  .t { flex: 1; min-width: 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .legend { display: flex; gap: 4px; flex: none; }
  .legend i { width: 8px; height: 8px; border-radius: 50%; }
  .msg { flex: 1; display: grid; place-items: center; color: var(--muted); font-size: .85em; text-align: center; padding: 8px; }
  .list { flex: 1; min-height: 0; overflow-y: auto; display: flex; flex-direction: column; gap: 3px; margin-right: -6px; padding-right: 6px; scrollbar-width: thin; }
  .day { font-size: .72em; font-weight: 600; letter-spacing: .06em; text-transform: uppercase; color: var(--muted); margin-top: 6px; }
  .day:first-child { margin-top: 0; }
  .day.today { color: var(--accent); }
  .ev { display: flex; align-items: flex-start; gap: 9px; width: 100%; text-align: left; background: rgba(255,255,255,.04); border: 0; border-radius: 10px; padding: 7px 10px 7px 0; color: inherit; font: inherit; cursor: pointer; }
  .ev.now { background: color-mix(in srgb, var(--c) 16%, transparent); }
  .bar { width: 3px; align-self: stretch; border-radius: 2px; background: var(--c); flex: none; margin-left: 6px; }
  .when { flex: none; width: 6.9em; white-space: nowrap; font-size: .8em; color: var(--muted); font-variant-numeric: tabular-nums; padding-top: .1em; }
  .ev.now .when { color: var(--c); font-weight: 600; }
  .what { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 2px; }
  .ti { font-size: .9em; font-weight: 500; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .ev.open .ti { white-space: normal; }
  .sub { display: flex; align-items: center; gap: 4px; font-size: .75em; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .desc { font-size: .78em; color: var(--muted); white-space: pre-line; max-height: 8em; overflow: auto; }
  .small .day:not(.today) { margin-top: 2px; }
  .small .ev { padding-top: 5px; padding-bottom: 5px; }
  .small .when { width: 3.6em; }
  /* Fade out at the bottom when there's more to scroll to. */
  .list { mask-image: linear-gradient(to bottom, #000 calc(100% - 18px), transparent); }
</style>
