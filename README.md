import React, { useState, useMemo, useEffect } from "react";
import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from "recharts";

/* ─── Persistence ────────────────────────────────────────────────────────────── */
const STORAGE_KEY = "paddock_v1";
const save = (data) => { try { localStorage.setItem(STORAGE_KEY, JSON.stringify(data)); } catch(e) {} };
const load = () => { try { const d = localStorage.getItem(STORAGE_KEY); return d ? JSON.parse(d) : null; } catch(e) { return null; } };

/* ─── Constants ──────────────────────────────────────────────────────────────── */
const STATUS = {
  healthy:   { label:"Healthy",         color:"#4ade80", bg:"rgba(74,222,128,0.12)"  },
  attention: { label:"Needs Attention", color:"#f59e0b", bg:"rgba(245,158,11,0.12)"  },
  critical:  { label:"Critical",        color:"#f87171", bg:"rgba(248,113,113,0.12)" },
};
const LOG_ICONS = { health:"🩺", farrier:"🔨", feeding:"🌾", vaccine:"💉", weight:"⚖️", medication:"💊", other:"📝" };
const TABS = ["Overview","Weight","Medications","Appointments","Feeding","Care Log","Staff"];
const ROLES = ["Barn Owner","Head Groom","Veterinarian","Farrier","Stable Hand","Trainer"];
const PLANS = [
  { id:"starter", name:"Starter", price:"$19", period:"/mo", horses:5,  staff:3,  color:"#c9a84c", features:["Up to 5 horses","3 staff accounts","Health & care logs","PDF export"] },
  { id:"pro",     name:"Pro",     price:"$49", period:"/mo", horses:20, staff:10, color:"#7dd3fc", features:["Up to 20 horses","10 staff accounts","Everything in Starter","Weight charts","Medication tracking","Appointment scheduling"], popular:true },
  { id:"barn",    name:"Barn",    price:"$99", period:"/mo", horses:999,staff:999,color:"#a78bfa", features:["Unlimited horses","Unlimited staff","Everything in Pro","Custom branding","Priority support","Multi-barn management"] },
];

const uid  = () => Math.random().toString(36).slice(2);
const today = () => new Date().toISOString().split("T")[0];
const avatarColor = str => { let h=0; for(let c of str) h=(h*31+c.charCodeAt(0))&0xffffffff; return `hsl(${Math.abs(h)%360},42%,36%)`; };
const initials = name => name.split(" ").map(w=>w[0]).join("").toUpperCase().slice(0,2);

/* ─── PDF Export ─────────────────────────────────────────────────────────────── */
function exportPDF(horse, barnName) {
  const win = window.open("","_blank"); if(!win) return;
  const activeMeds = (horse.medications||[]).filter(m=>m.active);
  const upcomingAppts = (horse.appointments||[]).filter(a=>a.status==="scheduled").sort((a,b)=>a.date.localeCompare(b.date));
  const reminders = (horse.feedingReminders||[]).filter(r=>r.active).sort((a,b)=>a.time.localeCompare(b.time));
  const recentLogs = (horse.logs||[]).slice(0,6);
  const latestWeight = horse.weightHistory?.length ? horse.weightHistory[horse.weightHistory.length-1] : null;
  win.document.write(`<!DOCTYPE html><html><head><meta charset="utf-8"><title>${barnName} – ${horse.name}</title>
  <style>@import url('https://urldefense.com/v3/__https://fonts.googleapis.com/css2?family=Cormorant*Garamond:ital,wght@0,300;0,400;1,300&family=Josefin*Sans:wght@400;600&display=swap__;Kys!!FRfS_D4!YgckYrkKpuvdGHRduUY7zealOiq0_6HW-k9EsuI7295-TXjJFOvq6qcD2x2GGqTRYulA8LAH0j8m7KW35IV9rZUyxoMs$ ');
  *{box-sizing:border-box;margin:0;padding:0}body{font-family:'Cormorant Garamond',Georgia,serif;background:#fff;color:#1a1208;padding:40px 48px;max-width:780px;margin:0 auto}
  .header{display:flex;align-items:flex-start;justify-content:space-between;border-bottom:2px solid #c9a84c;padding-bottom:18px;margin-bottom:24px}
  .brand{font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.2em;text-transform:uppercase;color:#c9a84c;margin-bottom:4px}
  .hname{font-size:32px;font-style:italic;font-weight:300}.meta{font-family:'Josefin Sans',sans-serif;font-size:11px;color:#6b5a42;margin-top:3px}
  .badge{font-family:'Josefin Sans',sans-serif;font-size:9px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;padding:3px 9px;border-radius:20px;display:inline-block;margin-top:7px;color:${STATUS[horse.status].color};border:1px solid ${STATUS[horse.status].color}55}
  .gen{font-family:'Josefin Sans',sans-serif;font-size:10px;color:#aaa;text-align:right}
  h2{font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.2em;text-transform:uppercase;color:#c9a84c;margin:22px 0 10px;padding-bottom:4px;border-bottom:1px solid #e8d9c0}
  .grid2{display:grid;grid-template-columns:1fr 1fr;gap:10px}.card{background:#faf7f2;border:1px solid #e8d9c0;border-radius:3px;padding:11px 13px}
  .cl{font-family:'Josefin Sans',sans-serif;font-size:9px;font-weight:600;letter-spacing:.14em;text-transform:uppercase;color:#c9a84c;margin-bottom:3px}
  .cv{font-size:13px;font-weight:300;line-height:1.4}.row{display:flex;gap:12px;padding:8px 0;border-bottom:1px solid #f0e8dc;align-items:flex-start}
  .dt{font-family:'Josefin Sans',sans-serif;font-size:10px;color:#8b7a62;min-width:85px;padding-top:2px}.ti{font-size:14px;font-weight:400}
  .sm{font-family:'Josefin Sans',sans-serif;font-size:10px;color:#8b7a62;margin-top:2px}.ft{font-family:'Josefin Sans',sans-serif;font-size:12px;font-weight:600;color:#c9a84c;min-width:52px}
  .footer{margin-top:32px;padding-top:12px;border-top:1px solid #e8d9c0;font-family:'Josefin Sans',sans-serif;font-size:9px;color:#bbb;letter-spacing:.1em;text-align:center;text-transform:uppercase}
  </style></head><body>
  <div class="header"><div><div class="brand">🐴 ${barnName} — Powered by Paddock</div><div class="hname">${horse.name}</div>
  <div class="meta">${horse.breed} · ${horse.age} years old · Stall ${horse.stall}</div>
  <span class="badge">● ${STATUS[horse.status].label}</span></div>
  <div class="gen">Generated ${new Date().toLocaleDateString("en-US",{month:"long",day:"numeric",year:"numeric"})}</div></div>
  <h2>Horse Profile</h2><div class="grid2">
  <div class="card"><div class="cl">Current Weight</div><div class="cv">${latestWeight?latestWeight.weight.toLocaleString()+" lbs":"—"}</div></div>
  <div class="card"><div class="cl">Last Vet Visit</div><div class="cv">${horse.lastVet||"—"}</div></div>
  <div class="card"><div class="cl">Next Farrier</div><div class="cv">${horse.nextFarrier||"—"}</div></div>
  <div class="card"><div class="cl">Next Vaccine</div><div class="cv">${horse.nextVaccine||"—"}</div></div>
  <div class="card" style="grid-column:1/-1"><div class="cl">Feed Plan</div><div class="cv">${horse.feed||"—"}</div></div>
  ${horse.notes?`<div class="card" style="grid-column:1/-1"><div class="cl">Notes</div><div class="cv">${horse.notes}</div></div>`:""}
  </div>
  ${upcomingAppts.length?`<h2>Upcoming Appointments</h2>${upcomingAppts.map(a=>`<div class="row"><div class="dt">${a.date}${a.time?" · "+a.time:""}</div><div><div class="ti">${a.type==="vet"?"🩺":"🔨"} ${a.title}</div><div class="sm">${a.provider}${a.notes?" · "+a.notes:""}</div></div></div>`).join("")}`:""}
  ${activeMeds.length?`<h2>Active Medications</h2>${activeMeds.map(m=>`<div class="row"><div style="flex:1"><div class="ti">💊 ${m.name}</div><div class="sm">${m.dose} · ${m.frequency} · Started ${m.startDate}${m.endDate?" · Ends "+m.endDate:""}</div>${m.notes?`<div class="sm" style="font-style:italic">${m.notes}</div>`:""}</div></div>`).join("")}`:""}
  ${reminders.length?`<h2>Feeding Schedule</h2>${reminders.map(r=>`<div class="row"><div class="ft">${r.time}</div><div><div class="ti">${r.label}</div><div class="sm">${r.amount}</div></div></div>`).join("")}`:""}
  ${recentLogs.length?`<h2>Recent Care Log</h2>${recentLogs.map(l=>`<div class="row"><div class="dt">${l.type}</div><div><div class="ti">${l.note}</div><div class="sm">${l.date}${l.by?" · "+l.by:""}</div></div></div>`).join("")}`:""}
  <div class="footer">${barnName} · Powered by Paddock · Confidential Health Record · ${horse.name}</div>
  </body></html>`);
  win.document.close(); setTimeout(()=>win.print(),400);
}

/* ─── Chart Tooltip ──────────────────────────────────────────────────────────── */
const ChartTip = ({active,payload,label}) => {
  if(!active||!payload?.length) return null;
  return <div style={{background:"#1c1a16",border:"1px solid rgba(201,168,76,.3)",borderRadius:4,padding:"8px 12px"}}>
    <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"#c9a84c",marginBottom:2}}>{label}</div>
    <div style={{fontSize:15,fontWeight:300,color:"#e8e0d0"}}>{payload[0].value?.toLocaleString()} lbs</div>
  </div>;
};

