# ailogistics.arc03c
import React, { useState, useEffect, useMemo, useRef } from "react";
import {
  BarChart, Bar, LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip,
  ResponsiveContainer, RadarChart, PolarGrid, PolarAngleAxis, Radar, Legend,
} from "recharts";
import {
  Truck, Route, Map as MapIcon, Activity, AlertTriangle, Accessibility,
  BarChart3, CloudRain, Mountain, Wifi, Clock, TrendingDown, TrendingUp,
  Navigation, MapPin, Package, Shield, Zap, Leaf, ChevronRight, X, Info,
  CheckCircle2, Layers, Radio, Gauge, IndianRupee, Sparkles, ArrowRight,
  Menu, Compass, HeartPulse, Waves,
} from "lucide-react";

/* ============================================================
   TOKENS
============================================================ */
const C = {
  ink: "#081310",
  panel: "#0E1E19",
  panel2: "#132B23",
  line: "#1F3A30",
  lineSoft: "#17281F",
  teal: "#2ED3A0",
  tealDim: "#1B7A5C",
  amber: "#E3A542",
  clay: "#D97A52",
  river: "#4FA9D6",
  danger: "#E5595B",
  textHi: "#EEF6F1",
  textMid: "#93AFA2",
  textDim: "#5C7568",
};

const FONTS = `
@import url('https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,400..800&family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500;600&display=swap');
.f-display{font-family:'Bricolage Grotesque',sans-serif;}
.f-body{font-family:'IBM Plex Sans',sans-serif;}
.f-mono{font-family:'IBM Plex Mono',monospace;}
@keyframes dashmove{ to { stroke-dashoffset: -200; } }
@keyframes pulseRing{ 0%{transform:scale(0.6);opacity:.9} 100%{transform:scale(2.6);opacity:0} }
@keyframes floaty{ 0%,100%{transform:translateY(0)} 50%{transform:translateY(-4px)} }
@keyframes marquee{ 0%{transform:translateX(0)} 100%{transform:translateX(-50%)} }
.route-flow{ stroke-dasharray: 6 10; animation: dashmove 3.2s linear infinite; }
.pulse-node::before{ content:''; position:absolute; inset:0; border-radius:999px; border:2px solid currentColor; animation: pulseRing 2.4s ease-out infinite; }
.scrollbar-thin::-webkit-scrollbar{ width:6px; height:6px; }
.scrollbar-thin::-webkit-scrollbar-thumb{ background:#1F3A30; border-radius:99px; }
.scrollbar-thin::-webkit-scrollbar-track{ background:transparent; }
input[type=range]{ accent-color:#2ED3A0; }
`;

/* ============================================================
   DATA
============================================================ */
const CITIES = [
  { id: "gau", name: "Guwahati", state: "Assam", x: 350, y: 330, terrain: "valley", hub: true },
  { id: "dib", name: "Dibrugarh", state: "Assam", x: 700, y: 260, terrain: "valley", hub: true },
  { id: "jor", name: "Jorhat", state: "Assam", x: 600, y: 285, terrain: "valley", hub: false },
  { id: "tez", name: "Tezpur", state: "Assam", x: 440, y: 245, terrain: "riverine", hub: false },
  { id: "sil", name: "Silchar", state: "Assam", x: 430, y: 540, terrain: "riverine", hub: true },
  { id: "shi", name: "Shillong", state: "Meghalaya", x: 335, y: 430, terrain: "hilly", hub: true },
  { id: "aga", name: "Agartala", state: "Tripura", x: 320, y: 610, terrain: "valley", hub: true },
  { id: "aiz", name: "Aizawl", state: "Mizoram", x: 500, y: 630, terrain: "mountainous", hub: true },
  { id: "imp", name: "Imphal", state: "Manipur", x: 690, y: 530, terrain: "hilly", hub: true },
  { id: "koh", name: "Kohima", state: "Nagaland", x: 705, y: 405, terrain: "mountainous", hub: true },
  { id: "dim", name: "Dimapur", state: "Nagaland", x: 640, y: 370, terrain: "valley", hub: false },
  { id: "gan", name: "Gangtok", state: "Sikkim", x: 120, y: 150, terrain: "mountainous", hub: true },
  { id: "ita", name: "Itanagar", state: "Arunachal Pradesh", x: 610, y: 90, terrain: "hilly", hub: true },
  { id: "taw", name: "Tawang", state: "Arunachal Pradesh", x: 400, y: 45, terrain: "mountainous", hub: false },
  { id: "alo", name: "Along", state: "Arunachal Pradesh", x: 750, y: 150, terrain: "hilly", hub: false },
  { id: "zir", name: "Ziro", state: "Arunachal Pradesh", x: 555, y: 120, terrain: "hilly", hub: false },
];
const CITY = Object.fromEntries(CITIES.map((c) => [c.id, c]));

