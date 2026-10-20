<script module>
  export const meta = {
    type: 'alarm', name: 'Alarm panel', icon: 'mdi:shield-home', category: 'Controls',
    size: { w: 300, h: 380 }, tap: 'none',
    defaults: { entity: '', keypad: true },
    fields: [
      { key: 'entity', label: 'Alarm', type: 'entity', domain: 'alarm_control_panel' },
      { key: 'name', label: 'Name', type: 'text' },
      { key: 'keypad', label: 'Keypad', type: 'bool' },
    ],
  };
</script>

<script>
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { name, stateText } from '../lib/entity.js';
  import { callService } from '../lib/ha.svelte.js';
  let { props, w = 300, h = 380 } = $props();
  const e = $derived(ent(props.entity));
  let code = $state('');
  const armed = $derived(e?.state?.startsWith('armed') || e?.state === 'triggered');
  const go = (s) => { callService('alarm_control_panel', s, code ? { code } : {}, { entity_id: e.entity_id }); code = ''; };
  // Everything (buttons and keypad too) scales with the card, so resizing it
  // keeps the same look instead of squashing or cutting off the keypad.
  const k = $derived(Math.min(1.6, Math.max(0.5, Math.min(w / 300, h / (props.keypad ? 380 : 150)))));
  const color = $derived(e?.state === 'triggered' ? '#ff5b5b' : armed ? '#ff9f43' : '#5bd88f');
</script>

<div class="al" style="--ac:{color};font-size:{(k * 100).toFixed(1)}%">
  <div class="head"><Icon icon={armed ? 'mdi:shield-lock' : 'mdi:shield-off-outline'} size="1.8em" style="color:var(--ac)" />
    <div><div class="name">{t(props.name) || name(e)}</div><div class="state">{stateText(e)}</div></div></div>
  <div class="acts" data-stop>
    {#if armed}<button onclick={() => go('alarm_disarm')}>Disarm</button>
    {:else}<button onclick={() => go('alarm_arm_home')}>Home</button><button onclick={() => go('alarm_arm_away')}>Away</button><button onclick={() => go('alarm_arm_night')}>Night</button>{/if}
  </div>
  {#if props.keypad}
    <div class="code">{'•'.repeat(code.length) || ' '}</div>
    <div class="pad" data-stop>
      {#each ['1', '2', '3', '4', '5', '6', '7', '8', '9', 'C', '0', '⌫'] as k}
        <button onclick={() => (k === 'C' ? (code = '') : k === '⌫' ? (code = code.slice(0, -1)) : (code += k))}>{k}</button>
      {/each}
    </div>
  {/if}
</div>

<style>
  .al { height: 100%; display: flex; flex-direction: column; gap: .65em; min-height: 0; }
  .head { display: flex; align-items: center; gap: .75em; }
  .name { font-weight: 600; }
  .state { color: var(--ac); font-size: .9em; }
  .acts { display: flex; gap: .4em; }
  .acts button { flex: 1; padding: .65em; border-radius: .75em; border: 0; background: color-mix(in srgb, var(--ac) 25%, transparent); color: inherit; font: inherit; font-weight: 600; }
  .code { text-align: center; letter-spacing: .4em; font-size: 1.4em; min-height: 1.4em; }
  .pad { flex: 1; min-height: 0; display: grid; grid-template-columns: repeat(3, 1fr); grid-auto-rows: minmax(0, 1fr); gap: .4em; }
  .pad button { min-height: 0; padding: 0; border-radius: .9em; border: 0; background: rgba(255,255,255,.07); color: inherit; font: inherit; font-size: 1.2em; }
  .pad button:active { background: rgba(255,255,255,.2); }
</style>