/* ─── CSS ────────────────────────────────────────────────────────────────────── */
const CSS = `
  @import url('https://urldefense.com/v3/__https://fonts.googleapis.com/css2?family=Cormorant*Garamond:ital,wght@0,300;0,400;0,600;1,300;1,400&family=Josefin*Sans:wght@300;400;600&display=swap__;Kys!!FRfS_D4!YgckYrkKpuvdGHRduUY7zealOiq0_6HW-k9EsuI7295-TXjJFOvq6qcD2x2GGqTRYulA8LAH0j8m7KW35IV9rTOc7KZF$ ');
  *{box-sizing:border-box;margin:0;padding:0}
  ::-webkit-scrollbar{width:3px}::-webkit-scrollbar-track{background:#161410}::-webkit-scrollbar-thumb{background:#4a3a2a;border-radius:2px}
  .hcard{transition:all .22s;cursor:pointer;border:1px solid rgba(255,255,255,.06);border-radius:5px}
  .hcard:hover{border-color:rgba(201,168,76,.35);background:rgba(255,255,255,.035)!important}
  .hcard.on{border-color:rgba(201,168,76,.55)!important;background:rgba(201,168,76,.07)!important}
  .bg{background:linear-gradient(135deg,#c9a84c,#e8c96d);color:#1a1208;padding:8px 15px;border-radius:3px;border:none;font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
  .bg:hover{opacity:.88;transform:translateY(-1px)}
  .bo{background:transparent;border:1px solid rgba(201,168,76,.45);color:#c9a84c;padding:7px 13px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
  .bo:hover{background:rgba(201,168,76,.09)}
  .gh{background:transparent;border:1px solid rgba(255,255,255,.09);color:rgba(232,224,208,.55);padding:5px 10px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;cursor:pointer;transition:all .2s;white-space:nowrap}
  .gh:hover{border-color:rgba(255,255,255,.22);color:#e8e0d0}
  .gh.act{border-color:rgba(201,168,76,.5);color:#c9a84c;background:rgba(201,168,76,.07)}
  .inp{background:rgba(255,255,255,.04);border:1px solid rgba(255,255,255,.1);color:#e8e0d0;padding:9px 12px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:13px;width:100%;outline:none;transition:border .2s}
  .inp:focus{border-color:rgba(201,168,76,.5);background:rgba(201,168,76,.03)}
  .inp-light{background:rgba(0,0,0,.04);border:1px solid rgba(0,0,0,.12);color:#2a1f0e;padding:9px 12px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:13px;width:100%;outline:none;transition:border .2s}
  .inp-light:focus{border-color:#c9a84c}
  .sel2{background:#181510;border:1px solid rgba(255,255,255,.1);color:#e8e0d0;padding:8px 11px;border-radius:3px;font-family:'Josefin Sans',sans-serif;font-size:12px;width:100%;outline:none}
  .dtab{font-family:'Josefin Sans',sans-serif;font-size:10px;font-weight:600;letter-spacing:.12em;text-transform:uppercase;padding:8px 11px;cursor:pointer;border:none;background:none;border-bottom:2px solid transparent;color:rgba(232,224,208,.32);transition:all .2s;flex-shrink:0}
  .dtab:hover{color:rgba(232,224,208,.65)}
  .dtab.on{color:#c9a84c;border-bottom-color:#c9a84c}
  .fi{animation:fi .26s ease}
  @keyframes fi{from{opacity:0;transform:translateY(5px)}to{opacity:1;transform:translateY(0)}}
  .lbl{font-family:'Josefin Sans',sans-serif;font-size:10px;letter-spacing:.14em;color:rgba(232,224,208,.42);text-transform:uppercase;display:block;margin-bottom:5px}
  .lbl-light{font-family:'Josefin Sans',sans-serif;font-size:10px;letter-spacing:.14em;color:rgba(42,31,14,.5);text-transform:uppercase;display:block;margin-bottom:5px}
  .card{background:rgba(255,255,255,.025);border:1px solid rgba(255,255,255,.07);border-radius:4px;padding:14px}
  .pill{font-family:'Josefin Sans',sans-serif;font-size:9px;font-weight:600;letter-spacing:.1em;text-transform:uppercase;padding:2px 8px;border-radius:20px}
  .cap{font-family:'Josefin Sans',sans-serif;font-size:9px;letter-spacing:.2em;text-transform:uppercase;color:rgba(232,224,208,.28)}
  .divline{border:none;border-top:1px solid rgba(255,255,255,.07);margin:16px 0}
  .appt-card{border-radius:4px;padding:12px 14px;margin-bottom:8px;border-left:3px solid transparent;background:rgba(255,255,255,.02);transition:all .2s}
  .appt-vet{border-left-color:#7dd3fc}
  .appt-farrier{border-left-color:#c9a84c}
  .appt-completed{opacity:.4}
  .feed-row{display:flex;align-items:center;gap:10px;padding:10px 0;border-bottom:1px solid rgba(255,255,255,.06)}
  .staff-av{width:36px;height:36px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Josefin Sans',sans-serif;font-size:11px;font-weight:600;cursor:pointer;border:2px solid transparent;transition:all .2s;flex-shrink:0}
  .staff-av.on{border-color:rgba(201,168,76,.6)!important}
  .plan-card{border:1px solid rgba(255,255,255,.1);border-radius:8px;padding:28px 24px;transition:all .25s;cursor:pointer;position:relative}
  .plan-card:hover{transform:translateY(-3px);border-color:rgba(201,168,76,.35)}
  .plan-card.popular{border-color:rgba(125,211,252,.45)}
  .step-dot{width:32px;height:32px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Josefin Sans',sans-serif;font-size:13px;font-weight:600;flex-shrink:0}
  .wizard-inp{background:rgba(0,0,0,.06);border:1px solid rgba(201,168,76,.3);color:#2a1a08;padding:12px 14px;border-radius:4px;font-family:'Josefin Sans',sans-serif;font-size:14px;width:100%;outline:none;transition:all .2s}
  .wizard-inp:focus{border-color:#c9a84c;background:rgba(201,168,76,.05)}
  .wizard-sel{background:#fff;border:1px solid rgba(201,168,76,.3);color:#2a1a08;padding:12px 14px;border-radius:4px;font-family:'Josefin Sans',sans-serif;font-size:14px;width:100%;outline:none}
`;

/* ═══════════════════════════════════════════════════════════════════════════════
   SCREENS
═══════════════════════════════════════════════════════════════════════════════ */

/* ── Landing ── */
function Landing({ onGetStarted, onLogin }) {
  return (
    <div style={{minHeight:"100vh",background:"#0d0c0a",color:"#e8e0d0",display:"flex",flexDirection:"column"}}>
      {/* Nav */}
      <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",padding:"0 48px",height:64,borderBottom:"1px solid rgba(255,255,255,.07)"}}>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <span style={{fontSize:22}}>🐴</span>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:14,fontWeight:600,letterSpacing:".22em",textTransform:"uppercase",color:"#c9a84c"}}>Paddock</div>
        </div>
        <div style={{display:"flex",gap:12,alignItems:"center"}}>
          <button className="gh" onClick={onLogin}>Sign In</button>
          <button className="bg" onClick={onGetStarted}>Start Free Trial</button>
        </div>
      </div>

      {/* Hero */}
      <div style={{flex:1,display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",padding:"80px 24px 40px",textAlign:"center"}}>
        <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,letterSpacing:".25em",textTransform:"uppercase",color:"#c9a84c",marginBottom:16}}>Barn Management Software</div>
        <div style={{fontSize:"clamp(36px,6vw,72px)",fontStyle:"italic",fontWeight:300,lineHeight:1.1,marginBottom:20,maxWidth:700}}>
          Everything your barn needs,<br/>in one elegant place.
        </div>
        <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:14,color:"rgba(232,224,208,.5)",maxWidth:520,lineHeight:1.7,marginBottom:40}}>
          Track horse health, schedule vet & farrier visits, manage medications, feeding schedules, and your entire barn team — all from one beautiful dashboard.
        </div>
        <div style={{display:"flex",gap:12,flexWrap:"wrap",justifyContent:"center"}}>
          <button className="bg" style={{fontSize:13,padding:"13px 28px"}} onClick={onGetStarted}>Start 14-Day Free Trial</button>
          <button className="bo" style={{fontSize:13,padding:"13px 28px"}} onClick={onLogin}>Sign In to Your Barn</button>
        </div>
        <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.3)",marginTop:16,letterSpacing:".05em"}}>No credit card required · Cancel anytime</div>

        {/* Feature pills */}
        <div style={{display:"flex",gap:10,flexWrap:"wrap",justifyContent:"center",marginTop:48}}>
          {["🩺 Health Tracking","⚖️ Weight Charts","💊 Medications","📅 Appointments","🌾 Feeding Schedules","👥 Staff Management","⬇ PDF Export"].map(f=>(
            <span key={f} style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,padding:"6px 14px",borderRadius:20,border:"1px solid rgba(255,255,255,.1)",color:"rgba(232,224,208,.6)",background:"rgba(255,255,255,.03)"}}>{f}</span>
          ))}
        </div>
      </div>

      {/* Pricing */}
      <div style={{padding:"60px 48px",background:"rgba(255,255,255,.02)",borderTop:"1px solid rgba(255,255,255,.06)"}}>
        <div style={{textAlign:"center",marginBottom:36}}>
          <div style={{fontSize:32,fontStyle:"italic",fontWeight:300,marginBottom:6}}>Simple, transparent pricing</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(232,224,208,.4)",letterSpacing:".1em"}}>Start free, scale as your barn grows</div>
        </div>
        <div style={{display:"grid",gridTemplateColumns:"repeat(3,1fr)",gap:16,maxWidth:860,margin:"0 auto"}}>
          {PLANS.map(p=>(
            <div key={p.id} className={`plan-card${p.popular?" popular":""}`} style={{background:p.popular?"rgba(125,211,252,.04)":"rgba(255,255,255,.02)"}}>
              {p.popular && <div style={{position:"absolute",top:-12,left:"50%",transform:"translateX(-50%)",fontFamily:"'Josefin Sans',sans-serif",fontSize:9,fontWeight:600,letterSpacing:".12em",textTransform:"uppercase",background:"#7dd3fc",color:"#0a1a2a",padding:"3px 12px",borderRadius:20}}>Most Popular</div>}
              <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".18em",textTransform:"uppercase",color:p.color,marginBottom:10}}>{p.name}</div>
              <div style={{fontSize:36,fontWeight:300,color:"#e8e0d0"}}>{p.price}<span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(232,224,208,.4)"}}>{p.period}</span></div>
              <div style={{marginTop:20,display:"flex",flexDirection:"column",gap:8}}>
                {p.features.map(f=>(
                  <div key={f} style={{display:"flex",gap:8,alignItems:"center"}}>
                    <span style={{color:p.color,fontSize:12}}>✓</span>
                    <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.7)"}}>{f}</span>
                  </div>
                ))}
              </div>
              <button className="bg" style={{width:"100%",marginTop:24,background:p.popular?"linear-gradient(135deg,#7dd3fc,#a5e3ff)":"linear-gradient(135deg,#c9a84c,#e8c96d)",fontSize:11}} onClick={onGetStarted}>Get Started</button>
            </div>
          ))}
        </div>
      </div>

      <div style={{textAlign:"center",padding:"24px",borderTop:"1px solid rgba(255,255,255,.06)",fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.2)",letterSpacing:".1em"}}>
        © 2026 PADDOCK · BARN HEALTH MANAGEMENT · ALL RIGHTS RESERVED
      </div>
    </div>
  );
}