const STATES = [
  "Assam", "Meghalaya", "Tripura", "Mizoram", "Manipur", "Nagaland", "Sikkim", "Arunachal Pradesh",
];

const TERRAIN_SPEED = { valley: 50, riverine: 38, hilly: 32, mountainous: 21 };
const TERRAIN_RISK = { valley: 1, riverine: 1.4, hilly: 2, mountainous: 3 };

const MODE_COST = { Road: 8, Rail: 5, Air: 46, Waterway: 6 };
const MODE_EMISSION = { Road: 0.12, Rail: 0.04, Air: 0.51, Waterway: 0.07 };
const CARGO_FACTOR = { General: 1, Perishable: 1.3, Medical: 1.55, Fragile: 1.2 };

// deterministic small hash so results are stable per query (feels like a real model, not RNG)
function seedNum(str, mod = 10) {
  let h = 0;
  for (let i = 0; i < str.length; i++) h = (h * 31 + str.charCodeAt(i)) >>> 0;
  return h % mod;
}

const IS_MONSOON = true; // current date (Aug) falls in NER monsoon window — drives risk model

function distanceKm(a, b) {
  const dx = a.x - b.x, dy = a.y - b.y;
  return Math.sqrt(dx * dx + dy * dy) * 0.85;
}

function computeRouteOptions(originId, destId, mode, cargo) {
  const a = CITY[originId], b = CITY[destId];
  if (!a || !b || a.id === b.id) return null;
  const baseDist = distanceKm(a, b);
  const terrainAvg = (TERRAIN_RISK[a.terrain] + TERRAIN_RISK[b.terrain]) / 2;
  const speedAvg = (TERRAIN_SPEED[a.terrain] + TERRAIN_SPEED[b.terrain]) / 2;
  const monsoonBase = IS_MONSOON ? 18 : 6;
  const jitter = seedNum(a.id + b.id, 7); // 0-6 deterministic variance

  const variants = [
    {
      key: "fast",
      label: "Fastest NH Corridor",
      icon: "Zap",
      distMult: 1.0,
      speedMult: 1.18,
      weatherExposure: 1.0,
      costMult: 1.15,
      blurb: "Shortest highway path through primary NH corridor.",
    },
    {
      key: "balanced",
      label: "AI-Balanced Route",
      icon: "Sparkles",
      distMult: 1.08,
      speedMult: 1.0,
      weatherExposure: 0.75,
      costMult: 1.0,
      blurb: "Blends speed, cost and terrain risk using live conditions.",
    },
    {
      key: "safe",
      label: "Resilient Alternate",
      icon: "Shield",
      distMult: 1.24,
      speedMult: 0.82,
      weatherExposure: 0.35,
      costMult: 1.12,
      blurb: "Detours around known landslide/flood-prone stretches.",
    },
  ];

  const options = variants.map((v) => {
    const dist = baseDist * v.distMult;
    const speed = speedAvg * v.speedMult;
    const etaHrs = dist / speed;
    const risk = Math.min(
      98,
      Math.round(terrainAvg * 9 + monsoonBase * v.weatherExposure + jitter * (v.key === "fast" ? 1.4 : v.key === "balanced" ? 0.8 : 0.3))
    );
    const cost = Math.round(dist * MODE_COST[mode] * v.costMult * CARGO_FACTOR[cargo]);
    const carbon = Math.round(dist * MODE_EMISSION[mode] * v.costMult);
    return { ...v, distance: Math.round(dist), etaHrs, risk, cost, carbon };
  });

  return { origin: a, dest: b, baseDist, terrainAvg, options };
}

function pickRecommended(options, priority) {
  const key = { speed: "etaHrs", cost: "cost", safety: "risk", eco: "carbon" }[priority];
  return options.reduce((best, o) => (o[key] < best[key] ? o : best), options[0]);
}

