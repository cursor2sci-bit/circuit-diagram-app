<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Circuit Visualizer — Poe Edition</title>
  <style>
    :root {
      --bg: #0f1419; --surface: #1a2332; --surface2: #243044; --border: #2d3f56;
      --text: #e8eef4; --muted: #94a3b8; --accent: #3b82f6; --warn: #f59e0b; --ok: #22c55e; --err: #ef4444;
      --mono: ui-monospace, "Cascadia Code", Consolas, monospace;
    }
    * { box-sizing: border-box; }
    body { margin: 0; font-family: system-ui, sans-serif; background: var(--bg); color: var(--text); line-height: 1.5; }
    .wrap { max-width: 1200px; margin: 0 auto; padding: 12px; }
    h1 { margin: 0 0 4px; font-size: 1.35rem; }
    .sub { color: var(--muted); font-size: 0.9rem; margin-bottom: 12px; }
    .panel { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; margin-bottom: 10px; overflow: hidden; }
    .ph { padding: 8px 12px; border-bottom: 1px solid var(--border); display: flex; gap: 8px; flex-wrap: wrap; align-items: center; }
    .ph h2 { margin: 0; font-size: 0.95rem; flex: 1; }
    button, .btn {
      padding: 6px 12px; border-radius: 6px; border: 1px solid var(--border);
      background: var(--surface2); color: var(--text); cursor: pointer; font-size: 0.85rem;
    }
    button.primary { background: var(--accent); border-color: var(--accent); color: #fff; }
    button.chip { padding: 3px 10px; border-radius: 999px; font-size: 0.78rem; }
    button:hover { filter: brightness(1.1); }
    input[type=text], textarea {
      width: 100%; padding: 8px; border-radius: 6px; border: 1px solid var(--border);
      background: var(--surface2); color: var(--text); font-family: var(--mono); font-size: 12px;
    }
    textarea { min-height: 200px; resize: vertical; }
    .grid { display: grid; grid-template-columns: 1fr 1.2fr; gap: 10px; }
    @media (max-width: 800px) { .grid { grid-template-columns: 1fr; } }
    .err { color: #fca5a5; font-size: 0.85rem; padding: 6px 12px; background: rgba(239,68,68,.12); }
    .hint { color: var(--muted); font-size: 0.8rem; padding: 6px 12px; }
    #svg { width: 100%; height: 380px; background: #121a24; display: block; }
    .stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 8px; padding: 10px 12px; }
    .stat { background: var(--surface2); border-radius: 6px; padding: 8px 10px; }
    .stat b { display: block; font-size: 0.75rem; color: var(--muted); }
    .stat span { font-family: var(--mono); color: var(--ok); font-weight: 600; }
    ol.steps { margin: 0; padding: 8px 12px 12px 28px; color: var(--muted); font-size: 0.88rem; }
    .slider-row { display: flex; align-items: center; gap: 10px; padding: 4px 12px; font-size: 0.85rem; }
    .slider-row input { flex: 1; accent-color: var(--accent); }
    @keyframes flow { to { stroke-dashoffset: -40; } }
    .flow { animation: flow 1s linear infinite; }
    .zoom-bar { display: flex; gap: 6px; align-items: center; padding: 6px 12px; border-bottom: 1px solid var(--border); font-size: 0.8rem; color: var(--muted); }
  </style>
</head>
<body>
<div class="wrap">
  <h1>⚡ Circuit Visualizer</h1>
  <p class="sub">Physics study tool — paste circuit JSON from Poe, edit code, watch current flow</p>

  <div class="panel">
    <div class="ph">
      <input type="text" id="textIn" placeholder='Describe: "12V battery with 4 ohm and 8 ohm in series"' style="flex:2" />
      <button class="primary" onclick="parseSimpleText()">Quick parse</button>
      <button class="chip" onclick="loadEx('series')">Series</button>
      <button class="chip" onclick="loadEx('parallel')">Parallel</button>
      <button class="chip" onclick="loadEx('mixed')">Mixed</button>
    </div>
    <div class="hint">Tip: Ask any Poe bot to output circuit JSON (use prompt in POE_BOT_PROMPT.txt), then paste below and click Apply.</div>
  </div>

  <div class="grid">
    <div class="panel">
      <div class="ph">
        <h2>Circuit code (JSON)</h2>
        <button onclick="formatCode()">Format</button>
        <button class="primary" onclick="applyCode()">Apply</button>
      </div>
      <div id="codeErr" class="err" style="display:none"></div>
      <div style="padding:8px"><textarea id="code"></textarea></div>
    </div>
    <div class="panel">
      <div class="ph">
        <h2>Diagram</h2>
        <label style="font-size:0.8rem;color:var(--muted)"><input type="checkbox" id="overlays" checked onchange="render()"> Labels</label>
        <button onclick="paused=!paused;render()">Pause/Play flow</button>
      </div>
      <div class="zoom-bar">
        <button class="chip" onclick="zoom=Math.min(2.5,zoom+.2);render()">+</button>
        <span id="zoomLbl">100%</span>
        <button class="chip" onclick="zoom=Math.max(.5,zoom-.2);render()">−</button>
        <button class="chip" onclick="zoom=1;panX=0;panY=0;render()">Reset</button>
        <span style="margin-left:auto">Scroll on diagram to zoom</span>
      </div>
      <svg id="svg" viewBox="0 0 700 420"></svg>
      <div id="solveErr" class="err" style="display:none"></div>
    </div>
  </div>

  <div class="panel">
    <div class="ph"><h2>Physics results</h2></div>
    <div class="stats" id="stats"></div>
    <div id="sliders"></div>
    <ol class="steps" id="steps"></ol>
  </div>
</div>

<script>
const EXAMPLES = {
  series: {"name":"Series resistors","components":[{"id":"V1","type":"voltage_source","value":12,"unit":"V","nodes":["n0","n1"]},{"id":"R1","type":"resistor","value":4,"unit":"ohm","nodes":["n1","n2"]},{"id":"R2","type":"resistor","value":8,"unit":"ohm","nodes":["n2","n0"]}],"layout":{"V1":{"x":80,"y":200},"R1":{"x":280,"y":200},"R2":{"x":480,"y":200}}},
  parallel: {"name":"Parallel resistors","components":[{"id":"V1","type":"voltage_source","value":9,"unit":"V","nodes":["gnd","top"]},{"id":"R1","type":"resistor","value":6,"unit":"ohm","nodes":["top","gnd"]},{"id":"R2","type":"resistor","value":3,"unit":"ohm","nodes":["top","gnd"]}],"layout":{"V1":{"x":100,"y":280},"R1":{"x":300,"y":160},"R2":{"x":300,"y":320}}},
  mixed: {"name":"Mixed circuit","components":[{"id":"V1","type":"voltage_source","value":12,"unit":"V","nodes":["n0","n1"]},{"id":"R1","type":"resistor","value":4,"unit":"ohm","nodes":["n1","n2"]},{"id":"R2","type":"resistor","value":12,"unit":"ohm","nodes":["n2","n3"]},{"id":"R3","type":"resistor","value":6,"unit":"ohm","nodes":["n2","n0"]},{"id":"R4","type":"resistor","value":6,"unit":"ohm","nodes":["n3","n0"]}],"layout":{"V1":{"x":60,"y":220},"R1":{"x":200,"y":220},"R2":{"x":360,"y":160},"R3":{"x":360,"y":280},"R4":{"x":520,"y":220}}}
};

let circuit = EXAMPLES.series;
let paused = false;
let zoom = 1, panX = 0, panY = 0;
let dragging = null, dragOff = {x:0,y:0};

const codeEl = document.getElementById('code');
const svg = document.getElementById('svg');

function validateCircuit(data) {
  const errors = [];
  if (!data || typeof data !== 'object') return { ok: false, errors: ['Not an object'] };
  if (!Array.isArray(data.components) || !data.components.length) errors.push('components array required');
  const ids = new Set();
  for (const c of data.components || []) {
    if (!c.id) errors.push('component missing id');
    if (ids.has(c.id)) errors.push('duplicate id: ' + c.id);
    ids.add(c.id);
    if (!c.type || !c.nodes || c.nodes.length !== 2) errors.push(c.id + ': invalid component');
    if (c.type === 'voltage_source' && c.value == null) errors.push(c.id + ': needs value (V)');
    if ((c.type === 'resistor' || c.type === 'bulb') && c.value == null) errors.push(c.id + ': needs value (ohm)');
    if (c.type === 'switch' && c.closed === undefined) c.closed = true;
  }
  return errors.length ? { ok: false, errors } : { ok: true, circuit: data };
}

function getG(c) {
  if (c.type === 'resistor' || c.type === 'bulb') return c.value > 0 ? 1/c.value : null;
  if (c.type === 'ammeter') return 1000;
  if (c.type === 'voltmeter') return 1e-9;
  if (c.type === 'wire') return 1e6;
  if (c.type === 'switch') return c.closed !== false ? 1e6 : 0;
  return null;
}

function solve(circ) {
  const nodes = new Set();
  circ.components.forEach(c => { nodes.add(c.nodes[0]); nodes.add(c.nodes[1]); });
  const nodeList = [...nodes];
  const gnd = nodeList.includes('gnd') ? 'gnd' : nodeList.includes('n0') ? 'n0' : nodeList[0];
  const unk = nodeList.filter(n => n !== gnd);
  if (!unk.length) return { ok: false, error: 'No nodes to solve.' };

  const n = unk.length;
  const G = Array.from({length:n}, () => new Array(n).fill(0));
  const I = new Array(n).fill(0);

  for (const comp of circ.components) {
    const [a,b] = comp.nodes;
    if (comp.type === 'voltage_source') {
      const Rint = 0.01, g = 1/Rint, srcV = comp.value || 0;
      const stamp = (node, sign) => {
        if (node === gnd) return;
        const i = unk.indexOf(node);
        if (i >= 0) { G[i][i] += g; I[i] += sign * g * srcV; }
      };
      stamp(a, 1); stamp(b, -1);
      if (a !== gnd && b !== gnd) {
        const i = unk.indexOf(a), j = unk.indexOf(b);
        G[i][j] -= g; G[j][i] -= g;
      }
      continue;
    }
    const g = getG(comp);
    if (g == null || g === 0) continue;
    const stampR = (n1, n2) => {
      if (n1 !== gnd) { const i = unk.indexOf(n1); G[i][i] += g; }
      if (n2 !== gnd) { const j = unk.indexOf(n2); G[j][j] += g; }
      if (n1 !== gnd && n2 !== gnd) {
        const i = unk.indexOf(n1), j = unk.indexOf(n2);
        G[i][j] -= g; G[j][i] -= g;
      }
    };
    stampR(a, b);
  }

  const M = G.map((row,i) => [...row, I[i]]);
  for (let col = 0; col < n; col++) {
    let piv = col;
    for (let r = col+1; r < n; r++) if (Math.abs(M[r][col]) > Math.abs(M[piv][col])) piv = r;
    if (Math.abs(M[piv][col]) < 1e-12) return { ok: false, error: 'Cannot solve (open/short circuit).' };
    [M[col], M[piv]] = [M[piv], M[col]];
    for (let r = col+1; r < n; r++) {
      const f = M[r][col]/M[col][col];
      for (let j = col; j <= n; j++) M[r][j] -= f*M[col][j];
    }
  }
  const v = new Array(n).fill(0);
  for (let r = n-1; r >= 0; r--) {
    let s = M[r][n];
    for (let c = r+1; c < n; c++) s -= M[r][c]*v[c];
    v[r] = s / M[r][r];
  }

  const nodeV = { [gnd]: 0 };
  unk.forEach((node, i) => nodeV[node] = v[i]);

  const branches = [];
  let srcI = 0;
  for (const comp of circ.components) {
    const [a,b] = comp.nodes;
    const va = nodeV[a]||0, vb = nodeV[b]||0, vD = va-vb;
    let cur = 0;
    if (comp.type === 'voltage_source') {
      cur = (vD - (comp.value||0)) / 0.01;
      srcI = Math.abs(cur);
    } else {
      const g = getG(comp);
      if (g && g < 1e5) cur = vD * g;
      else if (g >= 1e5) cur = vD * 1e6;
    }
    branches.push({ id: comp.id, cur, vD, p: Math.abs(cur*vD) });
  }

  const steps = [];
  const vs = circ.components.find(c => c.type === 'voltage_source');
  if (vs) steps.push(`Voltage source ${vs.id} = ${vs.value} V.`);
  branches.forEach(b => {
    const c = circ.components.find(x => x.id === b.id);
    if (c && (c.type === 'resistor' || c.type === 'bulb'))
      steps.push(`${b.id}: ΔV=${Math.abs(b.vD).toFixed(2)}V, I=${Math.abs(b.cur).toFixed(3)}A, P=${b.p.toFixed(2)}W`);
  });
  if (srcI > 0) steps.push(`Total source current I = ${srcI.toFixed(3)} A (conventional + → −).`);
  const req = vs && srcI > 1e-9 ? vs.value/srcI : null;

  return { ok: true, nodeV, branches, srcI, req, steps };
}

function defaultLayout(circ) {
  const layout = { ...(circ.layout||{}) };
  circ.components.forEach((c,i) => {
    if (!layout[c.id]) layout[c.id] = { x: 80+i*140, y: 200 };
  });
  return layout;
}

function sym(comp) {
  const v = comp.value;
  switch (comp.type) {
    case 'resistor':
      return `<polyline points="-30,0 -22,-12 -14,12 -6,-12 2,12 10,-12 18,12 26,-12 30,0" fill="none" stroke="#e8eef4" stroke-width="2"/>
        <text y="-20" text-anchor="middle" fill="#94a3b8" font-size="11">${comp.id}</text>
        <text y="26" text-anchor="middle" fill="#f59e0b" font-size="10">${v}Ω</text>`;
    case 'voltage_source':
      return `<line x1="-20" y1="0" x2="-8" y2="0" stroke="#e8eef4" stroke-width="2"/>
        <line x1="8" y1="0" x2="20" y2="0" stroke="#e8eef4" stroke-width="2"/>
        <line x1="-8" y1="-14" x2="-8" y2="14" stroke="#e8eef4" stroke-width="3"/>
        <line x1="8" y1="-8" x2="8" y2="8" stroke="#e8eef4" stroke-width="3"/>
        <text y="-24" text-anchor="middle" fill="#94a3b8" font-size="11">${comp.id} ${v}V</text>`;
    case 'switch':
      return `<circle cx="-20" cy="0" r="3" fill="#e8eef4"/><circle cx="20" cy="0" r="3" fill="#e8eef4"/>
        ${comp.closed!==false ? '<line x1="-20" y1="0" x2="20" y2="0" stroke="#e8eef4" stroke-width="2"/>' : '<line x1="-20" y1="0" x2="15" y2="-12" stroke="#e8eef4" stroke-width="2"/>'}
        <text y="-20" text-anchor="middle" fill="#94a3b8" font-size="10">${comp.id}</text>`;
    case 'bulb':
      return `<circle r="16" fill="none" stroke="#e8eef4" stroke-width="2"/>
        <line x1="-10" y1="-10" x2="10" y2="10" stroke="#e8eef4"/><line x1="10" y1="-10" x2="-10" y2="10" stroke="#e8eef4"/>
        <text y="-22" text-anchor="middle" fill="#94a3b8" font-size="11">${comp.id} ${v}Ω</text>`;
    default:
      return `<line x1="-30" y1="0" x2="30" y2="0" stroke="#64748b" stroke-width="2"/>`;
  }
}

function render() {
  const layout = defaultLayout(circuit);
  const sol = solve(circuit);
  const show = document.getElementById('overlays').checked;
  document.getElementById('zoomLbl').textContent = Math.round(zoom*100)+'%';
  svg.setAttribute('viewBox', `${-panX} ${-panY} ${700/zoom} ${420/zoom}`);

  const se = document.getElementById('solveErr');
  if (!sol.ok) { se.style.display='block'; se.textContent=sol.error; }
  else se.style.display='none';

  const branchMap = {};
  if (sol.ok) sol.branches.forEach(b => branchMap[b.id]=b);
  const maxP = sol.ok ? Math.max(...sol.branches.map(b=>b.p), 0.001) : 1;

  const nodes = {};
  circuit.components.forEach(c => {
    const p = layout[c.id];
    if (!p) return;
    if (!nodes[c.nodes[0]]) nodes[c.nodes[0]] = {x:p.x-40,y:p.y};
    if (!nodes[c.nodes[1]]) nodes[c.nodes[1]] = {x:p.x+40,y:p.y};
  });

  let html = `<defs><marker id="ah" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><polygon points="0 0,8 3,0 6" fill="#f59e0b"/></marker></defs>`;

  const nlist = Object.entries(nodes);
  for (let i=0;i<nlist.length;i++) for (let j=i+1;j<nlist.length;j++) {
    const [na,pa]=nlist[i],[nb,pb]=nlist[j];
    const conn = circuit.components.some(c=>(c.nodes[0]===na&&c.nodes[1]===nb)||(c.nodes[0]===nb&&c.nodes[1]===na));
    if (conn) html += `<line x1="${pa.x}" y1="${pa.y}" x2="${pb.x}" y2="${pb.y}" stroke="#64748b" stroke-width="2"/>`;
  }

  if (sol.ok && !paused) {
    circuit.components.forEach(c => {
      const b = branchMap[c.id], p = layout[c.id];
      if (!b || !p || Math.abs(b.cur)<1e-6) return;
      const rev = b.cur < 0 ? ' style="animation-direction:reverse"' : '';
      const dur = 1/Math.min(Math.abs(b.cur)*3,4);
      html += `<line class="flow" x1="${p.x-28}" y1="${p.y}" x2="${p.x+28}" y2="${p.y}" stroke="#f59e0b" stroke-width="3" stroke-dasharray="8 12" marker-end="url(#ah)" style="animation-duration:${dur}s${rev?' ;animation-direction:reverse':''}"/>`;
    });
  }

  circuit.components.forEach(c => {
    const p = layout[c.id]; if (!p) return;
    const b = branchMap[c.id];
    const heat = b ? `rgba(255,${Math.round(80*(1-b.p/maxP))},${Math.round(80*(1-b.p/maxP))},0.3)` : 'transparent';
    html += `<g transform="translate(${p.x},${p.y})" data-id="${c.id}" style="cursor:grab">
      <rect x="-40" y="-25" width="80" height="50" fill="${heat}" rx="5"/>
      ${sym(c)}
      ${show&&b&&c.type!=='wire'?`<text y="38" text-anchor="middle" fill="#fbbf24" font-size="9" font-family="monospace">I=${Math.abs(b.cur).toFixed(2)}A ΔV=${Math.abs(b.vD).toFixed(1)}V</text>`:''}
    </g>`;
  });

  if (show && sol.ok) {
    Object.entries(nodes).forEach(([nid,pos]) => {
      if (sol.nodeV[nid]!==undefined)
        html += `<text x="${pos.x}" y="${pos.y-14}" text-anchor="middle" fill="#60a5fa" font-size="9" font-family="monospace">${nid}:${sol.nodeV[nid].toFixed(1)}V</text>`;
    });
  }

  svg.innerHTML = html;

  // Stats & sliders
  const stats = document.getElementById('stats');
  const steps = document.getElementById('steps');
  const sliders = document.getElementById('sliders');
  if (!sol.ok) { stats.innerHTML=''; steps.innerHTML=''; sliders.innerHTML=''; return; }

  stats.innerHTML = `
    <div class="stat"><b>Total current</b><span>${sol.srcI.toFixed(3)} A</span></div>
    <div class="stat"><b>R equivalent</b><span>${sol.req!=null?sol.req.toFixed(2)+' Ω':'—'}</span></div>
    <div class="stat"><b>Power</b><span>${sol.branches.filter(b=>b.p>.001).map(b=>b.id+':'+b.p.toFixed(1)+'W').join(' ')||'—'}</span></div>`;
  steps.innerHTML = sol.steps.map(s=>`<li>${s}</li>`).join('');
  sliders.innerHTML = circuit.components.filter(c=>['voltage_source','resistor','bulb','switch'].includes(c.type)).map(c => {
    if (c.type==='switch') return `<div class="slider-row"><span>${c.id}</span><button onclick="toggleSw('${c.id}')">${c.closed!==false?'Open':'Close'}</button></div>`;
    const unit = c.type==='voltage_source'?'V':'Ω';
    const max = c.type==='voltage_source'?24:50;
    return `<div class="slider-row"><span>${c.id}: <b id="sv-${c.id}">${c.value}</b> ${unit}</span>
      <input type="range" min="${c.type==='voltage_source'?1:0.5}" max="${max}" step="0.5" value="${c.value}"
        oninput="setVal('${c.id}',+this.value)"/></div>`;
  }).join('');
}

function setVal(id, val) {
  circuit.components.forEach(c => { if (c.id===id) c.value=val; });
  document.getElementById('sv-'+id).textContent = val;
  codeEl.value = JSON.stringify(circuit, null, 2);
  render();
}

function toggleSw(id) {
  circuit.components.forEach(c => { if (c.id===id && c.type==='switch') c.closed = !c.closed; });
  codeEl.value = JSON.stringify(circuit, null, 2);
  render();
}

function applyCode() {
  const errEl = document.getElementById('codeErr');
  try {
    const data = JSON.parse(codeEl.value);
    const v = validateCircuit(data);
    if (!v.ok) { errEl.style.display='block'; errEl.textContent=v.errors.join('; '); return; }
    errEl.style.display='none';
    circuit = v.circuit;
    render();
  } catch(e) { errEl.style.display='block'; errEl.textContent='Invalid JSON'; }
}

function formatCode() {
  try { codeEl.value = JSON.stringify(JSON.parse(codeEl.value), null, 2); } catch(e){}
}

function loadEx(name) {
  circuit = JSON.parse(JSON.stringify(EXAMPLES[name]));
  codeEl.value = JSON.stringify(circuit, null, 2);
  render();
}

function parseSimpleText() {
  const t = document.getElementById('textIn').value.toLowerCase();
  const vMatch = t.match(/(\d+(?:\.\d+)?)\s*v/);
  const rMatches = [...t.matchAll(/(\d+(?:\.\d+)?)\s*(?:ω|ohm|Ω|ohms)?/g)];
  const V = vMatch ? parseFloat(vMatch[1]) : 12;
  const resistors = rMatches.map(m => parseFloat(m[1])).filter((r,i) => !vMatch || r !== V);
  if (resistors.length < 1) { alert('Could not find resistor values. Try examples or paste JSON from Poe.'); return; }

  if (t.includes('parallel')) {
    const comps = [{id:'V1',type:'voltage_source',value:V,unit:'V',nodes:['gnd','top']}];
    const layout = { V1:{x:100,y:280} };
    resistors.forEach((r,i) => {
      const id = 'R'+(i+1);
      comps.push({id,type:'resistor',value:r,unit:'ohm',nodes:['top','gnd']});
      layout[id] = { x:300, y: 140 + i*80 };
    });
    circuit = { name:'Parsed parallel', components:comps, layout };
  } else {
    const comps = [{id:'V1',type:'voltage_source',value:V,unit:'V',nodes:['n0','n1']}];
    const layout = { V1:{x:80,y:200} };
    let prev = 'n1';
    resistors.forEach((r,i) => {
      const id = 'R'+(i+1);
      const next = i === resistors.length-1 ? 'n0' : 'n'+(i+2);
      comps.push({id,type:'resistor',value:r,unit:'ohm',nodes:[prev,next]});
      layout[id] = { x: 80+(i+1)*140, y:200 };
      prev = next;
    });
    circuit = { name:'Parsed series', components:comps, layout };
  }
  codeEl.value = JSON.stringify(circuit, null, 2);
  render();
}

// Drag components
svg.addEventListener('pointerdown', e => {
  const g = e.target.closest('[data-id]');
  if (!g) return;
  dragging = g.getAttribute('data-id');
  const pt = svg.createSVGPoint();
  pt.x = e.clientX; pt.y = e.clientY;
  const p = pt.matrixTransform(svg.getScreenCTM().inverse());
  const pos = defaultLayout(circuit)[dragging];
  dragOff = { x: p.x - pos.x, y: p.y - pos.y };
  g.setPointerCapture(e.pointerId);
});
svg.addEventListener('pointermove', e => {
  if (!dragging) return;
  const pt = svg.createSVGPoint();
  pt.x = e.clientX; pt.y = e.clientY;
  const p = pt.matrixTransform(svg.getScreenCTM().inverse());
  if (!circuit.layout) circuit.layout = {};
  circuit.layout[dragging] = { x: p.x - dragOff.x, y: p.y - dragOff.y };
  codeEl.value = JSON.stringify(circuit, null, 2);
  render();
  // restore drag state after render rebuilds DOM
  dragging = dragging;
});
svg.addEventListener('pointerup', () => dragging = null);
svg.addEventListener('wheel', e => {
  e.preventDefault();
  zoom = Math.min(2.5, Math.max(0.5, zoom + (e.deltaY > 0 ? -0.1 : 0.1)));
  render();
}, { passive: false });

loadEx('series');
</script>
</body>
</html>