/* ── Login ── */
function Login({ onLogin, onBack, onGetStarted }) {
  const [email,setEmail] = useState("");
  const [pass,setPass]   = useState("");
  const [err,setErr]     = useState("");

  const attempt = () => {
    if(!email.trim()||!pass.trim()){ setErr("Please enter your email and password."); return; }
    // Demo: any credentials work
    onLogin({ name: email.split("@")[0], email, role:"Barn Owner" });
  };

  return (
    <div style={{minHeight:"100vh",background:"#0d0c0a",display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",padding:24}}>
      <div style={{cursor:"pointer",marginBottom:32,display:"flex",alignItems:"center",gap:10}} onClick={onBack}>
        <span style={{fontSize:22}}>🐴</span>
        <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:14,fontWeight:600,letterSpacing:".22em",textTransform:"uppercase",color:"#c9a84c"}}>Paddock</div>
      </div>
      <div style={{width:"100%",maxWidth:380,background:"rgba(255,255,255,.03)",border:"1px solid rgba(255,255,255,.08)",borderRadius:8,padding:"36px 32px"}}>
        <div style={{fontSize:24,fontStyle:"italic",fontWeight:300,color:"#e8e0d0",marginBottom:4}}>Welcome back</div>
        <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.35)",letterSpacing:".08em",marginBottom:28}}>Sign in to your barn dashboard</div>
        <div style={{marginBottom:14}}>
          <label className="lbl">Email</label>
          <input className="inp" placeholder="you@yourbarn.com" value={email} onChange={e=>setEmail(e.target.value)} onKeyDown={e=>e.key==="Enter"&&attempt()}/>
        </div>
        <div style={{marginBottom:20}}>
          <label className="lbl">Password</label>
          <input className="inp" type="password" placeholder="••••••••" value={pass} onChange={e=>setPass(e.target.value)} onKeyDown={e=>e.key==="Enter"&&attempt()}/>
        </div>
        {err && <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"#f87171",marginBottom:14}}>{err}</div>}
        <button className="bg" style={{width:"100%",fontSize:12,padding:"11px"}} onClick={attempt}>Sign In</button>
        <div style={{marginTop:16,textAlign:"center",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.3)"}}>
          No account? <span style={{color:"#c9a84c",cursor:"pointer"}} onClick={onGetStarted}>Start free trial</span>
        </div>
        <div style={{marginTop:8,textAlign:"center",fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.2)",letterSpacing:".05em"}}>Demo: enter any email & password</div>
      </div>
    </div>
  );
}

