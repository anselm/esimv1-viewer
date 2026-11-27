<script>
  // The world scored, on a flat map: every country filled by one number from the run's
  // `region.scored` tables at the cursor. It replaced the globe on 2026-11-26 (Anselm: the globe
  // and its coloured borders could not be read, and countries began the run red). The default
  // picture is the change since the run's first table, so a run starts blank and a country
  // reddens as it is knocked back; the rank a country starts with is a separate choice.
  // Countries are placed by the run's `country_index` frame (one-degree cells holding a position in
  // the people module's `countries` table), so no outline file is needed to fill them.
  import { places } from './config.js'
  import { LUT, RAMP_CSS, FIRE_LUT, FIRE_CSS } from './ramp.js'

  let { run, base, cursor = null, source = 'region.scored' } = $props()
  const MODES = [
    { id: 'change', label: 'change since the start', say: 'how far the score has moved from the run\'s first year: red is a loss, blue a gain, grey no change' },
    { id: 'stress', label: 'people under heat stress', say: 'the share of a country\'s people living past the heat thresholds this year' },
    { id: 'score', label: 'rank among countries', say: 'where a country stands among the others on today\'s World Bank indicators, 0 lowest to 1 highest; it says nothing about harm and is the same on the first day as the record it was read from' },
  ]
  let mode = $state('change')
  let band = $state(-1)             // -1 the total, else a position in the bands
  let width = $state(640)
  const height = $derived(Math.round(width / 2))
  let canvas = $state(null)
  let tables = $state(null), index = $state(null), outlines = $state(null), error = $state(null), hover = $state(null)
  const grid = $derived(run?.fields?.country_index?.grid ?? null)
  const countries = $derived((run?.describe?.modules ?? []).map((m) => m.tables?.countries).find(Boolean) ?? null)

  $effect(() => {
    if (!base) return
    let live = true
    fetch(`${base}/log.jsonl?topic=${encodeURIComponent(source)}`).then((r) => (r.ok ? r.text() : '')).then((text) => {
      if (!live) return
      const out = []
      for (const line of text.split('\n')) { if (!line.includes(`"${source}"`)) continue; try { const e = JSON.parse(line); if (e.topic === source) out.push({ t: e.t, ...e.data }) } catch {} }
      tables = out.sort((a, b) => a.t - b.t)
    }).catch((e) => { if (live) error = e.message })
    fetch(`${base}/fields/country_index/00000.f32`).then((r) => { if (!r.ok) throw new Error('this run has no country map'); return r.arrayBuffer() }).then((buf) => {
      const dv = new DataView(buf), n = buf.byteLength >> 2, out = new Float32Array(n)
      for (let k = 0; k < n; k++) out[k] = dv.getFloat32(k * 4, true)
      if (live) index = out
    }).catch((e) => { if (live) error = e.message })
    fetch(places.outlines).then((r) => (r.ok ? r.json() : null)).then((o) => { if (live) outlines = o }).catch(() => {})
    return () => { live = false }
  })

  const table = $derived.by(() => {
    if (!tables?.length) return null
    if (cursor === null) return tables[tables.length - 1]
    let i = 0
    for (let k = 0; k < tables.length; k++) if (tables[k].t <= cursor) i = k
    return tables[i]
  })
  const bands = $derived(table?.bands ?? [])
  const pick = (c) => (c ? (band < 0 ? c.total : c.s?.[band]) : null)
  // the value drawn for one country, or null where the run holds nothing for it
  function valueOf(iso3) {
    const c = table?.by_iso3?.[iso3]; if (!c) return null
    if (mode === 'stress') return c.stress ?? null
    const now = pick(c); if (now == null) return null
    if (mode === 'score') return now
    const then = pick(tables[0].by_iso3?.[iso3]); return then == null ? null : now - then
  }
  // the change scale is fixed over the whole run, so a colour means the same in every year
  const span = $derived.by(() => {
    if (!tables?.length) return 0.05
    let m = 0.02
    for (const t of tables) for (const [k, c] of Object.entries(t.by_iso3 ?? {})) { const a = pick(c), b = pick(tables[0].by_iso3?.[k]); if (a != null && b != null) m = Math.max(m, Math.abs(a - b)) }
    return m
  })
  const STILL = 'still'
  const lut = (v) => {
    // change is one red or one blue whose strength is the size of the change, so a small loss is faint on a
    // light page and on a dark one (the diverging ramp's pale middle shone brightest on dark)
    if (mode === 'change') { const k = Math.min(1, Math.abs(v) / span); return [...(v < 0 ? [214, 64, 60] : [57, 135, 229]), Math.round(70 + 185 * k)] }
    if (mode === 'stress') return v < 0.005 ? STILL : FIRE_LUT[Math.max(0, Math.min(255, Math.round(40 + v * 215)))]
    return LUT[Math.max(0, Math.min(255, Math.round(v * 255)))]
  }

  $effect(() => {
    if (!canvas || !index || !grid || !countries || !table) return
    const { nx, ny } = grid, dpr = Math.min(2, globalThis.devicePixelRatio ?? 1)
    canvas.width = Math.round(width * dpr); canvas.height = Math.round(height * dpr)
    const g = canvas.getContext('2d'), img = new ImageData(nx, ny), px = img.data
    const vals = countries.map((c) => valueOf(c.iso3))
    for (let k = 0; k < nx * ny; k++) {
      const ci = index[k]; if (!(ci >= 0)) continue
      const v = vals[Math.round(ci)], c = v == null ? null : lut(v), o = k * 4
      // no change is a plain grey in both themes (the ramp's pale middle glares on a dark page); no score is fainter
      const still = mode === 'change' && v != null && Math.abs(v) < span * 0.03
      if (c && c !== STILL && !still) { px[o] = c[0]; px[o + 1] = c[1]; px[o + 2] = c[2]; px[o + 3] = c[3] ?? 255 } else { px[o] = px[o + 1] = px[o + 2] = 128; px[o + 3] = v == null ? 40 : 96 }
    }
    const off = new OffscreenCanvas(nx, ny); off.getContext('2d').putImageData(img, 0, 0)
    g.clearRect(0, 0, canvas.width, canvas.height); g.imageSmoothingEnabled = false
    g.drawImage(off, 0, 0, canvas.width, canvas.height)
    if (outlines) {
      g.strokeStyle = 'rgba(128,128,128,0.7)'; g.lineWidth = 0.6 * dpr; g.beginPath()
      const X = (lon) => ((lon - grid.lon_w) / (nx * grid.dlon)) * canvas.width, Y = (lat) => ((grid.lat_n - lat) / (ny * grid.dlat)) * canvas.height
      for (const ring of outlines) { for (let i = 0; i < ring.length; i++) { const [lon, lat] = ring[i]; if (i && Math.abs(lon - ring[i - 1][0]) < 90) g.lineTo(X(lon), Y(lat)); else g.moveTo(X(lon), Y(lat)) } }
      g.stroke()
    }
  })

  function onMove(e) {
    if (!index || !grid || !countries || !table) return
    const r = canvas.getBoundingClientRect(), x = e.clientX - r.left, y = e.clientY - r.top
    const i = Math.floor((x / r.width) * grid.nx), j = Math.floor((y / r.height) * grid.ny), ci = index[j * grid.nx + i]
    const c = ci >= 0 ? countries[Math.round(ci)] : null, row = c ? table.by_iso3?.[c.iso3] : null
    hover = c ? { x, y, name: c.name, iso3: c.iso3, row, then: tables[0].by_iso3?.[c.iso3] } : null
  }
  const f2 = (v) => (v == null ? '—' : v.toFixed(2))
  const signed = (v) => (v == null ? '—' : `${v > 0 ? '+' : v < 0 ? '−' : ''}${Math.abs(v).toFixed(2)}`)
  const say = $derived(MODES.find((m) => m.id === mode).say)
