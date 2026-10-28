# Panalume

Open **Panalume** in the HA sidebar. Press the faint pencil (top-right) or **E** to edit.

- **Add** cards from the panel; drag to move (between sidebar and main area too), drag any edge/corner to resize. Shift = 1px precision, arrows nudge, Delete, Ctrl+D duplicate, Ctrl+Z undo.
- **Card → Content / Style / Actions / Size**: any text field can be an HA template (`{{ states('sensor.x') }}`), including colours and styling.
- **Actions**: tap/hold → toggle, more-info pop-up, pop-up of other cards, navigate, service call, URL.
- **Layout**: one main layout shows on every device — desktop scales it, phone gets an automatic single-column version. Give phone or desktop its own custom layout if you want (Layout tab, or just start dragging on that device). New cards are added to every layout. Pin a screen with `?device=phone` / `desktop`.
- **Groups**: rooms move as one. Double-tap a card to move it alone; Shift/Ctrl-click or drag a box to select several, then Group (Ctrl+G), align or size them together.
- **Show / hide parts** of bigger cards (EV, Octopus, heat pump…) with the chips at the top of Content or the eye next to each field.
- **Theme**: colours, fonts (Google Fonts URL), card glass/blur (set blur 0 on slow tablets), global CSS.

Config is stored in the app's `/data/dashboard.json` (Layout → Export for a backup).

## Energy cards

Added from **Add → Energy**; entities are auto-detected for the Octopus Energy and myenergi integrations (editable after).

- **Octopus rates** — current/next rate, cheapest 2h, half-hourly bars (today / next 24h / two days) with Intelligent dispatches, Octoplus sessions and export rate overlaid.
- **Octoplus sessions** — live session banner with baseline vs this half-hour's usage, upcoming Saving / Power Down / Power Up / Free Electricity sessions with **Join**, points and 12-month stats.
- **Intelligent Octopus** — dispatch status, planned/recent slots, smart/bump charge, ready-by time, charge target.
- **Energy cost today** — cost, kWh, average p/kWh, export, solar, yesterday, peak/off-peak split, half-hourly usage coloured by rate.
- **Heat pump** — live COP/SCOP, kW in/out, flow & outdoor temp, hot water + Boost.
- **Zappi / Eddi** — status, mode buttons, boosts.
- **Electric vehicle** — battery ring vs target, range, charging, plug/lock, climate.
- **Power flow** — solar / grid / battery / home / EV.

## Weather card

Colour icons, and a minimal sky picture behind the card that follows the real sun (it uses `sun.sun` when you have it, otherwise the clock) and the current conditions. Settings: **Weather art background**, **Gentle movement in the art** (turn off on slow tablets) and **Icons** (full colour or simple line).

## Team tracker (sports scores)

Needs the [Team Tracker](https://github.com/vasqued2/ha-teamtracker) integration (HACS), which creates one sensor per team you follow. Add the **Team tracker** card (Info) and pick that sensor; it's chosen for you if you have one.

- Before the game: date and time, venue, TV channel and odds. During it: score, period and clock, and per sport the ball, down and distance, baseball runners and count, last play and win probability. After it: the final score.
- Use the eye chips in the card's settings to hide the league, records, live details, win probability or venue / TV / odds.
- **Large faded logos and team colours in the background** (on by default) put each team's logo big and translucent behind its half of the card.
- **Team logos** are loaded from the sports service's website (ESPN), which is the one place this card goes outside your home network. Untick it to avoid that; the sport's icon is shown instead.

## Wyze cameras (and other go2rtc cameras)

Cameras that Home Assistant can't stream (e.g. Wyze Bridge's MQTT cameras show *"does not support play stream service"*) play over WebRTC straight from **go2rtc**, the video server inside Wyze Bridge.

1. In the Camera card set **Mode → WebRTC from go2rtc / Wyze Bridge** and type the **go2rtc stream name** (the camera's Wyze Bridge name, lowercase with underscores, e.g. `wyze_bulb_cam`). Leave it blank to match it from the picked camera, or to see the available names in an error message.
2. Panalume reaches go2rtc at `http://172.30.32.1:1984` (the Home Assistant machine, where the Wyze Bridge app publishes port 1984). If yours is elsewhere, set **go2rtc_url** in the app's **Configuration** tab (e.g. `http://192.168.1.50:1984`). Fill in **go2rtc_username** / **go2rtc_password** only if you turned on go2rtc's API login.
3. The video itself goes straight from the bridge to your screen, so the bridge's WebRTC port (8889, plus UDP 8189) must be reachable from it, and the bridge's **IP** setting should be the Home Assistant machine's address on your network.

Tap opens it full screen, like any camera.

## Import from ha-fusion

Edit → **Layout → Import from ha-fusion** rebuilds the dashboard from your ha-fusion app: views, rooms, buttons (their state / name / icon / colour / service templates keep working, including `entity_id`), cameras and sidebar items, with tablet, phone and desktop layouts. An **Energy** view is added from auto-detected Octopus / myenergi / solar / EV entities. Icons from other sets (`tabler:`, `mingcute:`, `solar:` …) load from Iconify.