/* accessibility model per state */
const ACCESS_DATA = {
  "Assam": { road: 78, digital: 74, health: 70, lastmile: 58, resilience: 60 },
  "Meghalaya": { road: 55, digital: 52, health: 54, lastmile: 42, resilience: 45 },
  "Tripura": { road: 68, digital: 63, health: 60, lastmile: 50, resilience: 55 },
  "Mizoram": { road: 46, digital: 55, health: 48, lastmile: 38, resilience: 40 },
  "Manipur": { road: 50, digital: 50, health: 47, lastmile: 36, resilience: 38 },
  "Nagaland": { road: 48, digital: 49, health: 45, lastmile: 35, resilience: 39 },
  "Sikkim": { road: 60, digital: 58, health: 62, lastmile: 52, resilience: 44 },
  "Arunachal Pradesh": { road: 38, digital: 41, health: 36, lastmile: 27, resilience: 33 },
};
const ACCESS_LABELS = {
  road: "Road Connectivity", digital: "Digital Coverage", health: "Healthcare Reach",
  lastmile: "PwD / Elderly Access", resilience: "Climate Resilience",
};
function compositeScore(s) {
  const w = ACCESS_DATA[s];
  return Math.round((w.road + w.digital + w.health + w.lastmile * 1.2 + w.resilience) / 5.2);
}
function lowestFactor(s) {
  const w = ACCESS_DATA[s];
  return Object.entries(w).sort((a, b) => a[1] - b[1])[0][0];
}
const RECO_TEXT = {
  road: (s) => `Prioritise all-weather road upgrades on feeder routes into ${s} to cut monsoon downtime.`,
  digital: (s) => `Deploy additional mobile towers / satellite backhaul across ${s} to close last-mile signal gaps.`,
  health: (s) => `Add mobile health units on AI-flagged low-access corridors within ${s}.`,
  lastmile: (s) => `Introduce ramp-fitted transit stops and assisted last-mile shuttles in ${s}'s district hubs.`,
  resilience: (s) => `Pre-position disaster relief stock in ${s} ahead of the monsoon risk window.`,
};

/* alerts pool */
const ALERT_POOL = [
  { type: "Landslide", sev: "High", region: "Mizoram", route: "NH-306 · Aizawl–Lunglei", icon: "Mountain" },
  { type: "Flood Watch", sev: "High", region: "Assam", route: "Brahmaputra belt · Dibrugarh", icon: "Waves" },
  { type: "Fog Advisory", sev: "Medium", region: "Assam", route: "Guwahati approach roads", icon: "CloudRain" },
  { type: "Road Block", sev: "Medium", region: "Nagaland", route: "NH-2 · Kohima–Dimapur", icon: "AlertTriangle" },
  { type: "Festival Congestion", sev: "Low", region: "Manipur", route: "Imphal city core", icon: "Activity" },
  { type: "Bridge Maintenance", sev: "Medium", region: "Arunachal Pradesh", route: "Along–Pasighat link", icon: "AlertTriangle" },
  { type: "Heavy Rainfall", sev: "High", region: "Meghalaya", route: "Shillong–Dawki corridor", icon: "CloudRain" },
  { type: "Network Outage", sev: "Low", region: "Sikkim", route: "Gangtok telecom sector", icon: "Wifi" },
  { type: "Landslide Watch", sev: "Medium", region: "Manipur", route: "NH-2 · Imphal–Kohima", icon: "Mountain" },
  { type: "Waterway Disruption", sev: "Low", region: "Assam", route: "Brahmaputra ferry route", icon: "Waves" },
];
function makeAlert(i) {
  const p = ALERT_POOL[i % ALERT_POOL.length];
  return { ...p, id: `${Date.now()}-${i}-${Math.random().toString(36).slice(2, 6)}`, time: new Date() };
}
const SEV_COLOR = { High: C.danger, Medium: C.amber, Low: C.teal };

const TREND = [
  { day: "Mon", before: 14.2, after: 9.8, shipments: 132 },
  { day: "Tue", before: 15.1, after: 10.1, shipments: 148 },
  { day: "Wed", before: 13.8, after: 9.4, shipments: 121 },
  { day: "Thu", before: 16.4, after: 10.6, shipments: 159 },
  { day: "Fri", before: 15.0, after: 9.9, shipments: 167 },
  { day: "Sat", before: 12.6, after: 8.7, shipments: 103 },
  { day: "Sun", before: 11.9, after: 8.2, shipments: 88 },
];

const STATE_VOLUME = STATES.map((s) => ({
  state: s.length > 10 ? s.slice(0, 9) + "." : s,
  full: s,
  shipments: Math.round(60 + seedNum(s, 90) * 4 + s.length * 8),
}));