/* ── Onboarding Wizard ── */
function Onboarding({ onComplete }) {
  const [step, setStep] = useState(0);
  const [barn, setBarn] = useState({ name:"", location:"", type:"Private", plan:"pro" });
  const [owner, setOwner] = useState({ name:"", email:"", password:"", role:"Barn Owner" });
  const [staff, setStaff] = useState([]);
  const [newStaff, setNewStaff] = useState({ name:"", role:"Head Groom", email:"" });

  const steps = ["Welcome","Your Barn","Your Account","Your Team","Choose Plan","All Set!"];

  const addStaff = () => {
    if(!newStaff.name.trim()) return;
    setStaff(p=>[...p,{id:uid(),...newStaff}]);
    setNewStaff({name:"",role:"Head Groom",email:""});
  };

  const finish = () => {
    const allStaff = [
      {id:"owner",name:owner.name||"Barn Owner",role:"Barn Owner",email:owner.email,avatar:initials(owner.name||"BO")},
      ...staff.map(s=>({...s,avatar:initials(s.name)}))
    ];
    onComplete({ barn, owner, staff:allStaff, plan: barn.plan });
  };

  const bg = "linear-gradient(135deg,#f9f4ec 0%,#f2e8d5 100%)";
  const accent = "#8b6914";

  return (
    <div style={{minHeight:"100vh",background:bg,display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",padding:24,fontFamily:"'Cormorant Garamond',Georgia,serif"}}>
      <style>{CSS}</style>
      {/* Progress */}
      <div style={{display:"flex",gap:8,marginBottom:36,alignItems:"center"}}>
        {steps.map((s,i)=>(
          <React.Fragment key={s}>
            <div style={{display:"flex",flexDirection:"column",alignItems:"center",gap:5}}>
              <div style={{width:28,height:28,borderRadius:"50%",background:i<=step?"#c9a84c":"rgba(0,0,0,.1)",display:"flex",alignItems:"center",justifyContent:"center",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,color:i<=step?"#1a1208":"rgba(0,0,0,.3)",transition:"all .3s"}}>{i<step?"✓":i+1}</div>
              <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:8,letterSpacing:".1em",textTransform:"uppercase",color:i===step?accent:"rgba(0,0,0,.3)",whiteSpace:"nowrap"}}>{s}</div>
            </div>
            {i<steps.length-1&&<div style={{width:24,height:1,background:i<step?"#c9a84c":"rgba(0,0,0,.1)",marginBottom:14,transition:"all .3s"}}/>}
          </React.Fragment>
        ))}
      </div>

      <div style={{width:"100%",maxWidth:480,background:"#fff",borderRadius:12,padding:"40px 36px",boxShadow:"0 20px 60px rgba(0,0,0,.08)"}}>

        {/* Step 0: Welcome */}
        {step===0&&<div className="fi">
          <div style={{fontSize:36,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:8}}>Welcome to Paddock</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(42,26,8,.5)",lineHeight:1.7,marginBottom:28}}>
            The all-in-one barn management platform built for equestrian professionals. We'll get your barn set up in under 3 minutes.
          </div>
          <div style={{display:"flex",flexDirection:"column",gap:12,marginBottom:28}}>
            {["Set up your barn profile & branding","Add your horses and health records","Invite your team","Schedule vet & farrier appointments"].map((f,i)=>(
              <div key={f} style={{display:"flex",gap:12,alignItems:"center"}}>
                <div style={{width:24,height:24,borderRadius:"50%",background:"rgba(201,168,76,.15)",display:"flex",alignItems:"center",justifyContent:"center",fontFamily:"'Josefin Sans',sans-serif",fontSize:10,fontWeight:600,color:"#8b6914",flexShrink:0}}>{i+1}</div>
                <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(42,26,8,.65)"}}>{f}</span>
              </div>
            ))}
          </div>
          <button style={{background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"12px 24px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:12,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer",width:"100%"}} onClick={()=>setStep(1)}>Let's Get Started →</button>
        </div>}

        {/* Step 1: Barn Info */}
        {step===1&&<div className="fi">
          <div style={{fontSize:26,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:4}}>Your Barn</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(42,26,8,.45)",marginBottom:24}}>Tell us about your operation</div>
          <div style={{display:"flex",flexDirection:"column",gap:14}}>
            <div><label className="lbl-light">Barn Name *</label><input className="wizard-inp" placeholder="e.g. Sunridge Equestrian" value={barn.name} onChange={e=>setBarn(p=>({...p,name:e.target.value}))}/></div>
            <div><label className="lbl-light">Location</label><input className="wizard-inp" placeholder="e.g. Lexington, Kentucky" value={barn.location} onChange={e=>setBarn(p=>({...p,location:e.target.value}))}/></div>
            <div><label className="lbl-light">Barn Type</label>
              <select className="wizard-sel" value={barn.type} onChange={e=>setBarn(p=>({...p,type:e.target.value}))}>
                {["Private","Boarding","Training","Show","Breeding","Rescue"].map(t=><option key={t}>{t}</option>)}
              </select>
            </div>
          </div>
          <div style={{display:"flex",gap:10,marginTop:24}}>
            <button style={{flex:1,background:"rgba(0,0,0,.06)",color:"rgba(42,26,8,.5)",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(0)}>Back</button>
            <button style={{flex:2,background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>barn.name.trim()&&setStep(2)}>Continue →</button>
          </div>
        </div>}

        {/* Step 2: Owner Account */}
        {step===2&&<div className="fi">
          <div style={{fontSize:26,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:4}}>Your Account</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(42,26,8,.45)",marginBottom:24}}>Create your barn owner login</div>
          <div style={{display:"flex",flexDirection:"column",gap:14}}>
            <div><label className="lbl-light">Your Name *</label><input className="wizard-inp" placeholder="e.g. Sarah Mitchell" value={owner.name} onChange={e=>setOwner(p=>({...p,name:e.target.value}))}/></div>
            <div><label className="lbl-light">Email *</label><input className="wizard-inp" placeholder="sarah@sunridgebarn.com" value={owner.email} onChange={e=>setOwner(p=>({...p,email:e.target.value}))}/></div>
            <div><label className="lbl-light">Password *</label><input className="wizard-inp" type="password" placeholder="Choose a secure password" value={owner.password} onChange={e=>setOwner(p=>({...p,password:e.target.value}))}/></div>
          </div>
          <div style={{display:"flex",gap:10,marginTop:24}}>
            <button style={{flex:1,background:"rgba(0,0,0,.06)",color:"rgba(42,26,8,.5)",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(1)}>Back</button>
            <button style={{flex:2,background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>owner.name.trim()&&owner.email.trim()&&setStep(3)}>Continue →</button>
          </div>
        </div>}

        {/* Step 3: Staff */}
        {step===3&&<div className="fi">
          <div style={{fontSize:26,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:4}}>Your Team</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(42,26,8,.45)",marginBottom:20}}>Add staff members (you can do this later too)</div>
          <div style={{display:"flex",gap:8,marginBottom:12}}>
            <input className="wizard-inp" placeholder="Staff name" style={{flex:2}} value={newStaff.name} onChange={e=>setNewStaff(p=>({...p,name:e.target.value}))}/>
            <select className="wizard-sel" style={{flex:1}} value={newStaff.role} onChange={e=>setNewStaff(p=>({...p,role:e.target.value}))}>
              {ROLES.filter(r=>r!=="Barn Owner").map(r=><option key={r}>{r}</option>)}
            </select>
          </div>
          <button style={{background:"rgba(201,168,76,.15)",color:"#8b6914",padding:"8px 16px",borderRadius:3,border:"1px solid rgba(201,168,76,.3)",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer",marginBottom:16,width:"100%"}} onClick={addStaff}>+ Add Staff Member</button>
          {staff.length>0&&<div style={{display:"flex",flexDirection:"column",gap:6,marginBottom:16}}>
            {staff.map(s=>(
              <div key={s.id} style={{display:"flex",alignItems:"center",gap:10,padding:"8px 12px",background:"rgba(201,168,76,.07)",borderRadius:3,border:"1px solid rgba(201,168,76,.15)"}}>
                <div style={{width:28,height:28,borderRadius:"50%",background:avatarColor(s.name),display:"flex",alignItems:"center",justifyContent:"center",fontSize:10,fontWeight:600,color:"#fff",fontFamily:"'Josefin Sans',sans-serif"}}>{initials(s.name)}</div>
                <div style={{flex:1}}><div style={{fontSize:14,fontWeight:300,color:"#2a1a08"}}>{s.name}</div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(42,26,8,.45)"}}>{s.role}</div></div>
                <button style={{background:"none",border:"none",color:"rgba(42,26,8,.3)",cursor:"pointer",fontSize:14}} onClick={()=>setStaff(p=>p.filter(x=>x.id!==s.id))}>✕</button>
              </div>
            ))}
          </div>}
          <div style={{display:"flex",gap:10,marginTop:8}}>
            <button style={{flex:1,background:"rgba(0,0,0,.06)",color:"rgba(42,26,8,.5)",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(2)}>Back</button>
            <button style={{flex:2,background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(4)}>Continue →</button>
          </div>
        </div>}

        {/* Step 4: Plan */}
        {step===4&&<div className="fi">
          <div style={{fontSize:26,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:4}}>Choose Your Plan</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(42,26,8,.45)",marginBottom:20}}>14-day free trial on all plans · No credit card needed</div>
          <div style={{display:"flex",flexDirection:"column",gap:10}}>
            {PLANS.map(p=>(
              <div key={p.id} onClick={()=>setBarn(x=>({...x,plan:p.id}))} style={{padding:"14px 16px",borderRadius:6,border:`2px solid ${barn.plan===p.id?p.color:"rgba(0,0,0,.1)"}`,background:barn.plan===p.id?`${p.color}10`:"transparent",cursor:"pointer",transition:"all .2s"}}>
                <div style={{display:"flex",alignItems:"center",justifyContent:"space-between"}}>
                  <div>
                    <div style={{display:"flex",alignItems:"center",gap:8}}>
                      <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,fontWeight:600,letterSpacing:".12em",textTransform:"uppercase",color:p.color}}>{p.name}</span>
                      {p.popular&&<span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:8,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",background:p.color,color:"#fff",padding:"2px 7px",borderRadius:10}}>Popular</span>}
                    </div>
                    <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(42,26,8,.45)",marginTop:2}}>{p.features.slice(0,2).join(" · ")}</div>
                  </div>
                  <div style={{textAlign:"right"}}>
                    <span style={{fontSize:22,fontWeight:300,color:"#2a1a08"}}>{p.price}</span>
                    <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(42,26,8,.4)"}}>/mo</span>
                  </div>
                </div>
              </div>
            ))}
          </div>
          <div style={{display:"flex",gap:10,marginTop:20}}>
            <button style={{flex:1,background:"rgba(0,0,0,.06)",color:"rgba(42,26,8,.5)",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(3)}>Back</button>
            <button style={{flex:2,background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"11px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer"}} onClick={()=>setStep(5)}>Continue →</button>
          </div>
        </div>}

        {/* Step 5: Done */}
        {step===5&&<div className="fi" style={{textAlign:"center"}}>
          <div style={{fontSize:48,marginBottom:12}}>🎉</div>
          <div style={{fontSize:28,fontStyle:"italic",fontWeight:300,color:"#2a1a08",marginBottom:8}}>{barn.name} is ready!</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(42,26,8,.5)",lineHeight:1.7,marginBottom:28}}>
            Your barn dashboard is set up and ready to go. Start by adding your horses and their health records.
          </div>
          <div style={{background:"rgba(201,168,76,.08)",border:"1px solid rgba(201,168,76,.2)",borderRadius:6,padding:"16px",marginBottom:24,textAlign:"left"}}>
            <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,letterSpacing:".18em",textTransform:"uppercase",color:"#8b6914",marginBottom:10}}>Your Setup</div>
            {[["Barn",barn.name],["Location",barn.location||"—"],["Plan",PLANS.find(p=>p.id===barn.plan)?.name],["Team",`${staff.length+1} member${staff.length!==0?"s":""}`]].map(([k,v])=>(
              <div key={k} style={{display:"flex",justifyContent:"space-between",padding:"5px 0",borderBottom:"1px solid rgba(201,168,76,.1)"}}>
                <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(42,26,8,.45)"}}>{k}</span>
                <span style={{fontSize:14,fontWeight:300,color:"#2a1a08"}}>{v}</span>
              </div>
            ))}
          </div>
          <button style={{background:"linear-gradient(135deg,#c9a84c,#e8c96d)",color:"#1a1208",padding:"13px 24px",borderRadius:4,border:"none",fontFamily:"'Josefin Sans',sans-serif",fontSize:12,fontWeight:600,letterSpacing:".1em",textTransform:"uppercase",cursor:"pointer",width:"100%"}} onClick={finish}>Enter My Barn Dashboard →</button>
        </div>}
      </div>
    </div>
  );
}

/* ─── Settings Panel ─────────────────────────────────────────────────────────── */
function Settings({ appState, onSave, onBack }) {
  const [barn, setBarn]   = useState({...appState.barn});
  const [staff, setStaff] = useState([...appState.staff]);
  const [newS, setNewS]   = useState({name:"",role:"Head Groom",email:""});
  const [tab, setTab]     = useState("barn");

  const addS = () => {
    if(!newS.name.trim()) return;
    setStaff(p=>[...p,{id:uid(),...newS,avatar:initials(newS.name)}]);
    setNewS({name:"",role:"Head Groom",email:""});
  };

  return (
    <div style={{minHeight:"100vh",background:"#0d0c0a",color:"#e8e0d0"}}>
      <style>{CSS}</style>
      <div style={{borderBottom:"1px solid rgba(255,255,255,.07)",padding:"0 24px",display:"flex",alignItems:"center",justifyContent:"space-between",height:54,background:"#0f0d0b"}}>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <button className="gh" style={{fontSize:9,padding:"4px 9px"}} onClick={onBack}>← Back</button>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,color:"rgba(232,224,208,.5)",letterSpacing:".1em"}}>Barn Settings</div>
        </div>
        <button className="bg" style={{fontSize:10,padding:"6px 14px"}} onClick={()=>onSave({barn,staff})}>Save Changes</button>
      </div>
      <div style={{maxWidth:600,margin:"0 auto",padding:"32px 24px"}}>
        <div style={{display:"flex",gap:4,marginBottom:24}}>
          {[["barn","🏠 Barn Profile"],["team","👥 Team"],["plan","💳 Plan"]].map(([k,l])=>(
            <button key={k} className={`gh${tab===k?" act":""}`} onClick={()=>setTab(k)}>{l}</button>
          ))}
        </div>

        {tab==="barn"&&<div className="fi">
          <div style={{fontSize:22,fontStyle:"italic",fontWeight:300,marginBottom:18}}>Barn Profile</div>
          <div style={{display:"flex",flexDirection:"column",gap:14}}>
            <div><label className="lbl">Barn Name</label><input className="inp" value={barn.name} onChange={e=>setBarn(p=>({...p,name:e.target.value}))}/></div>
            <div><label className="lbl">Location</label><input className="inp" value={barn.location||""} onChange={e=>setBarn(p=>({...p,location:e.target.value}))}/></div>
            <div><label className="lbl">Barn Type</label>
              <select className="sel2" value={barn.type||"Private"} onChange={e=>setBarn(p=>({...p,type:e.target.value}))}>
                {["Private","Boarding","Training","Show","Breeding","Rescue"].map(t=><option key={t}>{t}</option>)}
              </select>
            </div>
          </div>
        </div>}

        {tab==="team"&&<div className="fi">
          <div style={{fontSize:22,fontStyle:"italic",fontWeight:300,marginBottom:18}}>Team Members</div>
          <div style={{display:"flex",gap:8,marginBottom:12}}>
            <input className="inp" placeholder="Name" style={{flex:2}} value={newS.name} onChange={e=>setNewS(p=>({...p,name:e.target.value}))}/>
            <select className="sel2" style={{flex:1}} value={newS.role} onChange={e=>setNewS(p=>({...p,role:e.target.value}))}>
              {ROLES.map(r=><option key={r}>{r}</option>)}
            </select>
          </div>
          <input className="inp" placeholder="Email (optional)" style={{marginBottom:10}} value={newS.email} onChange={e=>setNewS(p=>({...p,email:e.target.value}))}/>
          <button className="bo" style={{width:"100%",marginBottom:18,fontSize:10}} onClick={addS}>+ Add Team Member</button>
          {staff.map(s=>(
            <div key={s.id} style={{display:"flex",alignItems:"center",gap:12,padding:"10px 14px",background:"rgba(255,255,255,.025)",borderRadius:4,border:"1px solid rgba(255,255,255,.07)",marginBottom:7}}>
              <div style={{width:32,height:32,borderRadius:"50%",background:avatarColor(s.name),display:"flex",alignItems:"center",justifyContent:"center",fontSize:11,fontWeight:600,color:"#fff",fontFamily:"'Josefin Sans',sans-serif",flexShrink:0}}>{s.avatar||initials(s.name)}</div>
              <div style={{flex:1}}>
                <div style={{fontSize:14,fontWeight:300}}>{s.name}</div>
                <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.38)"}}>{s.role}{s.email&&` · ${s.email}`}</div>
              </div>
              {s.id!=="owner"&&<button className="gh" style={{fontSize:9}} onClick={()=>setStaff(p=>p.filter(x=>x.id!==s.id))}>Remove</button>}
            </div>
          ))}
        </div>}

        {tab==="plan"&&<div className="fi">
          <div style={{fontSize:22,fontStyle:"italic",fontWeight:300,marginBottom:6}}>Your Plan</div>
          <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.4)",marginBottom:20}}>Current: <span style={{color:"#c9a84c"}}>{PLANS.find(p=>p.id===barn.plan)?.name||"Pro"}</span></div>
          {PLANS.map(p=>(
            <div key={p.id} onClick={()=>setBarn(x=>({...x,plan:p.id}))} style={{padding:"14px 16px",borderRadius:6,border:`2px solid ${barn.plan===p.id?p.color:"rgba(255,255,255,.08)"}`,background:barn.plan===p.id?`${p.color}08`:"transparent",cursor:"pointer",transition:"all .2s",marginBottom:10}}>
              <div style={{display:"flex",alignItems:"center",justifyContent:"space-between"}}>
                <div>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,fontWeight:600,letterSpacing:".12em",textTransform:"uppercase",color:p.color}}>{p.name} {p.popular&&"⭐"}</div>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.4)",marginTop:2}}>{p.features.slice(0,2).join(" · ")}</div>
                </div>
                <div style={{fontSize:20,fontWeight:300,color:"#e8e0d0"}}>{p.price}<span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.35)"}}>/mo</span></div>
              </div>
            </div>
          ))}
        </div>}
      </div>
    </div>
  );
}

