import { useState, useMemo } from "react"; import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from "recharts";

// ── Seed Data ────────────────────────────────────────────────────────────────
const STAFF_LIST = [
  { id: "s1", name: "Maria Santos", role: "Head Groom", avatar: "MS" },
  { id: "s2", name: "Jake Ferrier", role: "Farrier", avatar: "JF" },
  { id: "s3", name: "Dr. Harmon", role: "Veterinarian", avatar: "DH" },
  { id: "s4", name: "Tom Bradley", role: "Stable Hand", avatar: "TB" }, ];

const initialHorses = [
  {
    id: 1, name: "Midnight Storm", breed: "Thoroughbred", age: 8, stall: "A1",
    color: "#3b5998", avatar: "🐴", status: "healthy",
    assignedStaff: ["s1", "s3"],
    weightHistory: [
      { date: "2025-10-01", weight: 1150 }, { date: "2025-11-01", weight: 1160 },
      { date: "2025-12-01", weight: 1170 }, { date: "2026-01-01", weight: 1165 },
      { date: "2026-02-01", weight: 1175 }, { date: "2026-03-01", weight: 1180 },
    ],
    lastVet: "2026-02-10", nextFarrier: "2026-03-15", nextVaccine: "2026-04-01",
    notes: "Slightly sensitive left front hoof.",
    feed: "12 lbs grain + hay 3x daily",
    medications: [
      { id: "m1", name: "Bute (Phenylbutazone)", dose: "2g", frequency: "Once daily", startDate: "2026-02-20", endDate: "2026-03-05", active: false, notes: "Post-hoof sensitivity" },
      { id: "m2", name: "Biotin Supplement", dose: "20mg", frequency: "Daily", startDate: "2026-01-01", endDate: null, active: true, notes: "Hoof health support" },
    ],
    logs: [
      { date: "2026-02-28", type: "health", note: "Routine check - all clear", by: "Dr. Harmon" },
      { date: "2026-02-15", type: "farrier", note: "New shoes fitted", by: "Jake Ferrier" },
    ],
  },
  {
    id: 2, name: "Golden Sunrise", breed: "Palomino Quarter Horse", age: 5, stall: "A2",
    color: "#c9a84c", avatar: "🐎", status: "attention",
    assignedStaff: ["s1", "s4"],
    weightHistory: [
      { date: "2025-10-01", weight: 1020 }, { date: "2025-11-01", weight: 1030 },
      { date: "2025-12-01", weight: 1045 }, { date: "2026-01-01", weight: 1040 },
      { date: "2026-02-01", weight: 1035 }, { date: "2026-03-01", weight: 1050 },
    ],
    lastVet: "2026-01-20", nextFarrier: "2026-03-08", nextVaccine: "2026-03-20",
    notes: "Monitoring slight cough. On probiotics.",
    feed: "10 lbs grain + hay 2x daily",
    medications: [
      { id: "m3", name: "Probiotics (FortiFlora)", dose: "1 sachet", frequency: "Daily with feed", startDate: "2026-02-25", endDate: null, active: true, notes: "Gut health & cough management" },
    ],
    logs: [
      { date: "2026-03-01", type: "health", note: "Cough persisting, vet visit scheduled", by: "Barn Staff" },
      { date: "2026-02-20", type: "feeding", note: "Switched to new hay blend", by: "Maria Santos" },
    ],
  },
  {
    id: 3, name: "River Duchess", breed: "Hanoverian", age: 12, stall: "B1",
    color: "#7b5e3a", avatar: "🐴", status: "healthy",
    assignedStaff: ["s1", "s2", "s3"],
    weightHistory: [
      { date: "2025-10-01", weight: 1290 }, { date: "2025-11-01", weight: 1295 },
      { date: "2025-12-01", weight: 1305 }, { date: "2026-01-01", weight: 1310 },
      { date: "2026-02-01", weight: 1315 }, { date: "2026-03-01", weight: 1320 },
    ],
    lastVet: "2026-02-25", nextFarrier: "2026-03-22", nextVaccine: "2026-05-10",
    notes: "Senior care plan. Joint supplements daily.",
    feed: "14 lbs senior grain + soaked hay",
    medications: [
      { id: "m4", name: "Adequan (Polysulfated GAG)", dose: "500mg IM", frequency: "Every 4 days × 7 doses", startDate: "2026-01-15", endDate: "2026-02-12", active: false, notes: "Joint health cycle" },
      { id: "m5", name: "Cosequin ASU", dose: "2 scoops", frequency: "Daily with feed", startDate: "2026-01-01", endDate: null, active: true, notes: "Ongoing joint supplement" },
      { id: "m6", name: "Vitamin E", dose: "5000 IU", frequency: "Daily", startDate: "2025-09-01", endDate: null, active: true, notes: "Neurological & muscle support" },
    ],
    logs: [
      { date: "2026-02-25", type: "health", note: "Joint mobility good, continue supplements", by: "Dr. Harmon" },
    ],
  },
];

// ── Constants ─────────────────────────────────────────────────────────────────
const STATUS = {
  healthy:   { label: "Healthy",         color: "#4ade80", bg: "rgba(74,222,128,0.12)"  },
  attention: { label: "Needs Attention", color: "#f59e0b", bg: "rgba(245,158,11,0.12)"  },
  critical:  { label: "Critical",        color: "#f87171", bg: "rgba(248,113,113,0.12)" },
};
const LOG_ICONS = { health:"🩺", farrier:"🔨", feeding:"🌾", vaccine:"💉", weight:"⚖️", medication:"💊", other:"📝" }; const TABS_DETAIL = ["Overview","Weight","Medications","Care Log","Staff"];