/* ============================================================
   SMALL UI PARTS
============================================================ */
function TopoBackdrop() {
  const rings = new Array(10).fill(0);
  return (
    <svg className="absolute inset-0 w-full h-full opacity-[0.12]" preserveAspectRatio="none">
      {rings.map((_, i) => (
        <ellipse
          key={i}
          cx="18%" cy="10%"
          rx={80 + i * 70} ry={50 + i * 46}
          fill="none" stroke={C.teal} strokeWidth="1"
        />
      ))}
    </svg>
  );
}

function Badge({ children, color = C.teal, soft }) {
  return (
    <span
      className="f-mono inline-flex items-center gap-1 rounded-full px-2.5 py-0.5 text-[11px] tracking-wide"
      style={{
        color: soft ? color : C.ink,
        background: soft ? `${color}1c` : color,
        border: `1px solid ${color}55`,
      }}
    >
      {children}
    </span>
  );
}

function KPICard({ icon: Icon, label, value, sub, trend, color }) {
  return (
    <div
      className="rounded-2xl p-4 flex flex-col gap-3"
      style={{ background: C.panel, border: `1px solid ${C.line}` }}
    >
      <div className="flex items-center justify-between">
        <div
          className="w-9 h-9 rounded-xl flex items-center justify-center"
          style={{ background: `${color}22`, color }}
        >
          <Icon size={18} />
        </div>
        {trend && (
          <span
            className="f-mono text-[11px] flex items-center gap-0.5"
            style={{ color: trend > 0 ? C.teal : C.danger }}
          >
            {trend > 0 ? <TrendingUp size={12} /> : <TrendingDown size={12} />}
            {Math.abs(trend)}%
          </span>
        )}
      </div>
      <div>
        <div className="f-display text-2xl" style={{ color: C.textHi }}>{value}</div>
        <div className="f-body text-[12px]" style={{ color: C.textMid }}>{label}</div>
      </div>
      {sub && <div className="f-mono text-[10px]" style={{ color: C.textDim }}>{sub}</div>}
    </div>
  );
}

function SectionTitle({ eyebrow, title, desc }) {
  return (
    <div className="mb-4">
      {eyebrow && (
        <div className="f-mono text-[11px] tracking-[0.18em] mb-1" style={{ color: C.teal }}>
          {eyebrow}
        </div>
      )}
      <h2 className="f-display text-xl md:text-2xl" style={{ color: C.textHi }}>{title}</h2>
      {desc && <p className="f-body text-[13px] mt-1" style={{ color: C.textMid }}>{desc}</p>}
    </div>
  );
}

/* ============================================================
   SCHEMATIC NER MAP  (not to scale — internal-state schematic only)
============================================================ */
function SchematicMap({ selected, onSelect, alerts, compact }) {
  const affectedStates = new Set(alerts.map((a) => a.region));
  const flowPairs = [
    ["gau", "dib"], ["gau", "shi"], ["gau", "sil"], ["gau", "koh"],
    ["gau", "ita"], ["gau", "gan"], ["sil", "aga"], ["sil", "aiz"],
    ["dim", "koh"], ["dim", "imp"], ["ita", "zir"], ["ita", "alo"],
  ];
  return (
    <div className="relative w-full h-full">
      <svg viewBox="0 0 860 700" className="w-full h-full">
        <defs>
          <radialGradient id="mapGlow" cx="35%" cy="30%" r="70%">
            <stop offset="0%" stopColor="#12261F" />
            <stop offset="100%" stopColor="#0A1713" />
          </radialGradient>
        </defs>
        <rect x="0" y="0" width="860" height="700" fill="url(#mapGlow)" rx="18" />

        {flowPairs.map(([f, t], i) => {
          const a = CITY[f], b = CITY[t];
          const risky = affectedStates.has(a.state) || affectedStates.has(b.state);
          return (
            <line
              key={i} x1={a.x} y1={a.y} x2={b.x} y2={b.y}
              stroke={risky ? C.amber : C.tealDim}
              strokeWidth={risky ? 2 : 1.4}
              opacity={0.75}
              className="route-flow"
            />
          );
        })}

        {CITIES.map((c) => {
          const isSel = selected && (selected.origin === c.id || selected.dest === c.id);
          const flagged = affectedStates.has(c.state);
          const r = c.hub ? 7 : 4.5;
          return (
            <g
              key={c.id}
              transform={`translate(${c.x},${c.y})`}
              onClick={() => onSelect && onSelect(c.id)}
              style={{ cursor: onSelect ? "pointer" : "default" }}
            >
              {flagged && <circle r={r + 6} fill="none" stroke={C.danger} strokeWidth="1.5" opacity="0.55" />}
              <circle
                r={r}
                fill={isSel ? C.amber : flagged ? C.danger : c.hub ? C.teal : C.river}
                stroke={C.ink}
                strokeWidth="1.5"
              />
              {!compact && (
                <text x={r + 6} y={4} className="f-mono" fontSize="11" fill={isSel ? C.amber : C.textMid}>
                  {c.name}
                </text>
              )}
            </g>
          );
        })}
      </svg>
    </div>
  );
}