/* ─── Main Dashboard ─────────────────────────────────────────────────────────── */
function Dashboard({ appState, onUpdateApp, onSettings, onLogout }) {
  const { barn, staff: staffList, horses: initHorses } = appState;

  const [horses, setHorses] = useState(initHorses||[]);
  const [sel, setSel]       = useState(null);
  const [view, setView]     = useState("overview");
  const [tab, setTab]       = useState("Overview");
  const [filter, setFilter] = useState("all");

  const [showLog,  setShowLog]  = useState(false);
  const [showMed,  setShowMed]  = useState(false);
  const [showWt,   setShowWt]   = useState(false);
  const [showAppt, setShowAppt] = useState(false);
  const [showFeed, setShowFeed] = useState(false);

  const eLog  = {type:"health",note:"",by:""};
  const eMed  = {name:"",dose:"",frequency:"",startDate:today(),endDate:"",notes:"",active:true};
  const eWt   = {weight:"",date:today()};
  const eAppt = {type:"vet",title:"",date:"",time:"",provider:"",notes:""};
  const eFeed = {time:"",label:"",amount:""};
  const eAdd  = {name:"",breed:"",age:"",stall:"",feed:"",notes:"",status:"healthy",weight:""};

  const [logF,  setLogF]  = useState(eLog);
  const [medF,  setMedF]  = useState(eMed);
  const [wtF,   setWtF]   = useState(eWt);
  const [apptF, setApptF] = useState(eAppt);
  const [feedF, setFeedF] = useState(eFeed);
  const [addF,  setAddF]  = useState(eAdd);

  // Persist whenever horses change
  useEffect(() => { onUpdateApp({ horses }); }, [horses]);

  const filtered    = filter==="all" ? horses : horses.filter(h=>h.status===filter);
  const attnCount   = horses.filter(h=>h.status==="attention").length;
  const critCount   = horses.filter(h=>h.status==="critical").length;
  const allUpcoming = horses.flatMap(h=>(h.appointments||[]).filter(a=>a.status==="scheduled").map(a=>({...a,horseName:h.name}))).sort((a,b)=>a.date.localeCompare(b.date));

  const upd = (id,patch) => {
    setHorses(p=>p.map(h=>h.id===id?{...h,...patch}:h));
    if(sel?.id===id) setSel(s=>({...s,...patch}));
  };
  const pick = h => { setSel(h); setView("detail"); setTab("Overview"); setShowLog(false); setShowMed(false); setShowWt(false); setShowAppt(false); setShowFeed(false); };

  const saveLog  = () => { if(!logF.note.trim())return; upd(sel.id,{logs:[{date:today(),...logF},...(sel.logs||[])]}); setLogF(eLog); setShowLog(false); };
  const saveMed  = () => { if(!medF.name.trim())return; upd(sel.id,{medications:[...(sel.medications||[]),{id:uid(),...medF}]}); setMedF(eMed); setShowMed(false); };
  const saveWt   = () => {
    if(!wtF.weight)return;
    const e={date:wtF.date,weight:Number(wtF.weight)};
    const wh=[...(sel.weightHistory||[]),e].sort((a,b)=>a.date.localeCompare(b.date));
    const logE={date:today(),type:"weight",note:`Weight: ${e.weight} lbs`,by:"Barn Staff"};
    upd(sel.id,{weightHistory:wh,weight:e.weight,logs:[logE,...(sel.logs||[])]});
    setWtF(eWt); setShowWt(false);
  };
  const saveAppt = () => {
    if(!apptF.title.trim()||!apptF.date)return;
    upd(sel.id,{appointments:[...(sel.appointments||[]),{id:uid(),...apptF,status:"scheduled"}]});
    if(apptF.type==="farrier") upd(sel.id,{nextFarrier:apptF.date});
    if(apptF.type==="vet")     upd(sel.id,{lastVet:apptF.date});
    setApptF(eAppt); setShowAppt(false);
  };
  const completeAppt = aid => {
    const appt=(sel.appointments||[]).find(a=>a.id===aid);
    upd(sel.id,{appointments:(sel.appointments||[]).map(a=>a.id===aid?{...a,status:"completed"}:a),logs:[{date:today(),type:"health",note:`Appt completed: ${appt?.title}`,by:"Barn Staff"},...(sel.logs||[])]});
  };
  const saveFeed   = () => { if(!feedF.label.trim()||!feedF.time)return; upd(sel.id,{feedingReminders:[...(sel.feedingReminders||[]),{id:uid(),...feedF,active:true}]}); setFeedF(eFeed); setShowFeed(false); };
  const toggleFeed = rid => upd(sel.id,{feedingReminders:(sel.feedingReminders||[]).map(r=>r.id===rid?{...r,active:!r.active}:r)});
  const deleteFeed = rid => upd(sel.id,{feedingReminders:(sel.feedingReminders||[]).filter(r=>r.id!==rid)});
  const toggleMed  = mid => upd(sel.id,{medications:(sel.medications||[]).map(m=>m.id===mid?{...m,active:!m.active}:m)});
  const toggleStaff= sid => { const c=sel.assignedStaff||[]; upd(sel.id,{assignedStaff:c.includes(sid)?c.filter(s=>s!==sid):[...c,sid]}); };
  const setStatus  = s   => upd(sel.id,{status:s});
  const saveHorse  = () => {
    if(!addF.name.trim())return;
    setHorses(p=>[...p,{id:Date.now(),...addF,age:+addF.age||0,weight:+addF.weight||0,avatar:"🐴",color:"#6b7280",lastVet:"—",nextFarrier:"—",nextVaccine:"—",logs:[],medications:[],appointments:[],feedingReminders:[],weightHistory:addF.weight?[{date:today(),weight:+addF.weight}]:[],assignedStaff:[]}]);
    setAddF(eAdd); setView("overview");
  };

  const weightDelta = useMemo(()=>{
    if(!sel?.weightHistory||sel.weightHistory.length<2)return null;
    const wh=sel.weightHistory; return wh[wh.length-1].weight-wh[0].weight;
  },[sel]);

  const G2 = ({children,style={}}) => <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:12,...style}}>{children}</div>;
  const FormBox = ({children,onClose}) => (
    <div className="fi card" style={{border:"1px solid rgba(201,168,76,.2)",marginBottom:16,background:"rgba(201,168,76,.03)"}}>
      {children}
      <div style={{marginTop:12}}><button className="gh" style={{fontSize:9}} onClick={onClose}>Cancel</button></div>
    </div>
  );
  const SH = ({title,btn,onBtn}) => (
    <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",marginBottom:14}}>
      <div style={{fontSize:19,fontStyle:"italic",fontWeight:300}}>{title}</div>
      {btn&&<button className="bo" style={{fontSize:10,padding:"5px 11px"}} onClick={onBtn}>{btn}</button>}
    </div>
  );

  const planInfo = PLANS.find(p=>p.id===(barn.plan||"pro"))||PLANS[1];

  return (
    <div style={{fontFamily:"'Cormorant Garamond',Georgia,serif",minHeight:"100vh",background:"#0d0c0a",color:"#e8e0d0"}}>
      <style>{CSS}</style>

      {/* Header */}
      <div style={{borderBottom:"1px solid rgba(255,255,255,.07)",padding:"0 20px",display:"flex",alignItems:"center",justifyContent:"space-between",height:52,background:"#0f0d0b"}}>
        <div style={{display:"flex",alignItems:"center",gap:10,cursor:"pointer"}} onClick={()=>setView("overview")}>
          <span style={{fontSize:18}}>🐴</span>
          <div>
            <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,fontWeight:600,letterSpacing:".18em",textTransform:"uppercase",color:"#c9a84c"}}>{barn.name||"Paddock"}</div>
            <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:8,letterSpacing:".15em",color:"rgba(232,224,208,.28)",textTransform:"uppercase"}}>{barn.type||"Barn"} · {planInfo.name} Plan</div>
          </div>
        </div>
        <div style={{display:"flex",gap:6,alignItems:"center"}}>
          {attnCount>0&&<span className="pill" style={{background:"rgba(245,158,11,.18)",color:"#f59e0b"}}>⚠ {attnCount}</span>}
          {critCount>0&&<span className="pill" style={{background:"rgba(248,113,113,.18)",color:"#f87171"}}>🚨 {critCount}</span>}
          {sel&&view==="detail"&&<button className="bo" style={{fontSize:9,padding:"4px 10px"}} onClick={()=>exportPDF(sel,barn.name||"Paddock")}>⬇ PDF</button>}
          <button className="bg" style={{fontSize:9,padding:"5px 12px"}} onClick={()=>setView("addHorse")}>+ Horse</button>
          <button className="gh" style={{fontSize:9,padding:"5px 10px"}} onClick={onSettings}>⚙ Settings</button>
          <button className="gh" style={{fontSize:9,padding:"5px 10px"}} onClick={onLogout}>Sign Out</button>
        </div>
      </div>

      <div style={{display:"flex",height:"calc(100vh - 52px)"}}>
        {/* Sidebar */}
        <div style={{width:244,borderRight:"1px solid rgba(255,255,255,.06)",overflowY:"auto",padding:"13px 10px",flexShrink:0,background:"#0f0d0b"}}>
          <div className="cap" style={{marginBottom:7,padding:"0 4px"}}>Filter</div>
          <div style={{display:"flex",gap:3,marginBottom:13,flexWrap:"wrap"}}>
            {["all","healthy","attention","critical"].map(s=>(
              <button key={s} className={`gh${filter===s?" act":""}`} style={{fontSize:9,padding:"3px 7px"}} onClick={()=>setFilter(s)}>
                {s==="all"?"All":STATUS[s].label}
              </button>
            ))}
          </div>
          <div className="cap" style={{marginBottom:7,padding:"0 4px"}}>{filtered.length} horses</div>
          {filtered.length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.25)",padding:"12px 4px",lineHeight:1.6}}>No horses yet.<br/><span style={{color:"#c9a84c",cursor:"pointer"}} onClick={()=>setView("addHorse")}>Add your first horse →</span></div>}
          {filtered.map(h=>(
            <div key={h.id} className={`hcard${sel?.id===h.id&&view==="detail"?" on":""}`}
              style={{padding:"10px",marginBottom:5,background:"rgba(255,255,255,.015)"}} onClick={()=>pick(h)}>
              <div style={{display:"flex",alignItems:"center",justifyContent:"space-between"}}>
                <div style={{display:"flex",alignItems:"center",gap:7}}>
                  <div style={{width:28,height:28,borderRadius:"50%",background:h.color+"28",border:`2px solid ${h.color}55`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:13}}>{h.avatar}</div>
                  <div>
                    <div style={{fontSize:13,fontWeight:600}}>{h.name}</div>
                    <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.38)"}}>{h.breed||"—"}</div>
                  </div>
                </div>
                <div style={{width:6,height:6,borderRadius:"50%",background:STATUS[h.status]?.color,flexShrink:0}}/>
              </div>
              <div style={{marginTop:6,display:"flex",gap:7}}>
                <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.3)"}}>Stall {h.stall||"—"} · {h.age||"?"}yr</span>
                {(h.medications||[]).filter(m=>m.active).length>0&&<span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(74,222,128,.5)"}}>💊{h.medications.filter(m=>m.active).length}</span>}
                {(h.appointments||[]).filter(a=>a.status==="scheduled").length>0&&<span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(125,211,252,.5)"}}>📅{h.appointments.filter(a=>a.status==="scheduled").length}</span>}
              </div>
            </div>
          ))}
        </div>

        {/* Main */}
        <div style={{flex:1,overflowY:"auto",padding:"22px 28px"}}>

          {/* OVERVIEW */}
          {view==="overview"&&<div className="fi">
            <div style={{fontSize:24,fontStyle:"italic",fontWeight:300,marginBottom:2}}>Good morning,</div>
            <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.35)",letterSpacing:".1em",marginBottom:20}}>{barn.name} · {new Date().toLocaleDateString("en-US",{weekday:"long",month:"long",day:"numeric"})}</div>
            <div style={{display:"grid",gridTemplateColumns:"repeat(4,1fr)",gap:9,marginBottom:20}}>
              {[{l:"Horses",v:horses.length,i:"🐴"},{l:"Healthy",v:horses.filter(h=>h.status==="healthy").length,i:"✅",c:"#4ade80"},{l:"Appts",v:allUpcoming.length,i:"📅",c:"#7dd3fc"},{l:"Active Meds",v:horses.reduce((a,h)=>a+(h.medications||[]).filter(m=>m.active).length,0),i:"💊",c:"#a78bfa"}].map(s=>(
                <div key={s.l} className="card">
                  <div style={{fontSize:17}}>{s.i}</div>
                  <div style={{fontSize:20,fontWeight:300,color:s.c||"#e8e0d0",marginTop:4}}>{s.v}</div>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.3)",marginTop:2}}>{s.l}</div>
                </div>
              ))}
            </div>
            <div style={{fontSize:17,fontStyle:"italic",fontWeight:300,marginBottom:10}}>Upcoming Appointments</div>
            {allUpcoming.length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.28)",marginBottom:18}}>No upcoming appointments scheduled.</div>}
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:20}}>
              {allUpcoming.slice(0,4).map(a=>(
                <div key={a.id} className={`appt-card appt-${a.type}`} style={{border:"1px solid rgba(255,255,255,.07)"}}>
                  <div style={{display:"flex",gap:7,alignItems:"center",marginBottom:3}}>
                    <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:a.type==="vet"?"#7dd3fc":"#c9a84c",letterSpacing:".08em",textTransform:"uppercase"}}>{a.type==="vet"?"🩺 Vet":"🔨 Farrier"}</span>
                  </div>
                  <div style={{fontSize:13,fontWeight:400}}>{a.title}</div>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.4)",marginTop:2}}>{a.horseName} · {a.date}{a.time&&" at "+a.time}</div>
                </div>
              ))}
            </div>
            <div style={{fontSize:17,fontStyle:"italic",fontWeight:300,marginBottom:10}}>Today's Feeding</div>
            {horses.flatMap(h=>(h.feedingReminders||[]).filter(r=>r.active).map(r=>({...r,horseName:h.name}))).sort((a,b)=>a.time.localeCompare(b.time)).slice(0,6).map((r,i)=>(
              <div key={i} style={{display:"flex",alignItems:"center",gap:12,padding:"8px 0",borderBottom:"1px solid rgba(255,255,255,.05)"}}>
                <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:12,fontWeight:600,color:"#c9a84c",minWidth:46}}>{r.time}</div>
                <div style={{flex:1,fontSize:13,fontWeight:300}}>{r.label} <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.35)"}}>{r.horseName}</span></div>
                <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.38)"}}>{r.amount}</div>
              </div>
            ))}
            {horses.length===0&&<div style={{marginTop:24,padding:"28px",background:"rgba(201,168,76,.05)",border:"1px dashed rgba(201,168,76,.2)",borderRadius:6,textAlign:"center"}}>
              <div style={{fontSize:24,marginBottom:8}}>🐴</div>
              <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.4)",marginBottom:12}}>No horses added yet</div>
              <button className="bg" style={{fontSize:10}} onClick={()=>setView("addHorse")}>Add Your First Horse</button>
            </div>}
          </div>}

          {/* DETAIL */}
          {view==="detail"&&sel&&<div className="fi">
            <div style={{display:"flex",alignItems:"flex-start",justifyContent:"space-between",marginBottom:16}}>
              <div style={{display:"flex",gap:13,alignItems:"center"}}>
                <div style={{width:54,height:54,borderRadius:"50%",background:sel.color+"28",border:`2px solid ${sel.color}66`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:26}}>{sel.avatar}</div>
                <div>
                  <div style={{fontSize:24,fontStyle:"italic",fontWeight:300}}>{sel.name}</div>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.38)",marginTop:2}}>{sel.breed} · {sel.age}yr · Stall {sel.stall}</div>
                  <span className="pill" style={{marginTop:5,display:"inline-block",background:STATUS[sel.status].bg,color:STATUS[sel.status].color}}>● {STATUS[sel.status].label}</span>
                </div>
              </div>
              <div style={{display:"flex",gap:4,flexWrap:"wrap"}}>
                {Object.entries(STATUS).map(([k,v])=>(
                  <button key={k} className="gh" style={{fontSize:9,color:sel.status===k?v.color:undefined,borderColor:sel.status===k?v.color:undefined}} onClick={()=>setStatus(k)}>{v.label}</button>
                ))}
              </div>
            </div>
            <div style={{display:"flex",borderBottom:"1px solid rgba(255,255,255,.07)",marginBottom:18,overflowX:"auto"}}>
              {TABS.map(t=><button key={t} className={`dtab${tab===t?" on":""}`} onClick={()=>setTab(t)}>{t}</button>)}
            </div>

            {/* Overview */}
            {tab==="Overview"&&<div className="fi">
              <G2 style={{marginBottom:10}}>
                {[{l:"Weight",v:sel.weightHistory?.length?`${sel.weightHistory[sel.weightHistory.length-1].weight.toLocaleString()} lbs`:"—",i:"⚖️"},{l:"Last Vet",v:sel.lastVet||"—",i:"🩺"},{l:"Next Farrier",v:sel.nextFarrier||"—",i:"🔨"},{l:"Next Vaccine",v:sel.nextVaccine||"—",i:"💉"},{l:"Feed",v:sel.feed||"—",i:"🌾"},{l:"Active Meds",v:`${(sel.medications||[]).filter(m=>m.active).length}`,i:"💊"}].map(x=>(
                  <div key={x.l} className="card">
                    <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"#c9a84c",letterSpacing:".13em",textTransform:"uppercase",marginBottom:4}}>{x.i} {x.l}</div>
                    <div style={{fontSize:13,fontWeight:300,lineHeight:1.4}}>{x.v}</div>
                  </div>
                ))}
              </G2>
              {sel.notes&&<div className="card"><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"#c9a84c",letterSpacing:".13em",textTransform:"uppercase",marginBottom:5}}>📋 Notes</div><div style={{fontSize:13,fontWeight:300,lineHeight:1.5}}>{sel.notes}</div></div>}
            </div>}

            {/* Weight */}
            {tab==="Weight"&&<div className="fi">
              <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",marginBottom:14}}>
                <div><div style={{fontSize:19,fontStyle:"italic",fontWeight:300}}>Weight History</div>{weightDelta!==null&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,marginTop:2,color:weightDelta>=0?"#4ade80":"#f87171"}}>{weightDelta>=0?"▲":"▼"} {Math.abs(weightDelta)} lbs overall</div>}</div>
                <button className="bo" style={{fontSize:10,padding:"5px 11px"}} onClick={()=>setShowWt(!showWt)}>+ Log</button>
              </div>
              {showWt&&<FormBox onClose={()=>setShowWt(false)}><G2 style={{marginBottom:10}}><div><label className="lbl">Weight (lbs)</label><input className="inp" type="number" placeholder="1180" value={wtF.weight} onChange={e=>setWtF(p=>({...p,weight:e.target.value}))}/></div><div><label className="lbl">Date</label><input className="inp" type="date" value={wtF.date} onChange={e=>setWtF(p=>({...p,date:e.target.value}))}/></div></G2><button className="bg" style={{fontSize:10}} onClick={saveWt}>Save</button></FormBox>}
              {(sel.weightHistory||[]).length<2?<div className="card" style={{textAlign:"center",padding:"32px"}}><div style={{fontSize:24,marginBottom:7}}>⚖️</div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.3)"}}>Log 2+ entries to see chart</div></div>:
              <div className="card" style={{padding:"16px 6px 6px"}}>
                <ResponsiveContainer width="100%" height={190}>
                  <LineChart data={sel.weightHistory} margin={{top:4,right:16,left:0,bottom:4}}>
                    <CartesianGrid strokeDasharray="3 3" stroke="rgba(255,255,255,.05)"/>
                    <XAxis dataKey="date" tick={{fontFamily:"'Josefin Sans',sans-serif",fontSize:8,fill:"rgba(232,224,208,.3)"}} tickLine={false} axisLine={{stroke:"rgba(255,255,255,.08)"}}/>
                    <YAxis tick={{fontFamily:"'Josefin Sans',sans-serif",fontSize:8,fill:"rgba(232,224,208,.3)"}} tickLine={false} axisLine={false} domain={["auto","auto"]}/>
                    <Tooltip content={<ChartTip/>}/>
                    <Line type="monotone" dataKey="weight" stroke="#c9a84c" strokeWidth={2} dot={{r:3,fill:"#c9a84c",stroke:"#0d0c0a",strokeWidth:2}} activeDot={{r:5}}/>
                  </LineChart>
                </ResponsiveContainer>
              </div>}
              <div style={{marginTop:14}}>{[...(sel.weightHistory||[])].reverse().map((w,i)=>(
                <div key={i} style={{display:"flex",justifyContent:"space-between",padding:"7px 0",borderBottom:"1px solid rgba(255,255,255,.05)"}}>
                  <span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.38)"}}>{w.date}</span>
                  <span style={{fontSize:13,fontWeight:300}}>{w.weight.toLocaleString()} lbs</span>
                </div>
              ))}</div>
            </div>}

            {/* Medications */}
            {tab==="Medications"&&<div className="fi">
              <SH title="Medications" btn="+ Add" onBtn={()=>setShowMed(!showMed)}/>
              {showMed&&<FormBox onClose={()=>setShowMed(false)}>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:10}}>
                  <div style={{gridColumn:"1/-1"}}><label className="lbl">Name</label><input className="inp" placeholder="e.g. Bute" value={medF.name} onChange={e=>setMedF(p=>({...p,name:e.target.value}))}/></div>
                  <div><label className="lbl">Dose</label><input className="inp" placeholder="2g" value={medF.dose} onChange={e=>setMedF(p=>({...p,dose:e.target.value}))}/></div>
                  <div><label className="lbl">Frequency</label><input className="inp" placeholder="Once daily" value={medF.frequency} onChange={e=>setMedF(p=>({...p,frequency:e.target.value}))}/></div>
                  <div><label className="lbl">Start</label><input className="inp" type="date" value={medF.startDate} onChange={e=>setMedF(p=>({...p,startDate:e.target.value}))}/></div>
                  <div><label className="lbl">End (opt)</label><input className="inp" type="date" value={medF.endDate} onChange={e=>setMedF(p=>({...p,endDate:e.target.value}))}/></div>
                  <div style={{gridColumn:"1/-1"}}><label className="lbl">Notes</label><input className="inp" placeholder="Purpose, prescribing vet..." value={medF.notes} onChange={e=>setMedF(p=>({...p,notes:e.target.value}))}/></div>
                </div>
                <button className="bg" style={{fontSize:10}} onClick={saveMed}>Save</button>
              </FormBox>}
              {(sel.medications||[]).length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.28)"}}>No medications recorded.</div>}
              {(sel.medications||[]).filter(m=>m.active).length>0&&<><div className="cap" style={{marginBottom:8}}>Active</div>{(sel.medications||[]).filter(m=>m.active).map(m=>(
                <div key={m.id} className="card" style={{marginBottom:7}}>
                  <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start"}}>
                    <div style={{flex:1}}><div style={{display:"flex",gap:7,alignItems:"center",marginBottom:3}}><span style={{fontSize:13}}>💊 {m.name}</span><span className="pill" style={{background:"rgba(74,222,128,.12)",color:"#4ade80"}}>Active</span></div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.4)"}}>{m.dose} · {m.frequency} · {m.startDate}{m.endDate&&` → ${m.endDate}`}</div>{m.notes&&<div style={{fontSize:12,fontStyle:"italic",color:"rgba(232,224,208,.42)",marginTop:4}}>{m.notes}</div>}</div>
                    <button className="gh" style={{fontSize:9,marginLeft:8}} onClick={()=>toggleMed(m.id)}>Inactive</button>
                  </div>
                </div>
              ))}</>}
              {(sel.medications||[]).filter(m=>!m.active).length>0&&<><div className="cap" style={{marginBottom:8,marginTop:14,opacity:.5}}>Past</div>{(sel.medications||[]).filter(m=>!m.active).map(m=>(
                <div key={m.id} className="card" style={{marginBottom:7,opacity:.5}}>
                  <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                    <div><span style={{fontSize:13,fontWeight:300}}>💊 {m.name}</span><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.35)",marginTop:2}}>{m.dose} · {m.startDate}</div></div>
                    <button className="gh" style={{fontSize:9}} onClick={()=>toggleMed(m.id)}>Reactivate</button>
                  </div>
                </div>
              ))}</>}
            </div>}

            {/* Appointments */}
            {tab==="Appointments"&&<div className="fi">
              <SH title="Appointments" btn="+ Schedule" onBtn={()=>setShowAppt(!showAppt)}/>
              {showAppt&&<FormBox onClose={()=>setShowAppt(false)}>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:10}}>
                  <div><label className="lbl">Type</label><select className="sel2" value={apptF.type} onChange={e=>setApptF(p=>({...p,type:e.target.value}))}><option value="vet">🩺 Vet</option><option value="farrier">🔨 Farrier</option></select></div>
                  <div><label className="lbl">Title</label><input className="inp" placeholder="e.g. Annual Exam" value={apptF.title} onChange={e=>setApptF(p=>({...p,title:e.target.value}))}/></div>
                  <div><label className="lbl">Date</label><input className="inp" type="date" value={apptF.date} onChange={e=>setApptF(p=>({...p,date:e.target.value}))}/></div>
                  <div><label className="lbl">Time</label><input className="inp" type="time" value={apptF.time} onChange={e=>setApptF(p=>({...p,time:e.target.value}))}/></div>
                  <div><label className="lbl">Provider</label><input className="inp" placeholder="Dr. Harmon" value={apptF.provider} onChange={e=>setApptF(p=>({...p,provider:e.target.value}))}/></div>
                  <div><label className="lbl">Notes</label><input className="inp" placeholder="Special instructions" value={apptF.notes} onChange={e=>setApptF(p=>({...p,notes:e.target.value}))}/></div>
                </div>
                <button className="bg" style={{fontSize:10}} onClick={saveAppt}>Save Appointment</button>
              </FormBox>}
              {(sel.appointments||[]).filter(a=>a.status==="scheduled").length>0&&<><div className="cap" style={{marginBottom:8}}>Scheduled</div>
              {(sel.appointments||[]).filter(a=>a.status==="scheduled").sort((a,b)=>a.date.localeCompare(b.date)).map(a=>(
                <div key={a.id} className={`appt-card appt-${a.type}`} style={{border:"1px solid rgba(255,255,255,.07)"}}>
                  <div style={{display:"flex",alignItems:"flex-start",justifyContent:"space-between"}}>
                    <div style={{flex:1}}>
                      <div style={{display:"flex",gap:7,alignItems:"center",marginBottom:4}}><span style={{fontSize:15}}>{a.type==="vet"?"🩺":"🔨"}</span><span style={{fontSize:14,fontWeight:400}}>{a.title}</span><span className="pill" style={{background:a.type==="vet"?"rgba(125,211,252,.12)":"rgba(201,168,76,.12)",color:a.type==="vet"?"#7dd3fc":"#c9a84c"}}>{a.type}</span></div>
                      <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.42)",lineHeight:1.6}}>📅 {a.date}{a.time&&` at ${a.time}`}{a.provider&&<><br/>👤 {a.provider}</>}{a.notes&&<><br/>📋 {a.notes}</>}</div>
                    </div>
                    <button className="gh" style={{fontSize:9,marginLeft:8}} onClick={()=>completeAppt(a.id)}>✓ Done</button>
                  </div>
                </div>
              ))}</>}
              {(sel.appointments||[]).filter(a=>a.status==="completed").length>0&&<><div className="cap" style={{marginBottom:8,marginTop:16,opacity:.5}}>Completed</div>
              {(sel.appointments||[]).filter(a=>a.status==="completed").map(a=>(
                <div key={a.id} className="appt-card appt-completed" style={{border:"1px solid rgba(255,255,255,.05)"}}>
                  <div style={{display:"flex",alignItems:"center",gap:8}}><span>{a.type==="vet"?"🩺":"🔨"}</span><div><div style={{fontSize:13,fontWeight:300}}>{a.title}</div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.3)"}}>{a.date}{a.provider&&` · ${a.provider}`}</div></div><span className="pill" style={{marginLeft:"auto",background:"rgba(74,222,128,.1)",color:"rgba(74,222,128,.55)"}}>✓ Done</span></div>
                </div>
              ))}</>}
              {(sel.appointments||[]).length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.28)"}}>No appointments yet.</div>}
            </div>}

            {/* Feeding */}
            {tab==="Feeding"&&<div className="fi">
              <SH title="Feeding Schedule" btn="+ Add" onBtn={()=>setShowFeed(!showFeed)}/>
              {showFeed&&<FormBox onClose={()=>setShowFeed(false)}>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:10}}>
                  <div><label className="lbl">Time</label><input className="inp" type="time" value={feedF.time} onChange={e=>setFeedF(p=>({...p,time:e.target.value}))}/></div>
                  <div><label className="lbl">Label</label><input className="inp" placeholder="Morning feed" value={feedF.label} onChange={e=>setFeedF(p=>({...p,label:e.target.value}))}/></div>
                  <div style={{gridColumn:"1/-1"}}><label className="lbl">Amount / Instructions</label><input className="inp" placeholder="4 lbs grain + hay" value={feedF.amount} onChange={e=>setFeedF(p=>({...p,amount:e.target.value}))}/></div>
                </div>
                <button className="bg" style={{fontSize:10}} onClick={saveFeed}>Save</button>
              </FormBox>}
              {(sel.feedingReminders||[]).length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.28)"}}>No feeding reminders set.</div>}
              {[...(sel.feedingReminders||[])].sort((a,b)=>a.time.localeCompare(b.time)).map(r=>(
                <div key={r.id} className="feed-row" style={{opacity:r.active?1:.45}}>
                  <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:13,fontWeight:600,color:"#c9a84c",minWidth:48}}>{r.time}</div>
                  <div style={{flex:1}}><div style={{fontSize:14,fontWeight:300}}>{r.label}</div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.38)",marginTop:1}}>{r.amount}</div></div>
                  <div style={{width:32,height:17,borderRadius:9,background:r.active?"#c9a84c":"rgba(255,255,255,.14)",position:"relative",cursor:"pointer",flexShrink:0,transition:"background .2s"}} onClick={()=>toggleFeed(r.id)}>
                    <div style={{width:13,height:13,borderRadius:"50%",background:"#fff",position:"absolute",top:2,left:r.active?17:2,transition:"left .2s"}}/>
                  </div>
                  <button className="gh" style={{fontSize:9,padding:"2px 6px"}} onClick={()=>deleteFeed(r.id)}>✕</button>
                </div>
              ))}
              {sel.feed&&<div style={{marginTop:14,padding:"10px 13px",background:"rgba(201,168,76,.05)",border:"1px solid rgba(201,168,76,.14)",borderRadius:4}}>
                <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"#c9a84c",letterSpacing:".13em",textTransform:"uppercase",marginBottom:4}}>💡 Feed Plan</div>
                <div style={{fontSize:13,fontWeight:300}}>{sel.feed}</div>
              </div>}
            </div>}

            {/* Care Log */}
            {tab==="Care Log"&&<div className="fi">
              <SH title="Care Log" btn="+ Entry" onBtn={()=>setShowLog(!showLog)}/>
              {showLog&&<FormBox onClose={()=>setShowLog(false)}>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:10}}>
                  <div><label className="lbl">Type</label><select className="sel2" value={logF.type} onChange={e=>setLogF(p=>({...p,type:e.target.value}))}>{Object.entries(LOG_ICONS).map(([k,v])=><option key={k} value={k}>{v} {k[0].toUpperCase()+k.slice(1)}</option>)}</select></div>
                  <div><label className="lbl">By</label><input className="inp" placeholder="Name" value={logF.by} onChange={e=>setLogF(p=>({...p,by:e.target.value}))}/></div>
                </div>
                <div style={{marginBottom:10}}><label className="lbl">Note</label><input className="inp" placeholder="Observation or treatment..." value={logF.note} onChange={e=>setLogF(p=>({...p,note:e.target.value}))}/></div>
                <button className="bg" style={{fontSize:10}} onClick={saveLog}>Save</button>
              </FormBox>}
              {(sel.logs||[]).length===0&&<div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:11,color:"rgba(232,224,208,.28)"}}>No entries yet.</div>}
              {(sel.logs||[]).map((l,i)=>(
                <div key={i} style={{display:"flex",gap:10,padding:"10px 0",borderBottom:"1px solid rgba(255,255,255,.05)"}}>
                  <div style={{width:30,height:30,borderRadius:"50%",background:"rgba(255,255,255,.04)",display:"flex",alignItems:"center",justifyContent:"center",fontSize:13,flexShrink:0}}>{LOG_ICONS[l.type]||"📝"}</div>
                  <div><div style={{display:"flex",gap:7,marginBottom:2}}><span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"#c9a84c",textTransform:"uppercase",letterSpacing:".08em"}}>{l.type}</span><span style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.28)"}}>{l.date}{l.by&&` · ${l.by}`}</span></div><div style={{fontSize:13,fontWeight:300}}>{l.note}</div></div>
                </div>
              ))}
            </div>}

            {/* Staff */}
            {tab==="Staff"&&<div className="fi">
              <div style={{fontSize:19,fontStyle:"italic",fontWeight:300,marginBottom:4}}>Staff Assignment</div>
              <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.35)",marginBottom:18}}>Click to assign or unassign for {sel.name}</div>
              <G2>
                {staffList.map(s=>{
                  const on=(sel.assignedStaff||[]).includes(s.id);
                  return <div key={s.id} className="card" style={{display:"flex",alignItems:"center",gap:11,cursor:"pointer",border:on?"1px solid rgba(201,168,76,.35)":"1px solid rgba(255,255,255,.07)",background:on?"rgba(201,168,76,.06)":"rgba(255,255,255,.025)",transition:"all .2s"}} onClick={()=>toggleStaff(s.id)}>
                    <div className={`staff-av${on?" on":""}`} style={{background:avatarColor(s.name),color:"#fff",fontSize:10}}>{s.avatar||initials(s.name)}</div>
                    <div style={{flex:1}}><div style={{fontSize:13,fontWeight:on?400:300}}>{s.name}</div><div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:9,color:"rgba(232,224,208,.35)",marginTop:1}}>{s.role}</div></div>
                    <div style={{width:16,height:16,borderRadius:"50%",border:"1.5px solid",borderColor:on?"#c9a84c":"rgba(255,255,255,.2)",background:on?"#c9a84c":"transparent",display:"flex",alignItems:"center",justifyContent:"center",fontSize:8,color:"#1a1208",transition:"all .2s"}}>{on?"✓":""}</div>
                  </div>;
                })}
              </G2>
            </div>}
          </div>}

          {/* ADD HORSE */}
          {view==="addHorse"&&<div className="fi" style={{maxWidth:540}}>
            <div style={{fontSize:24,fontStyle:"italic",fontWeight:300,marginBottom:2}}>Register New Horse</div>
            <div style={{fontFamily:"'Josefin Sans',sans-serif",fontSize:10,color:"rgba(232,224,208,.35)",marginBottom:22}}>Add a horse to {barn.name}'s roster</div>
            <G2>
              {[{l:"Name *",k:"name",p:"e.g. Midnight Storm"},{l:"Breed",k:"breed",p:"e.g. Thoroughbred"},{l:"Age",k:"age",p:"Years"},{l:"Stall",k:"stall",p:"e.g. A1"},{l:"Weight (lbs)",k:"weight",p:"1200"}].map(f=>(
                <div key={f.k}><label className="lbl">{f.l}</label><input className="inp" placeholder={f.p} value={addF[f.k]} onChange={e=>setAddF(p=>({...p,[f.k]:e.target.value}))}/></div>
              ))}
              <div><label className="lbl">Status</label><select className="sel2" value={addF.status} onChange={e=>setAddF(p=>({...p,status:e.target.value}))}>{Object.entries(STATUS).map(([k,v])=><option key={k} value={k}>{v.label}</option>)}</select></div>
            </G2>
            <div style={{marginTop:11}}><label className="lbl">Feed Plan</label><input className="inp" placeholder="12 lbs grain + hay 3x daily" value={addF.feed} onChange={e=>setAddF(p=>({...p,feed:e.target.value}))}/></div>
            <div style={{marginTop:11}}><label className="lbl">Notes</label><input className="inp" placeholder="Special care instructions..." value={addF.notes} onChange={e=>setAddF(p=>({...p,notes:e.target.value}))}/></div>
            <div style={{display:"flex",gap:8,marginTop:18}}>
              <button className="bg" onClick={saveHorse}>Register Horse</button>
              <button className="gh" onClick={()=>setView("overview")}>Cancel</button>
            </div>
          </div>}
        </div>
      </div>
    </div>
  );
}