</script>

<figure class="min-w-0">
  <div class="mono mb-1.5 flex flex-wrap items-center gap-x-3 gap-y-1 text-[11px]">
    <span class="inline-flex select-none gap-1" role="radiogroup" aria-label="what the map shows">
      {#each MODES as m (m.id)}
        <button type="button" role="radio" aria-checked={mode === m.id} class="px-1.5 leading-5 rounded-[2px] {mode === m.id ? 'text-brand ring-1 ring-brand/40' : 'muted hover:text-neutral-900 dark:hover:text-neutral-100'}" onclick={() => { mode = m.id }}>{m.label}</button>
      {/each}
    </span>
    {#if mode !== 'stress' && bands.length}
      <label class="muted">of <select bind:value={band} class="rounded-[2px] border border-neutral-300 bg-transparent px-1 dark:border-neutral-700">
        <option value={-1}>the total</option>
        {#each bands as b, i (b.name)}<option value={i}>{b.name}: {b.what}</option>{/each}
      </select></label>
    {/if}
  </div>
  <div class="relative min-w-0" bind:clientWidth={width}>
    <canvas bind:this={canvas} class="block w-full rounded-[3px]" style="height:{height}px" onmousemove={onMove} onmouseleave={() => { hover = null }}></canvas>
    {#if error}<p class="absolute left-2 top-2 text-sm text-leak">{error}</p>
    {:else if !tables || !index}<p class="absolute left-2 top-2 muted text-sm">loading the scores…</p>
    {:else if !table}<p class="absolute left-2 top-2 muted text-sm">no scores in this run</p>{/if}
    {#if hover}
      <div class="pointer-events-none absolute z-10 rounded-[3px] border border-neutral-200 bg-white/95 px-2 py-1.5 text-xs shadow-sm dark:border-neutral-700 dark:bg-neutral-900/95" style="left:{Math.min(hover.x + 12, width - 200)}px; top:{Math.max(hover.y - 10, 0)}px">
        <div class="font-medium">{hover.name} <span class="mono muted">{hover.iso3}</span></div>
        {#if hover.row}
          <div class="mono">total {f2(hover.row.total)} <span class="muted">({signed(hover.then ? hover.row.total - hover.then.total : null)} since the start)</span></div>
          {#if hover.row.stress > 0.005}<div class="mono text-leak">heat stress: {Math.round(hover.row.stress * 100)}% of its people</div>{/if}
          {#each bands as b, i (b.name)}
            <div class="mono flex gap-1.5"><span class="w-16">{b.name}</span><span>{f2(hover.row.s?.[i])}</span><span class="muted">{hover.then?.s?.[i] != null && hover.row.s?.[i] != null && Math.abs(hover.row.s[i] - hover.then.s[i]) >= 0.005 ? signed(hover.row.s[i] - hover.then.s[i]) : ''}</span></div>
          {/each}
        {:else}<div class="muted">no score in this run</div>{/if}
      </div>
    {/if}
  </div>
  <div class="mono mt-1.5 flex flex-wrap items-center gap-x-3 gap-y-1 text-[10px] muted">
    {#if table}<span>{table.year}</span>{/if}
    <span>{mode === 'change' ? `−${span.toFixed(2)} loss` : mode === 'stress' ? 'few' : '0 lowest'}</span>
    <span class="h-2 w-40 rounded-[2px]" style="background:{mode === 'change' ? 'linear-gradient(to right, rgb(214,64,60), rgba(214,64,60,0.27) 49%, rgba(57,135,229,0.27) 51%, rgb(57,135,229))' : mode === 'stress' ? FIRE_CSS : RAMP_CSS}" aria-hidden="true"></span>
    <span>{mode === 'change' ? `+${span.toFixed(2)} gain` : mode === 'stress' ? 'all of its people' : '1 highest'}</span>
    {#if mode !== 'score'}<span class="inline-flex items-center gap-1"><span class="inline-block h-2 w-3 rounded-[1px]" style="background:rgba(128,128,128,0.38)" aria-hidden="true"></span>{mode === 'stress' ? 'none' : 'no change'}</span>{/if}
    <span class="inline-flex items-center gap-1"><span class="inline-block h-2 w-3 rounded-[1px]" style="background:rgba(128,128,128,0.16)" aria-hidden="true"></span>no score</span>
  </div>
  <figcaption class="mt-1 max-w-3xl text-xs muted">Each country is filled by {say}. Hover a country for its nine bands.</figcaption>
</figure>