// ── Helpers ───────────────────────────────────────────────────────────────────
const uid = () => Math.random().toString(36).slice(2);
const today = () => new Date().toISOString().split("T")[0];
const avatarColor = (str) => {
  let h = 0; for (let c of str) h = (h * 31 + c.charCodeAt(0)) & 0xffffffff;
  const hue = Math.abs(h) % 360;
  return `hsl(${hue},45%,38%)`;
};

// ── Custom Tooltip ─────────────────────────────────────────────────────────────
const ChartTip = ({ active, payload, label }) => {
  if (!active || !payload?.length) return null;
  return (
    <div style={{ background: "#1c1a16", border: "1px solid rgba(201,168,76,.3)", borderRadius: 4, padding: "8px 12px" }}>
      <div style={{ fontFamily: "'Josefin Sans',sans-serif", fontSize: 10, color: "#c9a84c", letterSpacing: ".1em", marginBottom: 3 }}>{label}</div>
      <div style={{ fontSize: 16, fontWeight: 300, color: "#e8e0d0" }}>{payload[0].value.toLocaleString()} lbs</div>
    </div>
  );
};

// ── Main App ──────────────────────────────────────────────────────────────────
export default function App() {
  const [horses, setHorses]         = useState(initialHorses);
  const [selected, setSelected]     = useState(null);
  const [mainView, setMainView]     = useState("overview"); // overview | detail | addHorse
  const [detailTab, setDetailTab]   = useState("Overview");
  const [filter, setFilter]         = useState("all");
  const [showLogForm, setShowLogForm]   = useState(false);
  const [showMedForm, setShowMedForm]   = useState(false);
  const [showWeightForm, setShowWeightForm] = useState(false);
  const [logForm, setLogForm]   = useState({ type: "health", note: "", by: "" });
  const [medForm, setMedForm]   = useState({ name: "", dose: "", frequency: "", startDate: today(), endDate: "", notes: "", active: true });
  const [weightForm, setWeightForm] = useState({ weight: "", date: today() });
  const [addForm, setAddForm]   = useState({ name:"", breed:"", age:"", stall:"", feed:"", notes:"", status:"healthy", weight:"" });

  const filtered = filter === "all" ? horses : horses.filter(h => h.status === filter);
  const attnCount = horses.filter(h => h.status === "attention").length;
  const critCount = horses.filter(h => h.status === "critical").length;

  // ── Mutations ────────────────────────────────────────────────────────────────
  const updateHorse = (id, patch) => {
    const upd = horses.map(h => h.id === id ? { ...h, ...patch } : h);
    setHorses(upd);
    if (selected?.id === id) setSelected(s => ({ ...s, ...patch }));
  };

  const pickHorse = (h) => { setSelected(h); setMainView("detail"); setDetailTab("Overview"); setShowLogForm(false); setShowMedForm(false); setShowWeightForm(false); };

  const saveLog = () => {
    if (!logForm.note.trim()) return;
    const entry = { date: today(), ...logForm };
    updateHorse(selected.id, { logs: [entry, ...(selected.logs||[])] });
    setLogForm({ type: "health", note: "", by: "" }); setShowLogForm(false);
  };

  const saveMed = () => {
    if (!medForm.name.trim()) return;
    const entry = { id: uid(), ...medForm };
    updateHorse(selected.id, { medications: [...(selected.medications||[]), entry] });
    setMedForm({ name:"", dose:"", frequency:"", startDate: today(), endDate:"", notes:"", active:true }); setShowMedForm(false);
  };

  const toggleMed = (medId) => {
    const meds = selected.medications.map(m => m.id === medId ? { ...m, active: !m.active } : m);
    updateHorse(selected.id, { medications: meds });
  };

  const saveWeight = () => {
    if (!weightForm.weight) return;
    const entry = { date: weightForm.date, weight: Number(weightForm.weight) };
    const wh = [...(selected.weightHistory||[]), entry].sort((a,b)=>a.date.localeCompare(b.date));
    updateHorse(selected.id, { weightHistory: wh, weight: entry.weight });
    const logEntry = { date: today(), type: "weight", note: `Weight recorded: ${entry.weight} lbs`, by: "Barn Staff" };
    updateHorse(selected.id, { logs: [logEntry, ...(selected.logs||[])] });
    setWeightForm({ weight:"", date: today() }); setShowWeightForm(false);
  };

  const toggleStaff = (staffId) => {
    const cur = selected.assignedStaff || [];
    const next = cur.includes(staffId) ? cur.filter(s => s !== staffId) : [...cur, staffId];
    updateHorse(selected.id, { assignedStaff: next });
  };

  const setStatus = (s) => updateHorse(selected.id, { status: s });

  const saveHorse = () => {
    if (!addForm.name.trim()) return;
    const h = { id: Date.now(), ...addForm, age: +addForm.age||0, weight: +addForm.weight||0,
      avatar:"🐴", color:"#6b7280", lastVet:"—", nextFarrier:"—", nextVaccine:"—",
      logs:[], medications:[], weightHistory: addForm.weight ? [{ date: today(), weight: +addForm.weight }] : [],
      assignedStaff:[] };
    setHorses(p=>[...p,h]);
    setAddForm({ name:"",breed:"",age:"",stall:"",feed:"",notes:"",status:"healthy",weight:"" });
    setMainView("overview");
  };

  // ── Weight chart delta ────────────────────────────────────────────────────────
  const weightDelta = useMemo(() => {
    if (!selected?.weightHistory?.length) return null;
    const wh = selected.weightHistory;
    if (wh.length < 2) return null;
    return wh[wh.length-1].weight - wh[0].weight;
  }, [selected]);

  // ── CSS ───────────────────────────────────────────────────────────────────────
  const css = `
    @import url('https://urldefense.com/v3/__https://fonts.googleapis.com/css2?family=Cormorant*Garamond:ital,wght@0,300;0,400;0,600;1,300;1,400&family=Josefin*Sans:wght@300;400;600&display=swap__;Kys!!FRfS_D4!faSOOLFU_IragcLUAsS5ZnuRkHSVhYBhMu2SeUkN1SdZhV7uXlzBq20ihfDkx6C4AxYy9jolHED2twNAzQFoX9ibmqrX$ ');
    *{box-sizing:border-box;margin:0;padding:0}
    body{background:#0d0c0a}
    ::-webkit-scrollbar{width:3px}::-webkit-scrollbar-track{background:#161410}::-webkit-scrollbar-thumb{background:#4a3a2a;border-radius:2px}
    .hcard{transition:all .22s ease;cursor:pointer;border:1px solid rgba(255,255,255,.06);border-radius:5px}
    .hcard:hover{border-color:rgba(201,168,76,.35);background:rgba(255,255,255,.035)!important}
    .hcard.sel{border-color:rgba(201,168,76,.55)!important;background:rgba(201,168,76,.07)!important}
    .bg{background:linear-gradient(135deg,#c9a84c,#e8c96d);color:#1a1208;padding:9px 18px;border-radius:3px;border:none;font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;letter-spacing:.12em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
    .bg:hover{opacity:.88;transform:translateY(-1px)}
    .bo{background:transparent;border:1px solid rgba(201,168,76,.45);color:#c9a84c;padding:7px 14px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
    .bo:hover{background:rgba(201,168,76,.09)}
    .gh{background:transparent;border:1px solid rgba(255,255,255,.09);color:rgba(232,224,208,.55);padding:6px 11px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
    .gh:hover{border-color:rgba(255,255,255,.22);color:#e8e0d0}
    .gh.act{border-color:rgba(201,168,76,.5);color:#c9a84c;background:rgba(201,168,76,.07)}
    .inp{background:rgba(255,255,255,.04);border:1px solid rgba(255,255,255,.1);color:#e8e0d0;padding:9px 12px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:12px;width:100%;outline:none;transition:border .2s}
    .inp:focus{border-color:rgba(201,168,76,.5);background:rgba(201,168,76,.03)}
    .sel2{background:#181510;border:1px solid rgba(255,255,255,.1);color:#e8e0d0;padding:9px 12px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:12px;width:100%;outline:none}
    .dtab{font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;letter-spacing:.13em;text-transform:uppercase;padding:9px 14px;cursor:pointer;border:none;background:none;border-bottom:2px solid transparent;color:rgba(232,224,208,.35);transition:all .2s;flex-shrink:0}
    .dtab:hover{color:rgba(232,224,208,.7)}
    .dtab.on{color:#c9a84c;border-bottom-color:#c9a84c}
    .fi{animation:fi .28s ease}
    @keyframes fi{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:translateY(0)}}
    .lbl{font-family:'Josefin Sans',sans-serif;font-size:10px;letter-spacing:.15em;color:rgba(232,224,208,.45);text-transform:uppercase;display:block;margin-bottom:5px}
    .card{background:rgba(255,255,255,.025);border:1px solid rgba(255,255,255,.07);border-radius:4px;padding:16px}
    .pill{font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;padding:3px 9px;border-radius:20px}
    .med-active{background:rgba(74,222,128,.12);color:#4ade80}
    .med-inactive{background:rgba(255,255,255,.07);color:rgba(232,224,208,.4)}
    .staff-avatar{width:38px;height:38px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Josefin Sans',sans-serif;font-size:12px;font-weight:600;letter-spacing:.05em;flex-shrink:0;transition:all .2s;cursor:pointer;border:2px solid transparent}
    .staff-avatar.assigned{border-color:rgba(201,168,76,.6)!important}
    .divline{border:none;border-top:1px solid rgba(255,255,255,.07);margin:18px 0}
  `;

  // ── Render ────────────────────────────────────────────────────────────────────
  return (
    <div style={{ fontFamily:"'Cormorant Garamond',Georgia,serif", minHeight:"100vh", background:"#0d0c0a", color:"#e8e0d0" }}>
      <style>{css}</style>

      {/* ── Header ── */}
      <div style={{ borderBottom:"1px solid rgba(255,255,255,.07)", padding:"0 26px", display:"flex", alignItems:"center", justifyContent:"space-between", height:58, background:"#0f0d0b" }}>
        <div style={{ display:"flex", alignItems:"center", gap:10, cursor:"pointer" }} onClick={()=>setMainView("overview")}>
          <span style={{ fontSize:21 }}>🐴</span>
          <div>
            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:13, fontWeight:600, letterSpacing:".22em", textTransform:"uppercase", color:"#c9a84c" }}>Paddock</div>
            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".18em", color:"rgba(232,224,208,.35)", textTransform:"uppercase" }}>Barn Health Manager</div>
          </div>
        </div>
        <div style={{ display:"flex", gap:8, alignItems:"center" }}>
          {attnCount > 0 && <span className="pill" style={{ background:"rgba(245,158,11,.18)", color:"#f59e0b" }}>⚠ {attnCount} attention</span>}
          {critCount > 0 && <span className="pill" style={{ background:"rgba(248,113,113,.18)", color:"#f87171" }}>🚨 {critCount} critical</span>}
          <button className="bg" onClick={()=>setMainView("addHorse")}>+ Add Horse</button>
        </div>
      </div>

      <div style={{ display:"flex", height:"calc(100vh - 58px)" }}>

        {/* ── Sidebar ── */}
        <div style={{ width:262, borderRight:"1px solid rgba(255,255,255,.06)", overflowY:"auto", padding:"16px 12px", flexShrink:0, background:"#0f0d0b" }}>
          <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".22em", textTransform:"uppercase", color:"rgba(232,224,208,.3)", marginBottom:9, padding:"0 5px" }}>Filter</div>
          <div style={{ display:"flex", gap:4, marginBottom:16, flexWrap:"wrap" }}>
            {["all","healthy","attention","critical"].map(s=>(
              <button key={s} className={`gh${filter===s?" act":""}`} style={{ fontSize:9, padding:"4px 8px" }} onClick={()=>setFilter(s)}>
                {s==="all"?"All":STATUS[s].label}
              </button>
            ))}
          </div>
          <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".22em", textTransform:"uppercase", color:"rgba(232,224,208,.3)", marginBottom:9, padding:"0 5px" }}>{filtered.length} horses</div>
          {filtered.map(h=>(
            <div key={h.id} className={`hcard${selected?.id===h.id&&mainView==="detail"?" sel":""}`}
              style={{ padding:"12px 12px", marginBottom:6, background:"rgba(255,255,255,.015)" }}
              onClick={()=>pickHorse(h)}>
              <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between" }}>
                <div style={{ display:"flex", alignItems:"center", gap:9 }}>
                  <div style={{ width:33, height:33, borderRadius:"50%", background:h.color+"28", border:`2px solid ${h.color}55`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:16 }}>{h.avatar}</div>
                  <div>
                    <div style={{ fontSize:14, fontWeight:600 }}>{h.name}</div>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.4)", marginTop:1 }}>{h.breed}</div>
                  </div>
                </div>
                <div style={{ width:7, height:7, borderRadius:"50%", background:STATUS[h.status]?.color, flexShrink:0 }}/>
              </div>
              <div style={{ marginTop:8, display:"flex", gap:10, alignItems:"center" }}>
                <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"rgba(232,224,208,.35)" }}>Stall {h.stall}</span>
                <span style={{ color:"rgba(232,224,208,.18)" }}>·</span>
                <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"rgba(232,224,208,.35)" }}>{h.age}yr</span>
                {(h.medications||[]).filter(m=>m.active).length > 0 &&
                  <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"rgba(74,222,128,.6)" }}>💊 {h.medications.filter(m=>m.active).length}</span>}
              </div>
            </div>
          ))}
        </div>

        {/* ── Main Panel ── */}
        <div style={{ flex:1, overflowY:"auto", padding:"28px 34px" }}>

          {/* ════ OVERVIEW ════ */}
          {mainView==="overview" && (
            <div className="fi">
              <div style={{ fontSize:28, fontStyle:"italic", fontWeight:300, marginBottom:3 }}>Good morning,</div>
              <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.4)", letterSpacing:".1em", marginBottom:26 }}>Your barn at a glance — {new Date().toLocaleDateString("en-US",{weekday:"long",month:"long",day:"numeric"})}</div>

              {/* Stats Row */}
              <div style={{ display:"grid", gridTemplateColumns:"repeat(4,1fr)", gap:12, marginBottom:28 }}>
                {[
                  { l:"Total Horses", v:horses.length, i:"🐴", c:null },
                  { l:"Healthy", v:horses.filter(h=>h.status==="healthy").length, i:"✅", c:"#4ade80" },
                  { l:"Need Attention", v:attnCount, i:"⚠️", c:"#f59e0b" },
                  { l:"Active Meds", v:horses.reduce((acc,h)=>acc+(h.medications||[]).filter(m=>m.active).length,0), i:"💊", c:"#a78bfa" },
                ].map(s=>(
                  <div key={s.l} className="card">
                    <div style={{ fontSize:19 }}>{s.i}</div>
                    <div style={{ fontSize:24, fontWeight:300, color:s.c||"#e8e0d0", marginTop:6 }}>{s.v}</div>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"rgba(232,224,208,.35)", letterSpacing:".1em", marginTop:3 }}>{s.l}</div>
                  </div>
                ))}
              </div>

              {/* Upcoming */}
              <div style={{ fontSize:18, fontStyle:"italic", fontWeight:300, marginBottom:13 }}>Upcoming Care</div>
              <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:10, marginBottom:26 }}>
                {horses.flatMap(h=>[
                  h.nextFarrier&&h.nextFarrier!=="—"&&{ horse:h.name, type:"Farrier", date:h.nextFarrier, icon:"🔨" },
                  h.nextVaccine&&h.nextVaccine!=="—"&&{ horse:h.name, type:"Vaccine", date:h.nextVaccine, icon:"💉" },
                ].filter(Boolean)).sort((a,b)=>a.date.localeCompare(b.date)).slice(0,6).map((item,i)=>(
                  <div key={i} className="card" style={{ display:"flex", alignItems:"center", gap:11 }}>
                    <span style={{ fontSize:18 }}>{item.icon}</span>
                    <div>
                      <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"#c9a84c", letterSpacing:".08em" }}>{item.type}</div>
                      <div style={{ fontSize:14, fontWeight:300 }}>{item.horse}</div>
                      <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.38)", marginTop:1 }}>{item.date}</div>
                    </div>
                  </div>
                ))}
              </div>

              {/* Active Medications Overview */}
              <div style={{ fontSize:18, fontStyle:"italic", fontWeight:300, marginBottom:13 }}>Active Medications</div>
              <div style={{ display:"flex", flexDirection:"column", gap:8 }}>
                {horses.flatMap(h=>(h.medications||[]).filter(m=>m.active).map(m=>({...m, horseName:h.name}))).length === 0 &&
                  <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.3)" }}>No active medications.</div>}
                {horses.flatMap(h=>(h.medications||[]).filter(m=>m.active).map(m=>({...m, horseName:h.name}))).map(m=>(
                  <div key={m.id} className="card" style={{ display:"flex", alignItems:"center", gap:14 }}>
                    <span style={{ fontSize:20 }}>💊</span>
                    <div style={{ flex:1 }}>
                      <div style={{ display:"flex", gap:8, alignItems:"center" }}>
                        <span style={{ fontSize:14, fontWeight:400 }}>{m.name}</span>
                        <span className="pill med-active">Active</span>
                      </div>
                      <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.4)", marginTop:2 }}>
                        {m.horseName} · {m.dose} · {m.frequency}
                      </div>
                    </div>
                  </div>
                ))}
              </div>

              <div style={{ marginTop:24, fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.28)", letterSpacing:".05em" }}>← Select a horse from the sidebar to view full profile</div>
            </div>
          )}

          {/* ════ HORSE DETAIL ════ */}
          {mainView==="detail" && selected && (
            <div className="fi">
              {/* Horse Header */}
              <div style={{ display:"flex", alignItems:"flex-start", justifyContent:"space-between", marginBottom:20 }}>
                <div style={{ display:"flex", gap:16, alignItems:"center" }}>
                  <div style={{ width:62, height:62, borderRadius:"50%", background:selected.color+"28", border:`2px solid ${selected.color}66`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:30 }}>{selected.avatar}</div>
                  <div>
                    <div style={{ fontSize:28, fontStyle:"italic", fontWeight:300 }}>{selected.name}</div>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.4)", letterSpacing:".08em", marginTop:2 }}>{selected.breed} · {selected.age}yr · Stall {selected.stall}</div>
                    <span className="pill" style={{ marginTop:7, display:"inline-block", background:STATUS[selected.status].bg, color:STATUS[selected.status].color }}>● {STATUS[selected.status].label}</span>
                  </div>
                </div>
                <div style={{ display:"flex", gap:5, flexWrap:"wrap" }}>
                  {Object.entries(STATUS).map(([k,v])=>(
                    <button key={k} className="gh" style={{ fontSize:9, color:selected.status===k?v.color:undefined, borderColor:selected.status===k?v.color:undefined }} onClick={()=>setStatus(k)}>{v.label}</button>
                  ))}
                </div>
              </div>

              {/* Tab Bar */}
              <div style={{ display:"flex", borderBottom:"1px solid rgba(255,255,255,.07)", marginBottom:22, gap:0 }}>
                {TABS_DETAIL.map(t=>(
                  <button key={t} className={`dtab${detailTab===t?" on":""}`} onClick={()=>setDetailTab(t)}>{t}</button>
                ))}
              </div>

              {/* ── TAB: Overview ── */}
              {detailTab==="Overview" && (
                <div className="fi">
                  <div style={{ display:"grid", gridTemplateColumns:"repeat(3,1fr)", gap:12, marginBottom:20 }}>
                    {[
                      { l:"Current Weight", v: selected.weightHistory?.length ? `${selected.weightHistory[selected.weightHistory.length-1].weight.toLocaleString()} lbs` : "—", i:"⚖️" },
                      { l:"Last Vet Visit",  v: selected.lastVet,    i:"🩺" },
                      { l:"Next Farrier",    v: selected.nextFarrier, i:"🔨" },
                      { l:"Next Vaccine",    v: selected.nextVaccine, i:"💉" },
                      { l:"Feed Plan",       v: selected.feed||"—",   i:"🌾" },
                      { l:"Active Meds",     v: `${(selected.medications||[]).filter(m=>m.active).length} medication(s)`, i:"💊" },
                    ].map(x=>(
                      <div key={x.l} className="card">
                        <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"#c9a84c", letterSpacing:".15em", textTransform:"uppercase", marginBottom:6 }}>{x.i} {x.l}</div>
                        <div style={{ fontSize:13, fontWeight:300, lineHeight:1.4 }}>{x.v}</div>
                      </div>
                    ))}
                  </div>
                  {selected.notes && (
                    <div className="card" style={{ marginBottom:14 }}>
                      <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"#c9a84c", letterSpacing:".15em", textTransform:"uppercase", marginBottom:7 }}>📋 Notes</div>
                      <div style={{ fontSize:14, fontWeight:300, lineHeight:1.5 }}>{selected.notes}</div>
                    </div>
                  )}
                  {/* Assigned Staff preview */}
                  <div className="card">
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, color:"#c9a84c", letterSpacing:".15em", textTransform:"uppercase", marginBottom:10 }}>👥 Assigned Staff</div>
                    {(selected.assignedStaff||[]).length === 0 && <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.3)" }}>No staff assigned yet.</div>}
                    <div style={{ display:"flex", gap:8, flexWrap:"wrap" }}>
                      {STAFF_LIST.filter(s=>(selected.assignedStaff||[]).includes(s.id)).map(s=>(
                        <div key={s.id} style={{ display:"flex", alignItems:"center", gap:8 }}>
                          <div className="staff-avatar assigned" style={{ background: avatarColor(s.name), color:"#fff", width:32, height:32, fontSize:11 }}>{s.avatar}</div>
                          <div>
                            <div style={{ fontSize:13, fontWeight:300 }}>{s.name}</div>
                            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.4)" }}>{s.role}</div>
                          </div>
                        </div>
                      ))}
                    </div>
                  </div>
                </div>
              )}

              {/* ── TAB: Weight ── */}
              {detailTab==="Weight" && (
                <div className="fi">
                  <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between", marginBottom:18 }}>
                    <div>
                      <div style={{ fontSize:22, fontStyle:"italic", fontWeight:300 }}>Weight History</div>
                      {weightDelta !== null && (
                        <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, marginTop:3, color: weightDelta >= 0 ? "#4ade80" : "#f87171" }}>
                          {weightDelta >= 0 ? "▲" : "▼"} {Math.abs(weightDelta)} lbs since first record
                        </div>
                      )}
                    </div>
                    <button className="bo" onClick={()=>setShowWeightForm(!showWeightForm)}>+ Log Weight</button>
                  </div>

                  {showWeightForm && (
                    <div className="fi card" style={{ border:"1px solid rgba(201,168,76,.2)", marginBottom:18, background:"rgba(201,168,76,.04)" }}>
                      <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12, marginBottom:12 }}>
                        <div><label className="lbl">Weight (lbs)</label><input className="inp" type="number" placeholder="e.g. 1180" value={weightForm.weight} onChange={e=>setWeightForm(p=>({...p,weight:e.target.value}))}/></div>
                        <div><label className="lbl">Date</label><input className="inp" type="date" value={weightForm.date} onChange={e=>setWeightForm(p=>({...p,date:e.target.value}))}/></div>
                      </div>
                      <div style={{ display:"flex", gap:8 }}>
                        <button className="bg" onClick={saveWeight}>Save</button>
                        <button className="gh" onClick={()=>setShowWeightForm(false)}>Cancel</button>
                      </div>
                    </div>
                  )}

                  {(selected.weightHistory||[]).length < 2 ? (
                    <div className="card" style={{ textAlign:"center", padding:"40px 20px" }}>
                      <div style={{ fontSize:28, marginBottom:10 }}>⚖️</div>
                      <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.35)", letterSpacing:".1em" }}>Log at least 2 weight entries to see the chart</div>
                    </div>
                  ) : (
                    <div className="card" style={{ padding:"20px 10px 10px" }}>
                      <ResponsiveContainer width="100%" height={220}>
                        <LineChart data={selected.weightHistory} margin={{ top:5, right:20, left:0, bottom:5 }}>
                          <CartesianGrid strokeDasharray="3 3" stroke="rgba(255,255,255,.05)"/>
                          <XAxis dataKey="date" tick={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, fill:"rgba(232,224,208,.35)", letterSpacing:".05em" }} tickLine={false} axisLine={{ stroke:"rgba(255,255,255,.08)" }}/>
                          <YAxis tick={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, fill:"rgba(232,224,208,.35)" }} tickLine={false} axisLine={false} domain={["auto","auto"]}/>
                          <Tooltip content={<ChartTip/>}/>
                          <Line type="monotone" dataKey="weight" stroke="#c9a84c" strokeWidth={2} dot={{ r:4, fill:"#c9a84c", stroke:"#0d0c0a", strokeWidth:2 }} activeDot={{ r:6 }}/>
                        </LineChart>
                      </ResponsiveContainer>
                    </div>
                  )}

                  {/* Weight log table */}
                  <div style={{ marginTop:18 }}>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".18em", textTransform:"uppercase", color:"rgba(232,224,208,.3)", marginBottom:10 }}>All Records</div>
                    {[...(selected.weightHistory||[])].reverse().map((w,i)=>(
                      <div key={i} style={{ display:"flex", justifyContent:"space-between", padding:"9px 0", borderBottom:"1px solid rgba(255,255,255,.05)" }}>
                        <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.45)" }}>{w.date}</span>
                        <span style={{ fontSize:15, fontWeight:300 }}>{w.weight.toLocaleString()} lbs</span>
                      </div>
                    ))}
                  </div>
                </div>
              )}

              {/* ── TAB: Medications ── */}
              {detailTab==="Medications" && (
                <div className="fi">
                  <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between", marginBottom:18 }}>
                    <div style={{ fontSize:22, fontStyle:"italic", fontWeight:300 }}>Medications</div>
                    <button className="bo" onClick={()=>setShowMedForm(!showMedForm)}>+ Add Medication</button>
                  </div>

                  {showMedForm && (
                    <div className="fi card" style={{ border:"1px solid rgba(201,168,76,.2)", marginBottom:18, background:"rgba(201,168,76,.04)" }}>
                      <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12, marginBottom:12 }}>
                        <div style={{ gridColumn:"1/-1" }}><label className="lbl">Medication Name</label><input className="inp" placeholder="e.g. Bute (Phenylbutazone)" value={medForm.name} onChange={e=>setMedForm(p=>({...p,name:e.target.value}))}/></div>
                        <div><label className="lbl">Dose</label><input className="inp" placeholder="e.g. 2g" value={medForm.dose} onChange={e=>setMedForm(p=>({...p,dose:e.target.value}))}/></div>
                        <div><label className="lbl">Frequency</label><input className="inp" placeholder="e.g. Twice daily" value={medForm.frequency} onChange={e=>setMedForm(p=>({...p,frequency:e.target.value}))}/></div>
                        <div><label className="lbl">Start Date</label><input className="inp" type="date" value={medForm.startDate} onChange={e=>setMedForm(p=>({...p,startDate:e.target.value}))}/></div>
                        <div><label className="lbl">End Date (optional)</label><input className="inp" type="date" value={medForm.endDate} onChange={e=>setMedForm(p=>({...p,endDate:e.target.value}))}/></div>
                        <div style={{ gridColumn:"1/-1" }}><label className="lbl">Notes</label><input className="inp" placeholder="Purpose, prescribing vet, special instructions..." value={medForm.notes} onChange={e=>setMedForm(p=>({...p,notes:e.target.value}))}/></div>
                      </div>
                      <div style={{ display:"flex", gap:8 }}>
                        <button className="bg" onClick={saveMed}>Save Medication</button>
                        <button className="gh" onClick={()=>setShowMedForm(false)}>Cancel</button>
                      </div>
                    </div>
                  )}

                  {(selected.medications||[]).length === 0 && <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.3)" }}>No medications recorded.</div>}

                  {/* Active */}
                  {(selected.medications||[]).filter(m=>m.active).length > 0 && <>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".18em", textTransform:"uppercase", color:"rgba(232,224,208,.3)", marginBottom:10 }}>Active</div>
                    {(selected.medications||[]).filter(m=>m.active).map(m=>(
                      <div key={m.id} className="card" style={{ marginBottom:9 }}>
                        <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start" }}>
                          <div style={{ flex:1 }}>
                            <div style={{ display:"flex", gap:8, alignItems:"center", marginBottom:5 }}>
                              <span style={{ fontSize:15, fontWeight:400 }}>💊 {m.name}</span>
                              <span className="pill med-active">Active</span>
                            </div>
                            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.45)", lineHeight:1.6 }}>
                              <span>{m.dose}</span> · <span>{m.frequency}</span><br/>
                              Started {m.startDate}{m.endDate&&` · Ends ${m.endDate}`}
                            </div>
                            {m.notes && <div style={{ fontSize:13, fontStyle:"italic", color:"rgba(232,224,208,.5)", marginTop:6 }}>{m.notes}</div>}
                          </div>
                          <button className="gh" style={{ marginLeft:12, fontSize:9 }} onClick={()=>toggleMed(m.id)}>Mark Inactive</button>
                        </div>
                      </div>
                    ))}
                  </>}

                  {/* Inactive */}
                  {(selected.medications||[]).filter(m=>!m.active).length > 0 && <>
                    <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:9, letterSpacing:".18em", textTransform:"uppercase", color:"rgba(232,224,208,.25)", marginBottom:10, marginTop:18 }}>Past / Inactive</div>
                    {(selected.medications||[]).filter(m=>!m.active).map(m=>(
                      <div key={m.id} className="card" style={{ marginBottom:9, opacity:.6 }}>
                        <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start" }}>
                          <div style={{ flex:1 }}>
                            <div style={{ display:"flex", gap:8, alignItems:"center", marginBottom:5 }}>
                              <span style={{ fontSize:14, fontWeight:300 }}>💊 {m.name}</span>
                              <span className="pill med-inactive">Inactive</span>
                            </div>
                            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.35)" }}>
                              {m.dose} · {m.frequency} · Started {m.startDate}
                            </div>
                            {m.notes && <div style={{ fontSize:12, fontStyle:"italic", color:"rgba(232,224,208,.35)", marginTop:5 }}>{m.notes}</div>}
                          </div>
                          <button className="gh" style={{ marginLeft:12, fontSize:9 }} onClick={()=>toggleMed(m.id)}>Reactivate</button>
                        </div>
                      </div>
                    ))}
                  </>}
                </div>
              )}

              {/* ── TAB: Care Log ── */}
              {detailTab==="Care Log" && (
                <div className="fi">
                  <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between", marginBottom:18 }}>
                    <div style={{ fontSize:22, fontStyle:"italic", fontWeight:300 }}>Care Log</div>
                    <button className="bo" onClick={()=>setShowLogForm(!showLogForm)}>+ Add Entry</button>
                  </div>

                  {showLogForm && (
                    <div className="fi card" style={{ border:"1px solid rgba(201,168,76,.2)", marginBottom:18, background:"rgba(201,168,76,.04)" }}>
                      <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12, marginBottom:12 }}>
                        <div><label className="lbl">Type</label>
                          <select className="sel2" value={logForm.type} onChange={e=>setLogForm(p=>({...p,type:e.target.value}))}>
                            {Object.entries(LOG_ICONS).map(([k,v])=><option key={k} value={k}>{v} {k[0].toUpperCase()+k.slice(1)}</option>)}
                          </select>
                        </div>
                        <div><label className="lbl">Recorded By</label><input className="inp" placeholder="Name" value={logForm.by} onChange={e=>setLogForm(p=>({...p,by:e.target.value}))}/></div>
                      </div>
                      <div style={{ marginBottom:14 }}><label className="lbl">Note</label><input className="inp" placeholder="Describe observation or treatment..." value={logForm.note} onChange={e=>setLogForm(p=>({...p,note:e.target.value}))}/></div>
                      <div style={{ display:"flex", gap:8 }}>
                        <button className="bg" onClick={saveLog}>Save Entry</button>
                        <button className="gh" onClick={()=>setShowLogForm(false)}>Cancel</button>
                      </div>
                    </div>
                  )}

                  {(selected.logs||[]).length===0 && <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.3)" }}>No entries yet.</div>}
                  {(selected.logs||[]).map((l,i)=>(
                    <div key={i} style={{ display:"flex", gap:12, padding:"12px 0", borderBottom:"1px solid rgba(255,255,255,.05)" }}>
                      <div style={{ width:33, height:33, borderRadius:"50%", background:"rgba(255,255,255,.04)", display:"flex", alignItems:"center", justifyContent:"center", fontSize:15, flexShrink:0 }}>{LOG_ICONS[l.type]||"📝"}</div>
                      <div>
                        <div style={{ display:"flex", alignItems:"center", gap:8, marginBottom:3 }}>
                          <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"#c9a84c", letterSpacing:".08em", textTransform:"uppercase" }}>{l.type}</span>
                          <span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.3)" }}>{l.date}</span>
                          {l.by&&<span style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.3)" }}>· {l.by}</span>}
                        </div>
                        <div style={{ fontSize:14, fontWeight:300 }}>{l.note}</div>
                      </div>
                    </div>
                  ))}
                </div>
              )}

              {/* ── TAB: Staff ── */}
              {detailTab==="Staff" && (
                <div className="fi">
                  <div style={{ fontSize:22, fontStyle:"italic", fontWeight:300, marginBottom:5 }}>Staff Assignment</div>
                  <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.4)", letterSpacing:".05em", marginBottom:22 }}>Toggle staff members to assign or unassign them to {selected.name}</div>
                  <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12 }}>
                    {STAFF_LIST.map(s=>{
                      const assigned = (selected.assignedStaff||[]).includes(s.id);
                      return (
                        <div key={s.id} className="card" style={{ display:"flex", alignItems:"center", gap:14, cursor:"pointer", border: assigned ? "1px solid rgba(201,168,76,.35)" : "1px solid rgba(255,255,255,.07)", background: assigned ? "rgba(201,168,76,.06)" : "rgba(255,255,255,.025)", transition:"all .2s" }}
                          onClick={()=>toggleStaff(s.id)}>
                          <div className={`staff-avatar${assigned?" assigned":""}`} style={{ background: avatarColor(s.name), color:"#fff", fontSize:12 }}>{s.avatar}</div>
                          <div style={{ flex:1 }}>
                            <div style={{ fontSize:15, fontWeight:assigned?400:300 }}>{s.name}</div>
                            <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.4)", marginTop:2 }}>{s.role}</div>
                          </div>
                          <div style={{ width:18, height:18, borderRadius:"50%", border:"1.5px solid", borderColor: assigned ? "#c9a84c" : "rgba(255,255,255,.2)", background: assigned ? "#c9a84c" : "transparent", display:"flex", alignItems:"center", justifyContent:"center", fontSize:10, color:"#1a1208", transition:"all .2s" }}>
                            {assigned ? "✓" : ""}
                          </div>
                        </div>
                      );
                    })}
                  </div>

                  <hr className="divline"/>
                  <div style={{ fontSize:16, fontStyle:"italic", fontWeight:300, marginBottom:12 }}>Currently Assigned ({(selected.assignedStaff||[]).length})</div>
                  {(selected.assignedStaff||[]).length === 0 && <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.3)" }}>No staff assigned.</div>}
                  <div style={{ display:"flex", flexDirection:"column", gap:8 }}>
                    {STAFF_LIST.filter(s=>(selected.assignedStaff||[]).includes(s.id)).map(s=>(
                      <div key={s.id} style={{ display:"flex", alignItems:"center", gap:12, padding:"10px 14px", background:"rgba(201,168,76,.05)", borderRadius:4, border:"1px solid rgba(201,168,76,.15)" }}>
                        <div className="staff-avatar assigned" style={{ background: avatarColor(s.name), color:"#fff", fontSize:12 }}>{s.avatar}</div>
                        <div>
                          <div style={{ fontSize:15, fontWeight:300 }}>{s.name}</div>
                          <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:10, color:"rgba(232,224,208,.4)" }}>{s.role}</div>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>
              )}
            </div>
          )}

          {/* ════ ADD HORSE ════ */}
          {mainView==="addHorse" && (
            <div className="fi" style={{ maxWidth:580 }}>
              <div style={{ fontSize:28, fontStyle:"italic", fontWeight:300, marginBottom:3 }}>Register New Horse</div>
              <div style={{ fontFamily:"'Josefin Sans',sans-serif", fontSize:11, color:"rgba(232,224,208,.4)", letterSpacing:".08em", marginBottom:26 }}>Add a horse to your barn's care roster</div>
              <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:13 }}>
                {[{l:"Horse Name *",k:"name",p:"e.g. Midnight Storm"},{l:"Breed",k:"breed",p:"e.g. Thoroughbred"},{l:"Age",k:"age",p:"Years"},{l:"Stall",k:"stall",p:"e.g. A1"},{l:"Starting Weight (lbs)",k:"weight",p:"e.g. 1200"}].map(f=>(
                  <div key={f.k}><label className="lbl">{f.l}</label><input className="inp" placeholder={f.p} value={addForm[f.k]} onChange={e=>setAddForm(p=>({...p,[f.k]:e.target.value}))}/></div>
                ))}
                <div><label className="lbl">Initial Status</label>
                  <select className="sel2" value={addForm.status} onChange={e=>setAddForm(p=>({...p,status:e.target.value}))}>
                    {Object.entries(STATUS).map(([k,v])=><option key={k} value={k}>{v.label}</option>)}
                  </select>
                </div>
              </div>
              <div style={{ marginTop:13 }}><label className="lbl">Feed Plan</label><input className="inp" placeholder="e.g. 12 lbs grain + hay 3x daily" value={addForm.feed} onChange={e=>setAddForm(p=>({...p,feed:e.target.value}))}/></div>
              <div style={{ marginTop:13 }}><label className="lbl">Notes</label><input className="inp" placeholder="Special care instructions or health conditions..." value={addForm.notes} onChange={e=>setAddForm(p=>({...p,notes:e.target.value}))}/></div>
              <div style={{ display:"flex", gap:10, marginTop:22 }}>
                <button className="bg" onClick={saveHorse}>Register Horse</button>
                <button className="gh" onClick={()=>setMainView("overview")}>Cancel</button>
              </div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}

