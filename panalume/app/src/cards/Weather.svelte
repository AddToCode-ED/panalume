<script module>
  export const meta = {
    type: 'weather', name: 'Weather', icon: 'mdi:weather-partly-cloudy', category: 'Info',
    size: { w: 380, h: 200 }, tap: 'more-info',
    defaults: { entity: '', forecast: 'daily', days: 5, art: true, animate: true, iconstyle: 'colour' },
    sections: [{ key: 'icon', label: 'Icon' }, { key: 'temp', label: 'Temperature' }, { key: 'cond', label: 'Condition' }, { key: 'humidity', label: 'Humidity' }, { key: 'wind', label: 'Wind' }, { key: 'forecast', label: 'Forecast', section: 'forecast' }, { key: 'low', label: 'Forecast lows' }],
    fields: [
      { key: 'entity', label: 'Weather', type: 'entity', domain: 'weather' },
      { key: 'name', label: 'Name', type: 'text' },
      { key: 'forecast', label: 'Forecast', type: 'select', options: ['daily', 'hourly', 'none'] },
      { key: 'days', label: 'Forecast items', type: 'number' },
      { key: 'hilo', label: "Show today's high and low", type: 'bool' },
      { key: 'art', label: 'Weather art background (a minimal picture of the sky right now)', type: 'bool' },
      { key: 'animate', label: 'Gentle movement in the art (clouds, rain, snow)', type: 'bool' },
      { key: 'iconstyle', label: 'Icons', type: 'select', options: [{ value: 'colour', label: 'Full colour' }, { value: 'line', label: 'Simple line' }] },
    ],
  };
  export const WX = { 'clear-night': 'mdi:weather-night', cloudy: 'mdi:weather-cloudy', exceptional: 'mdi:alert-circle-outline', fog: 'mdi:weather-fog', hail: 'mdi:weather-hail', lightning: 'mdi:weather-lightning', 'lightning-rainy': 'mdi:weather-lightning-rainy', partlycloudy: 'mdi:weather-partly-cloudy', pouring: 'mdi:weather-pouring', rainy: 'mdi:weather-rainy', snowy: 'mdi:weather-snowy', 'snowy-rainy': 'mdi:weather-snowy-rainy', sunny: 'mdi:weather-sunny', windy: 'mdi:weather-windy', 'windy-variant': 'mdi:weather-windy-variant' };
</script>

<script>
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { stateWord } from '../lib/i18n.svelte.js';
  import { subscribe, states } from '../lib/ha.svelte.js';
  import { clock } from '../lib/clock.svelte.js';
  import WeatherIcon from '../components/WeatherIcon.svelte';
  import WeatherArt from '../components/WeatherArt.svelte';
  let { props, w } = $props();
  const show = (k) => !props.hide?.[k];
  const e = $derived(ent(props.entity));
  const a = $derived(e?.attributes || {});
  let forecast = $state([]);
  $effect(() => {
    const id = e?.entity_id;
    const type = props.forecast;
    if (!id || type === 'none') return;
    return subscribe({ type: 'weather/subscribe_forecast', entity_id: id, forecast_type: type }, (ev) => (forecast = ev.forecast || []));
  });
  // Today's high and low come from the daily forecast, whatever the card shows.
  let daily = $state([]);
  $effect(() => {
    const id = e?.entity_id;
    if (!id || !props.hilo || props.forecast === 'daily') return;
    return subscribe({ type: 'weather/subscribe_forecast', entity_id: id, forecast_type: 'daily' }, (ev) => (daily = ev.forecast || []));
  });
  const today = $derived.by(() => {
    if (!props.hilo) return null;
    const list = props.forecast === 'daily' ? forecast : daily;
    const d = new Date().toDateString();
    return list.find((f) => new Date(f.datetime).toDateString() === d) || null;
  });
  const COND = { partlycloudy: 'Partly cloudy', 'clear-night': 'Clear', 'lightning-rainy': 'Thunderstorms', 'snowy-rainy': 'Sleet', 'windy-variant': 'Windy', exceptional: 'Severe weather' };
  const items = $derived(forecast.slice(0, Number(props.days) || 5));

  // Time of day for the art and the icons: from the sun entity when there is one
  // (dawn and dusk are told apart by whether it is rising), else from the clock.
  const sunE = $derived(states.get('sun.sun'));
  const hour = $derived(new Date(clock.now).getHours() + new Date(clock.now).getMinutes() / 60);
  const az = $derived(Number(sunE?.attributes.azimuth));
  const el = $derived(Number(sunE?.attributes.elevation));
  const phase = $derived.by(() => {
    if (Number.isFinite(el)) return el >= 8 ? 'day' : el > -6 ? (sunE.attributes.rising ? 'dawn' : 'dusk') : 'night';
    return hour >= 7 && hour < 18 ? 'day' : hour >= 5 && hour < 7 ? 'dawn' : hour >= 18 && hour < 20 ? 'dusk' : 'night';
  });
  const art = $derived(props.art !== false);
  const colour = $derived(props.iconstyle !== 'line');
  // Hourly forecast items after dark get the moon.
  const nightAt = (f) => { if (props.forecast !== 'hourly') return false; const h = new Date(f.datetime).getHours(); return h < 6 || h >= 20; };
  const label = (f) => props.forecast === 'hourly' ? new Date(f.datetime).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: false }) : new Date(f.datetime).toLocaleDateString([], { weekday: 'short' });