/* ============================================================
   PAGES
============================================================ */
function OverviewPage({ go }) {
  return (
    <div className="space-y-6">
      <div
        className="relative overflow-hidden rounded-3xl p-6 md:p-10"
        style={{ background: `linear-gradient(135deg, ${C.panel2}, ${C.panel})`, border: `1px solid ${C.line}` }}
      >
        <TopoBackdrop />
        <div className="relative z-10 max-w-2xl">
          <Badge>SIH PROTOTYPE · SMART LOGISTICS</Badge>
          <h1 className="f-display mt-4 text-3xl md:text-[2.6rem] leading-[1.08]" style={{ color: C.textHi }}>
            AI-Based Smart Logistics &amp; Accessibility Intelligence Platform for the North Eastern Region
          </h1>
          <p className="f-body mt-4 text-[14px] md:text-[15px]" style={{ color: C.textMid }}>
            A live decision-support layer that reads terrain, monsoon risk and road quality across the
            eight NER states, then recommends routes, flags accessibility gaps, and surfaces disruptions
            before they cost a shipment.
          </p>
          <div className="flex flex-wrap gap-3 mt-6">
            <button
              onClick={() => go("optimizer")}
              className="f-body inline-flex items-center gap-2 rounded-xl px-4 py-2.5 text-[13px] font-medium"
              style={{ background: C.teal, color: C.ink }}
            >
              Try the AI Route Optimizer <ArrowRight size={15} />
            </button>
            <button
              onClick={() => go("dashboard")}
              className="f-body inline-flex items-center gap-2 rounded-xl px-4 py-2.5 text-[13px] font-medium"
              style={{ border: `1px solid ${C.line}`, color: C.textHi }}
            >
              Open Command Dashboard
            </button>
          </div>
        </div>
      </div>

      <div className="grid md:grid-cols-3 gap-4">
        {[
          { icon: Route, t: "Route Intelligence", d: "Scores every corridor on terrain difficulty, live monsoon exposure, cost and carbon — then explains why a route was picked." },
          { icon: Accessibility, t: "Accessibility Index", d: "Composite score per state covering road, digital, healthcare and PwD/elderly last-mile access." },
          { icon: AlertTriangle, t: "Disruption Feed", d: "Simulated live feed of landslides, floods, road blocks and congestion pulled onto the network map." },
        ].map((f, i) => (
          <div key={i} className="rounded-2xl p-5" style={{ background: C.panel, border: `1px solid ${C.line}` }}>
            <f.icon size={20} color={C.teal} />
            <div className="f-display mt-3 text-[15px]" style={{ color: C.textHi }}>{f.t}</div>
            <div className="f-body mt-1 text-[13px]" style={{ color: C.textMid }}>{f.d}</div>
          </div>
        ))}
      </div>

      <div className="rounded-2xl p-5 grid md:grid-cols-3 gap-6" style={{ background: C.panel, border: `1px solid ${C.line}` }}>
        <div>
          <div className="f-mono text-[11px] tracking-widest" style={{ color: C.teal }}>PROBLEM STATEMENT</div>
          <p className="f-body text-[13px] mt-2" style={{ color: C.textMid }}>
            NER's hilly, monsoon-prone geography and thin road/rail density make freight movement slow and
            unpredictable, while accessibility infrastructure for PwD and elderly citizens lags the rest of
            the country. Planners lack a unified, data-driven view to route around risk and prioritise investment.
          </p>
        </div>
        <div>
          <div className="f-mono text-[11px] tracking-widest" style={{ color: C.teal }}>APPROACH</div>
          <ul className="f-body text-[13px] mt-2 space-y-1.5" style={{ color: C.textMid }}>
            <li>• Terrain + weather-aware route scoring engine</li>
            <li>• Composite accessibility index per state / district</li>
            <li>• Live disruption feed overlaid on the network graph</li>
            <li>• Explainable recommendations,