/* ═══════════════════════════════════════════════════════════════════════════════
   ROOT
═══════════════════════════════════════════════════════════════════════════════ */
export default function Root() {
  const [screen, setScreen] = useState("landing"); // landing | login | onboarding | dashboard | settings
  const [appState, setAppState] = useState(null);

  // Load from storage on mount
  useEffect(() => {
    const saved = load();
    if(saved?.barn?.name) { setAppState(saved); setScreen("dashboard"); }
  }, []);

  const persist = (patch) => {
    const next = {...appState,...patch};
    setAppState(next);
    save(next);
  };

  const handleOnboardComplete = (data) => {
    const state = { barn:data.barn, staff:data.staff, horses:[], plan:data.plan };
    setAppState(state);
    save(state);
    setScreen("dashboard");
  };

  const handleLogin = (user) => {
    const saved = load();
    if(saved?.barn?.name) { setAppState(saved); setScreen("dashboard"); }
    else { setAppState({ barn:{name:"My Barn",type:"Private",plan:"pro"}, staff:[{id:"owner",name:user.name,role:"Barn Owner",avatar:initials(user.name),email:user.email}], horses:[] }); setScreen("dashboard"); }
  };

  const handleLogout = () => { setScreen("landing"); };

  const handleSettingsSave = ({barn,staff}) => {
    persist({barn,staff});
    setScreen("dashboard");
  };

  return (
    <>
      <style>{CSS}</style>
      {screen==="landing"    && <Landing onGetStarted={()=>setScreen("onboarding")} onLogin={()=>setScreen("login")}/>}
      {screen==="login"      && <Login onLogin={handleLogin} onBack={()=>setScreen("landing")} onGetStarted={()=>setScreen("onboarding")}/>}
      {screen==="onboarding" && <Onboarding onComplete={handleOnboardComplete}/>}
      {screen==="settings"   && appState && <Settings appState={appState} onSave={handleSettingsSave} onBack={()=>setScreen("dashboard")}/>}
      {screen==="dashboard"  && appState && (
        <Dashboard
          appState={appState}
          onUpdateApp={patch=>persist(patch)}
          onSettings={()=>setScreen("settings")}
          onLogout={handleLogout}
        />
      )}
    </>
  );
}
