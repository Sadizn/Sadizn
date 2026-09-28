#!/usr/bin/env node
/**
 * Gera assets/contributions.svg — quadro animado com o total de contribuições,
 * sequência atual, maior sequência e linguagens mais usadas.
 *
 * Uso:
 *   GH_TOKEN=xxx GH_LOGIN=Sadizn node scripts/contributions-card.mjs
 *   node scripts/contributions-card.mjs --demo      (dados de exemplo, sem rede)
 */
import { mkdir, writeFile } from 'node:fs/promises';
import { dirname } from 'node:path';

const LOGIN = process.env.GH_LOGIN || 'Sadizn';
const TOKEN = process.env.GH_TOKEN || process.env.GITHUB_TOKEN || '';
const OUT = process.env.OUT || 'assets/contributions.svg';
const DEMO = process.argv.includes('--demo');

/* ───────────── utilidades ───────────── */
const MONTHS = ['jan', 'fev', 'mar', 'abr', 'mai', 'jun', 'jul', 'ago', 'set', 'out', 'nov', 'dez'];
const ENT = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' };
const esc = (s) => String(s).replace(/[&<>"']/g, (c) => ENT[c]);
const iso = (d) => d.toISOString().slice(0, 10);
const addDays = (s, n) => {
  const d = new Date(`${s}T00:00:00Z`);
  d.setUTCDate(d.getUTCDate() + n);
  return iso(d);
};
const dots = (n) => String(n).replace(/\B(?=(\d{3})+(?!\d))/g, '.');
const dmy = (s, year = true) => {
  const [y, m, d] = s.split('-').map(Number);
  return `${d} ${MONTHS[m - 1]}${year ? ` ${y}` : ''}`;
};
const span = (a, b, nowYear) => {
  if (!a) return 'sem sequência ativa';
  const short = a.startsWith(String(nowYear)) && b.startsWith(String(nowYear));
  return a === b ? dmy(a, !short) : `${dmy(a, !short)} – ${dmy(b, !short)}`;
};
const r1 = (n) => Math.round(n * 10) / 10;

/* ───────────── dados (API do GitHub) ───────────── */
async function api(path, body) {
  const res = await fetch(`https://api.github.com${path}`, {
    method: body ? 'POST' : 'GET',
    headers: {
      Accept: 'application/vnd.github+json',
      'User-Agent': 'contributions-card',
      ...(TOKEN ? { Authorization: `Bearer ${TOKEN}` } : {}),
      ...(body ? { 'Content-Type': 'application/json' } : {}),
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  const json = await res.json();
  if (!res.ok || json.errors) throw new Error(`${path}: ${JSON.stringify(json.errors || json.message)}`);
  return json;
}
const gql = async (query, variables) => (await api('/graphql', { query, variables })).data;

const CAL = `query($l:String!,$f:DateTime!,$t:DateTime!){user(login:$l){contributionsCollection(from:$f,to:$t){contributionCalendar{weeks{contributionDays{date contributionCount}}}}}}`;

function analyse(days, since, today) {
  const keys = [...days.keys()].filter((k) => k <= today).sort();
  const val = (k) => days.get(k) ?? 0;

  let total = 0;
  let run = 0;
  let runStart = null;
  let longest = { count: 0, start: null, end: null };
  for (const k of keys) {
    total += val(k);
    if (val(k) > 0) {
      if (run === 0) runStart = k;
      run += 1;
      if (run > longest.count) longest = { count: run, start: runStart, end: k };
    } else {
      run = 0;
    }
  }

  // sequência atual: hoje ainda pode contribuir, por isso só quebra se ontem também for 0
  let i = keys.length - 1;
  if (i >= 0 && val(keys[i]) === 0) i -= 1;
  let current = { count: 0, start: null, end: null };
  if (i >= 0 && val(keys[i]) > 0) {
    const end = keys[i];
    let n = 0;
    while (i >= 0 && val(keys[i]) > 0) {
      n += 1;
      i -= 1;
    }
    current = { count: n, start: keys[i + 1], end };
  }

  // últimas 26 semanas (blocos de 7 dias a terminar hoje)
  const weekly = [];
  for (let w = 25; w >= 0; w -= 1) {
    let s = 0;
    for (let d = 0; d < 7; d += 1) s += val(addDays(today, -(w * 7 + d)));
    weekly.push(s);
  }
  let last7 = 0;
  for (let d = 0; d < 7; d += 1) last7 += val(addDays(today, -d));

  return { total, since, current, longest, weekly, last7 };
}

async function fetchLangs() {
  const repos = await api(`/users/${LOGIN}/repos?per_page=100&type=owner`);
  const totals = {};
  await Promise.all(
    repos
      .filter((r) => !r.fork)
      .map(async (r) => {
        const l = await api(`/repos/${r.full_name}/languages`);
        for (const [name, bytes] of Object.entries(l)) totals[name] = (totals[name] ?? 0) + bytes;
      }),
  );
  const sum = Object.values(totals).reduce((a, b) => a + b, 0);
  const sorted = Object.entries(totals).sort((a, b) => b[1] - a[1]);
  const top = sorted.slice(0, 5);
  const rest = sorted.slice(5).reduce((a, [, v]) => a + v, 0);
  if (rest > 0) top.push(['Outras', rest]);
  return { langs: sum ? top.map(([name, bytes]) => ({ name, pct: (bytes / sum) * 100 })) : [], repos: repos.length };
}

async function fetchData() {
  if (!TOKEN) throw new Error('Defina GH_TOKEN (ou GITHUB_TOKEN) para consultar a API do GitHub.');
  const { user } = await gql('query($l:String!){user(login:$l){createdAt}}', { l: LOGIN });
  const created = new Date(user.createdAt);
  const now = new Date();
  const days = new Map();
  for (let y = created.getUTCFullYear(); y <= now.getUTCFullYear(); y += 1) {
    const from = new Date(Math.max(created.getTime(), Date.UTC(y, 0, 1)));
    const to = new Date(Math.min(now.getTime(), Date.UTC(y, 11, 31, 23, 59, 59)));
    const data = await gql(CAL, { l: LOGIN, f: from.toISOString(), t: to.toISOString() });
    for (const w of data.user.contributionsCollection.contributionCalendar.weeks) {
      for (const day of w.contributionDays) {
        days.set(day.date, Math.max(days.get(day.date) ?? 0, day.contributionCount));
      }
    }
  }
  const langs = await fetchLangs().catch((e) => {
    console.warn('aviso: linguagens indisponíveis —', e.message);
    return { langs: [], repos: 0 };
  });
  return { ...analyse(days, iso(created), iso(now)), ...langs, generatedAt: iso(now) };
}

const DEMO_DATA = {
  total: 1284,
  since: '2024-04-08',
  current: { count: 12, start: '2026-09-17', end: '2026-09-28' },
  longest: { count: 27, start: '2026-02-03', end: '2026-03-01' },
  weekly: [3, 8, 5, 12, 9, 14, 6, 18, 11, 7, 15, 22, 9, 13, 20, 17, 8, 25, 19, 14, 28, 21, 16, 31, 24, 36],
  last7: 38,
  langs: [
    { name: 'JavaScript', pct: 46.2 },
    { name: 'PHP', pct: 21.5 },
    { name: 'HTML', pct: 14.8 },
    { name: 'CSS', pct: 9.1 },
    { name: 'Python', pct: 6.3 },
    { name: 'Outras', pct: 2.1 },
  ],
  repos: 5,
  generatedAt: '2026-09-28',
};

/* ───────────── SVG ───────────── */
const FONT = 'Segoe UI, system-ui, -apple-system, BlinkMacSystemFont, Helvetica Neue, Arial, sans-serif';
const LANG_COLORS = {
  JavaScript: '#f1e05a', TypeScript: '#3178c6', PHP: '#8892bf', HTML: '#e34c26', CSS: '#7c5cbf',
  Python: '#3776ab', Shell: '#89e051', Java: '#e0761a', 'C++': '#f34b7d', C: '#8b8b8b', 'C#': '#3fa02a',
  Go: '#00add8', Rust: '#dea584', Ruby: '#cc342d', Dart: '#00b4ab', Vue: '#41b883', SCSS: '#c6538c',
  Batchfile: '#c1f12e', EJS: '#c02a52', Blade: '#f7523f', Dockerfile: '#4b8fb1', Outras: '#6e7681',
};
const FALLBACK = ['#22d3ee', '#a78bfa', '#f472b6', '#fb923c', '#facc15', '#4ade80'];

const rr = ({ x, y, w, h }, r = 18) =>
  `M${x + r} ${y}H${x + w - r}A${r} ${r} 0 0 1 ${x + w} ${y + r}V${y + h - r}A${r} ${r} 0 0 1 ${x + w - r} ${y + h}H${x + r}A${r} ${r} 0 0 1 ${x} ${y + h - r}V${y + r}A${r} ${r} 0 0 1 ${x + r} ${y}Z`;
const ring = (cx, cy, r) => `M${cx} ${cy - r}A${r} ${r} 0 1 1 ${cx} ${cy + r}A${r} ${r} 0 1 1 ${cx} ${cy - r}`;

let seed = 7;
const rnd = () => (seed = (seed * 16807) % 2147483647) / 2147483647;

const STATIC_CSS = `
text{font-family:${FONT}}
.lab{font-size:11.5px;font-weight:700;letter-spacing:1.8px;fill:#6ee7b7}
.muted{font-size:12.5px;fill:#8b949e}
.in{animation:rise .8s cubic-bezier(.2,.8,.2,1) both}
.up{animation:up .7s cubic-bezier(.2,.8,.2,1) both}
.beam{animation:beam 6.5s linear infinite}
.spin{animation:spin 14s linear infinite}
.spinr{animation:spinr 9s linear infinite}
.pulse{animation:pulse 2.6s ease-in-out infinite}
.ping{opacity:0;animation:ping 2.8s ease-out infinite}
.glow{animation:glow 4.2s ease-in-out infinite}
.drift1{animation:drift1 16s ease-in-out infinite}
.drift2{animation:drift2 19s ease-in-out infinite}
.bar{animation:barin .9s cubic-bezier(.2,1.2,.3,1) both,wave 3.4s ease-in-out infinite;animation-delay:calc(.9s + var(--i)*.045s),calc(2.8s + var(--i)*.09s)}
.ptc{opacity:0;animation:float 7s linear infinite}
.sweep{animation:sweep 8s ease-in-out 3.2s infinite both}
.flick{animation:flick 1.7s ease-in-out infinite}
.bob{animation:bob 1.6s ease-in-out infinite}
@keyframes rise{from{opacity:0;transform:translateY(20px)}to{opacity:1;transform:translateY(0)}}
@keyframes up{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:translateY(0)}}
@keyframes fade{from{opacity:0}to{opacity:1}}
@keyframes beam{to{stroke-dashoffset:-100}}
@keyframes spin{to{transform:rotate(360deg)}}
@keyframes spinr{to{transform:rotate(-360deg)}}
@keyframes pulse{0%,100%{transform:scale(1);opacity:.85}50%{transform:scale(1.2);opacity:1}}
@keyframes ping{0%{transform:scale(1);opacity:.5}100%{transform:scale(1.5);opacity:0}}
@keyframes ping2{0%{transform:scale(1);opacity:.6}100%{transform:scale(2.8);opacity:0}}
@keyframes glow{0%,100%{opacity:.35;transform:scale(1)}50%{opacity:.7;transform:scale(1.08)}}
@keyframes drift1{0%,100%{transform:translate(0,0)}50%{transform:translate(70px,26px)}}
@keyframes drift2{0%,100%{transform:translate(0,0)}50%{transform:translate(-80px,-22px)}}
@keyframes barin{from{transform:scaleY(0)}to{transform:scaleY(1)}}
@keyframes wave{0%,100%{opacity:.5}50%{opacity:1}}
@keyframes float{0%{transform:translateY(0);opacity:0}15%{opacity:.9}100%{transform:translateY(-118px);opacity:0}}
@keyframes sweep{0%{transform:translateX(0)}24%,100%{transform:translateX(720px)}}
@keyframes flick{0%,100%{transform:scale(1) rotate(0)}30%{transform:scale(1.1,.92) rotate(-4deg)}60%{transform:scale(.94,1.1) rotate(3deg)}}
@keyframes bob{0%,100%{transform:translateY(0)}50%{transform:translateY(-2.5px)}}
@media (prefers-reduced-motion:reduce){*{animation:none!important}}`;

const flame = (x, y) =>
  `<g class="flick" style="transform-origin:${x}px ${y + 4}px"><g transform="translate(${x} ${y})"><path d="M0-9C0-9-7-3-7 2.5A7 7 0 0 0 7 2.5C7-1 4.5-3 3.5-5C3-3 2-2 1-2C1-5 0-9 0-9Z" fill="#34d399"/><path d="M0 9A3.6 3.6 0 0 1-3.6 5.4C-3.6 3.2-1 2.4-.5 0C1 1.2 3.6 3.2 3.6 5.4A3.6 3.6 0 0 1 0 9Z" fill="#d1fae5"/></g></g>`;
const trophy = (x, y) =>
  `<g class="flick" style="transform-origin:${x}px ${y + 4}px"><g transform="translate(${x} ${y})" fill="none" stroke="#34d399" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M-5-8h10v5a5 5 0 0 1-10 0z"/><path d="M-5-6h-3v2a3.2 3.2 0 0 0 3.2 3.2M5-6h3v2a3.2 3.2 0 0 1-3.2 3.2"/><path d="M0 2v4M-4 8.5h8"/></g></g>`;

function render(d) {
  seed = 7;
  const hasLangs = d.langs.length > 0;
  const W = 900;
  const PAD = 24;
  const GAP = 16;
  const R1 = 256;
  const R2 = 108;
  const H = hasLangs ? PAD + R1 + GAP + R2 + PAD : PAD + R1 + PAD;
  const hero = { x: PAD, y: PAD, w: 400, h: R1 };
  const b2 = { x: hero.x + hero.w + GAP, y: PAD, w: 210, h: R1 };
  const b3 = { x: b2.x + b2.w + GAP, y: PAD, w: 210, h: R1 };
  const row2 = { x: PAD, y: PAD + R1 + GAP, w: W - PAD * 2, h: R2 };
  const nowYear = Number(d.generatedAt.slice(0, 4));

  const defs = [];
  const kf = [];
  const strips = new Map();
  let uid = 0;

  /* contador estilo "odómetro": cada dígito rola até ao valor final */
  function odometer(text, { cx = 0, left = 0, base, fs, dw, sw, delay, anchor = 'start' }) {
    const cols = [...text].map((ch) => (/\d/.test(ch) ? { ch, digit: true, w: dw } : { ch, digit: false, w: sw }));
    const total = cols.reduce((a, c) => a + c.w, 0);
    let x = anchor === 'middle' ? cx - total / 2 : left;
    const x0 = x;
    const lineH = Math.round(fs * 1.3);
    strips.set(fs, lineH);
    const nd = cols.filter((c) => c.digit).length;
    const id = `oc${++uid}`;
    defs.push(
      `<clipPath id="${id}"><rect x="${r1(x0 - 3)}" y="${r1(base - fs * 0.82)}" width="${r1(total + 6)}" height="${r1(fs * 0.98)}"/></clipPath>`,
    );
    let k = 0;
    const out = [];
    for (const c of cols) {
      const mid = r1(x + c.w / 2);
      if (c.digit) {
        const loops = 2 + ((nd - 1 - k) % 3);
        const dist = (10 * loops + Number(c.ch)) * lineH;
        const name = `o${++uid}`;
        const dur = (1.9 + 0.2 * k).toFixed(2);
        kf.push(`@keyframes ${name}{from{transform:translateY(0)}to{transform:translateY(-${dist}px)}}`);
        out.push(
          `<g transform="translate(0 -${dist})" style="animation:${name} ${dur}s cubic-bezier(.16,1,.3,1) ${delay}s both"><use href="#st${fs}" x="${mid}" y="${base}"/></g>`,
        );
        k += 1;
      } else {
        out.push(
          `<text x="${mid}" y="${base}" text-anchor="middle" font-size="${fs}" font-weight="800" fill="#6ee7b7" style="animation:fade .6s ease ${r1(delay + 1.2)}s both">${esc(c.ch)}</text>`,
        );
      }
      x += c.w;
    }
    return { svg: `<g clip-path="url(#${id})">${out.join('')}</g>`, total, left: x0 };
  }

  const box = (b, phase) => {
    const p = rr(b);
    return `<path d="${p}" fill="#fff" fill-opacity=".035"/>
<path d="${p}" fill="none" stroke="#fff" stroke-opacity=".09"/>
<path class="beam" d="${p}" pathLength="100" fill="none" stroke="url(#gBeam)" stroke-width="6" stroke-linecap="round" stroke-dasharray="14 86" filter="url(#fBlur)" opacity=".45" style="animation-delay:-${phase}s"/>
<path class="beam" d="${p}" pathLength="100" fill="none" stroke="url(#gBeam)" stroke-width="1.8" stroke-linecap="round" stroke-dasharray="14 86" style="animation-delay:-${phase}s"/>`;
  };
  const shimmer = (b, id, delay, grad = 'gShim') =>
    `<g clip-path="url(#${id})"><path class="sweep" d="M${b.x - 130} ${b.y}h64l-40 ${b.h}h-64z" fill="url(#${grad})" style="animation-delay:${delay}s"/></g>`;

  defs.push(`<clipPath id="cAll"><rect width="${W}" height="${H}" rx="22"/></clipPath>`);
  defs.push(`<clipPath id="cHero"><path d="${rr(hero)}"/></clipPath>`);
  defs.push(`<clipPath id="cB2"><path d="${rr(b2)}"/></clipPath>`);
  defs.push(`<clipPath id="cB3"><path d="${rr(b3)}"/></clipPath>`);
  defs.push(`<clipPath id="cRow"><path d="${rr(row2)}"/></clipPath>`);

  /* ── HERO: total de contribuições ── */
  const hx = hero.x;
  const hy = hero.y;
  const numStr = dots(d.total);
  const nd = numStr.replace(/\D/g, '').length;
  const ns = numStr.length - nd;
  const sw = 16;
  const dw = Math.min(46, Math.floor((252 - ns * sw) / nd));
  const fs = Math.round(dw * 1.55);
  const base = hy + 122;
  const num = odometer(numStr, { left: hx + 28, base, fs, dw, sw, delay: 0.55 });
  const gcx = r1(hx + 28 + num.total / 2);

  const ox = hx + hero.w - 68;
  const oy = hy + 104;
  const orbit = `<circle class="pulse" cx="${ox}" cy="${oy}" r="26" fill="url(#gGlow)" style="transform-origin:${ox}px ${oy}px"/>
<circle class="spin" cx="${ox}" cy="${oy}" r="40" fill="none" stroke="#34d399" stroke-opacity=".55" stroke-width="1.4" stroke-dasharray="2 6.98" stroke-linecap="round" style="transform-origin:${ox}px ${oy}px;animation-duration:22s"/>
<circle class="spinr" cx="${ox}" cy="${oy}" r="30" fill="none" stroke="url(#gRing)" stroke-width="2.4" stroke-linecap="round" stroke-dasharray="48 141" style="transform-origin:${ox}px ${oy}px"/>
<g class="spin" style="transform-origin:${ox}px ${oy}px;animation-duration:7s"><circle cx="${ox}" cy="${oy - 40}" r="3.4" fill="#a7f3d0"/></g>
<circle cx="${ox}" cy="${oy}" r="15" fill="#0d1117" stroke="#34d399" stroke-opacity=".6"/>
<path d="M${ox - 12} ${oy}h7M${ox + 5} ${oy}h7" stroke="#6ee7b7" stroke-width="2" stroke-linecap="round"/>
<circle cx="${ox}" cy="${oy}" r="4.6" fill="none" stroke="#6ee7b7" stroke-width="2"/>`;

  const chipText = `${d.last7 > 0 ? '+' : ''}${dots(d.last7)} nos últimos 7 dias`;
  const chipW = Math.round(46 + chipText.length * 6.3);
  const cy0 = hy + 166;
  const chip = `<g class="up" style="animation-delay:1.9s"><rect x="${hx + 28}" y="${cy0}" width="${chipW}" height="24" rx="12" fill="#10b981" fill-opacity=".14" stroke="#34d399" stroke-opacity=".35"/><path class="bob" d="M${hx + 42} ${cy0 + 15}l4-5 4 5" fill="none" stroke="#6ee7b7" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/><text x="${hx + 58}" y="${cy0 + 16}" font-size="12" font-weight="600" fill="#a7f3d0">${esc(chipText)}</text></g>`;

  const N = d.weekly.length;
  const bx0 = hx + 28;
  const bxw = hero.w - 56;
  const bw = 8;
  const bbase = hy + 232;
  const bmax = 32;
  const wmax = Math.max(...d.weekly, 1);
  const step = (bxw - bw) / (N - 1);
  const bars = d.weekly
    .map((v, i) => {
      const h = r1(4 + (v / wmax) * (bmax - 4));
      const x = r1(bx0 + i * step);
      const fill = i === N - 1 ? '#a7f3d0' : 'url(#gBar)';
      return `<rect class="bar" x="${x}" y="${r1(bbase - h)}" width="${bw}" height="${h}" rx="2.5" fill="${fill}" style="--i:${i};transform-origin:${r1(x + bw / 2)}px ${bbase}px"/>`;
    })
    .join('');
  const particles = Array.from({ length: 9 }, () => {
    const x = r1(hx + 150 + rnd() * (hero.w - 190));
    const r = r1(1.3 + rnd() * 1.4);
    const dur = r1(5 + rnd() * 4);
    const del = r1(rnd() * 6);
    return `<circle class="ptc" cx="${x}" cy="${bbase}" r="${r}" fill="#6ee7b7" style="animation-duration:${dur}s;animation-delay:${del}s"/>`;
  }).join('');

  const heroSvg = `<g class="in" style="animation-delay:.05s">
${box(hero, 0)}
<ellipse class="glow" cx="${gcx}" cy="${hy + 98}" rx="${r1(num.total / 2 + 46)}" ry="46" fill="url(#gGlow)" opacity=".4" style="transform-origin:${gcx}px ${hy + 98}px"/>
${particles}
<path d="M${bx0} ${bbase + 2}H${bx0 + bxw}" stroke="#fff" stroke-opacity=".08"/>
${bars}
<circle class="pulse" cx="${hx + 30}" cy="${hy + 31}" r="4" fill="#34d399" style="transform-origin:${hx + 30}px ${hy + 31}px"/>
<circle class="ping" cx="${hx + 30}" cy="${hy + 31}" r="4" fill="none" stroke="#34d399" stroke-width="1.4" style="transform-origin:${hx + 30}px ${hy + 31}px;animation-name:ping2"/>
<text class="lab" x="${hx + 44}" y="${hy + 35}">TOTAL DE CONTRIBUIÇÕES</text>
${num.svg}
<text class="muted" x="${hx + 30}" y="${hy + 152}">${dmy(d.since)} — hoje</text>
${chip}
${orbit}
${shimmer(hero, 'cHero', 3.2)}
</g>`;

  /* ── caixas de sequência ── */
  function streakBox(b, o) {
    const cx = b.x + b.w / 2;
    const cy = b.y + 132;
    const R = 54;
    const str = String(o.value);
    const dwS = Math.min(25, Math.floor(84 / str.length));
    const odo = odometer(str, { cx, base: cy + 14, fs: 40, dw: dwS, sw: 10, delay: o.delay + 0.15, anchor: 'middle' });
    const p = r1(Math.max(0, Math.min(1, o.ratio)) * 100);
    const name = `dr${++uid}`;
    kf.push(`@keyframes ${name}{from{stroke-dasharray:0 100}to{stroke-dasharray:${p} 100}}`);
    const prog =
      p > 0
        ? `<path d="${ring(cx, cy, R)}" pathLength="100" fill="none" stroke="url(#gRing)" stroke-width="9" stroke-linecap="round" stroke-dasharray="${p} 100" style="animation:${name} 1.9s cubic-bezier(.3,.7,.2,1) ${r1(o.delay + 0.1)}s both"/>`
        : '';
    return `<g class="in" style="animation-delay:${o.delay}s">
${box(b, o.phase)}
${o.icon(b.x + 24, b.y + 30)}
<text class="lab" x="${b.x + 40}" y="${b.y + 35}">${o.label}</text>
<circle class="ping" cx="${cx}" cy="${cy}" r="${R + 6}" fill="none" stroke="#34d399" stroke-width="1.2" style="transform-origin:${cx}px ${cy}px;animation-delay:${o.phase % 3}s"/>
<path d="${ring(cx, cy, R)}" fill="none" stroke="#fff" stroke-opacity=".08" stroke-width="9"/>
${prog}
<g class="spin" style="transform-origin:${cx}px ${cy}px;animation-duration:6s"><circle cx="${cx}" cy="${cy - R}" r="3.4" fill="#fff"/><circle cx="${cx}" cy="${cy - R}" r="8" fill="#34d399" opacity=".35" filter="url(#fBlur)"/></g>
${odo.svg}
<text class="muted" x="${cx}" y="${cy + 38}" text-anchor="middle">${o.value === 1 ? 'dia' : 'dias'}</text>
<text class="muted" x="${cx}" y="${b.y + b.h - 24}" text-anchor="middle">${esc(o.range)}</text>
${shimmer(b, o.clip, o.delay + 3.6)}
</g>`;
  }
  const cur = d.current;
  const lon = d.longest;
  const s2 = streakBox(b2, {
    label: 'SEQUÊNCIA ATUAL', value: cur.count, ratio: lon.count ? cur.count / lon.count : 0,
    range: span(cur.start, cur.end, nowYear), icon: flame, delay: 0.2, phase: 2, clip: 'cB2',
  });
  const s3 = streakBox(b3, {
    label: 'MAIOR SEQUÊNCIA', value: lon.count, ratio: 1,
    range: span(lon.start, lon.end, nowYear), icon: trophy, delay: 0.35, phase: 4, clip: 'cB3',
  });

  /* ── linguagens ── */
  let langSvg = '';
  if (hasLangs) {
    const x0 = row2.x + 28;
    const wBar = row2.w - 56;
    const yBar = row2.y + 52;
    const hBar = 14;
    defs.push(`<clipPath id="cBar"><rect x="${x0}" y="${yBar}" width="${wBar}" height="${hBar}" rx="7"/></clipPath>`);
    const wipe = `wp${++uid}`;
    kf.push(`@keyframes ${wipe}{from{transform:translateX(-${wBar + 30}px)}to{transform:translateX(0)}}`);
    let acc = 0;
    const segs = d.langs.map((l, i) => {
      const w = (l.pct / 100) * wBar;
      const s = { ...l, x: acc, w, color: LANG_COLORS[l.name] ?? FALLBACK[i % FALLBACK.length] };
      acc += w;
      return s;
    });
    const rects = segs
      .map((s) => `<rect x="${r1(x0 + s.x)}" y="${yBar}" width="${r1(s.w)}" height="${hBar}" fill="${s.color}" stroke="#0d1117" stroke-width="2"/>`)
      .join('');
    const colW = wBar / segs.length;
    const legend = segs
      .map((s, i) => {
        const lx = r1(x0 + i * colW);
        const nm = s.name.length > 11 ? `${s.name.slice(0, 10)}…` : s.name;
        return `<g class="up" style="animation-delay:${r1(1.6 + i * 0.1)}s"><circle cx="${r1(lx + 5)}" cy="${yBar + 36}" r="5" fill="${s.color}"/><text x="${r1(lx + 16)}" y="${yBar + 40}" font-size="12.5"><tspan font-weight="600" fill="#e6edf3">${esc(nm)}</tspan><tspan dx="6" fill="#8b949e">${s.pct.toFixed(1).replace('.', ',')}%</tspan></text></g>`;
      })
      .join('');
    langSvg = `<g class="in" style="animation-delay:.5s">
${box(row2, 6)}
<text class="lab" x="${x0}" y="${row2.y + 36}">LINGUAGENS MAIS USADAS</text>
<text class="muted" x="${row2.x + row2.w - 28}" y="${row2.y + 36}" text-anchor="end">${d.repos} ${d.repos === 1 ? 'repositório público' : 'repositórios públicos'}</text>
<g clip-path="url(#cBar)"><rect x="${x0}" y="${yBar}" width="${wBar}" height="${hBar}" fill="#fff" fill-opacity=".07"/><g style="animation:${wipe} 1.5s cubic-bezier(.2,.8,.2,1) 1.1s both">${rects}</g><path class="sweep" d="M${x0 - 60} ${yBar}h34l-8 ${hBar}h-34z" fill="url(#gGloss)" style="animation-delay:4s"/></g>
${legend}
${shimmer(row2, 'cRow', 5)}
</g>`;
  }

  /* ── montagem ── */
  const strip = [...strips]
    .map(([f, lh]) => {
      const lines = Array.from({ length: 50 }, (_, i) => `<tspan x="0"${i ? ` dy="${lh}"` : ''}>${i % 10}</tspan>`).join('');
      return `<g id="st${f}" font-size="${f}" font-weight="800" fill="#f0fdf4" text-anchor="middle"><text x="0" y="0">${lines}</text></g>`;
    })
    .join('');

  return `<svg xmlns="http://www.w3.org/2000/svg" width="${W}" height="${H}" viewBox="0 0 ${W} ${H}" role="img" aria-labelledby="t d" font-family="${FONT}">
<title id="t">Contribuições no GitHub de ${esc(LOGIN)}</title>
<desc id="d">${dots(d.total)} contribuições no total, sequência atual de ${cur.count} dias e maior sequência de ${lon.count} dias.</desc>
<style>${STATIC_CSS}
${kf.join('\n')}</style>
<defs>
<linearGradient id="gBeam" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#34d399"/><stop offset="1" stop-color="#22d3ee"/></linearGradient>
<linearGradient id="gRing" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#34d399"/><stop offset="1" stop-color="#22d3ee"/></linearGradient>
<linearGradient id="gBar" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="#6ee7b7"/><stop offset="1" stop-color="#059669"/></linearGradient>
<linearGradient id="gShim" x1="0" y1="0" x2="1" y2="0"><stop offset="0" stop-color="#fff" stop-opacity="0"/><stop offset=".5" stop-color="#fff" stop-opacity=".1"/><stop offset="1" stop-color="#fff" stop-opacity="0"/></linearGradient>
<linearGradient id="gGloss" x1="0" y1="0" x2="1" y2="0"><stop offset="0" stop-color="#fff" stop-opacity="0"/><stop offset=".5" stop-color="#fff" stop-opacity=".4"/><stop offset="1" stop-color="#fff" stop-opacity="0"/></linearGradient>
<radialGradient id="gAu1"><stop offset="0" stop-color="#10b981" stop-opacity=".3"/><stop offset="1" stop-color="#10b981" stop-opacity="0"/></radialGradient>
<radialGradient id="gAu2"><stop offset="0" stop-color="#22d3ee" stop-opacity=".18"/><stop offset="1" stop-color="#22d3ee" stop-opacity="0"/></radialGradient>
<radialGradient id="gGlow"><stop offset="0" stop-color="#34d399" stop-opacity=".6"/><stop offset="1" stop-color="#34d399" stop-opacity="0"/></radialGradient>
<pattern id="dots" width="18" height="18" patternUnits="userSpaceOnUse"><circle cx="1.5" cy="1.5" r=".9" fill="#fff" fill-opacity=".06"/></pattern>
<filter id="fBlur" x="-20%" y="-20%" width="140%" height="140%"><feGaussianBlur stdDeviation="3.2"/></filter>
${defs.join('\n')}
${strip}
</defs>
<rect width="${W}" height="${H}" rx="22" fill="#0d1117"/>
<rect width="${W}" height="${H}" rx="22" fill="url(#dots)"/>
<g clip-path="url(#cAll)"><ellipse class="drift1" cx="210" cy="130" rx="300" ry="170" fill="url(#gAu1)"/><ellipse class="drift2" cx="700" cy="${H - 100}" rx="330" ry="180" fill="url(#gAu2)"/></g>
<rect x=".5" y=".5" width="${W - 1}" height="${H - 1}" rx="21.5" fill="none" stroke="#fff" stroke-opacity=".09"/>
${heroSvg}
${s2}
${s3}
${langSvg}
<text x="${W - PAD}" y="${H - 9}" text-anchor="end" font-size="10" fill="#6e7681">Atualizado em ${dmy(d.generatedAt)}</text>
</svg>
`;
}

async function main() {
  const data = DEMO ? DEMO_DATA : await fetchData();
  const svg = render(data);
  await mkdir(dirname(OUT), { recursive: true });
  await writeFile(OUT, svg, 'utf8');
  console.log(`✔ ${OUT} (${(svg.length / 1024).toFixed(1)} KB) — ${data.total} contribuições`);
}

main().catch((e) => {
  console.error('✖', e.message);
  process.exit(1);
});