</script>

<div class="wx" class:art>
  {#if art && e}<WeatherArt cond={e.state} {phase} {az} {el} {hour} animate={props.animate !== false} />{/if}
  <div class="now">
    {#if show('icon')}{#if colour}<WeatherIcon c={e?.state} night={phase === 'night'} size="3.4em" />{:else}<Icon icon={WX[e?.state] || 'mdi:weather-cloudy'} size="3.2em" />{/if}{/if}
    <div>
      <div class="trow">
        {#if show('temp')}<div class="temp">{a.temperature != null ? Math.round(a.temperature) + '°' : '—'}</div>{/if}
        {#if today}<div class="hl"><span>↑ {Math.round(today.temperature)}°</span>{#if today.templow != null}<span class="lo">↓ {Math.round(today.templow)}°</span>{/if}</div>{/if}
      </div>
      {#if show('cond')}<div class="cond">{t(props.name) || stateWord('weather', null, e?.state) || COND[e?.state] || (e?.state || '').replace(/-/g, ' ')}</div>{/if}
    </div>
    <div class="extra">
      {#if a.humidity != null && show('humidity')}<span><Icon icon="mdi:water-percent" size="1em" /> {a.humidity}%</span>{/if}
      {#if a.wind_speed != null && show('wind')}<span><Icon icon="mdi:weather-windy" size="1em" /> {Math.round(a.wind_speed)} {a.wind_speed_unit || ''}</span>{/if}
    </div>
  </div>
  {#if items.length && show('forecast')}
    <div class="fc">
      {#each items as f}
        <div class="day">
          <div class="d">{label(f)}</div>
          {#if colour}<WeatherIcon c={f.condition} night={nightAt(f)} size="1.9em" />{:else}<Icon icon={WX[f.condition] || 'mdi:weather-cloudy'} size="1.6em" />{/if}
          <div class="hi">{Math.round(f.temperature)}°</div>
          {#if f.templow != null && show('low')}<div class="lo">{Math.round(f.templow)}°</div>{/if}
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  .wx { position: relative; isolation: isolate; height: 100%; display: flex; flex-direction: column; justify-content: space-between; gap: 8px; }
  /* The art fills the whole card (under its padding) and has its own light text so it reads on any theme. */
  .wx.art { color: #fff; --muted: rgba(255,255,255,.74); text-shadow: 0 1px 6px rgba(0,0,0,.28); }
  .wx.art :global(.art) { z-index: -1; inset: calc(var(--pad) * -1); width: calc(100% + var(--pad) * 2); height: calc(100% + var(--pad) * 2); }
  .now { display: flex; align-items: center; gap: 14px; }
  .temp { font-size: 2.2em; font-weight: 600; line-height: 1; }
  /* Today's high / low sit beside the temperature, so they take no extra height. */
  .trow { display: flex; align-items: center; gap: 8px; }
  .hl { display: flex; flex-direction: column; font-size: .78em; font-weight: 600; line-height: 1.2; }
  .cond { color: var(--muted); text-transform: capitalize; font-size: .9em; }
  .extra { margin-left: auto; display: flex; flex-direction: column; gap: 4px; color: var(--muted); font-size: .85em; text-align: right; }
  .extra span { display: flex; align-items: center; gap: 4px; justify-content: flex-end; }
  .fc { display: flex; justify-content: space-between; gap: 4px; }
  .day { display: flex; flex-direction: column; align-items: center; gap: 2px; font-size: .85em; flex: 1; }
  .d { color: var(--muted); }
  .hi { font-weight: 600; }
  .lo { color: var(--muted); }
</style>
