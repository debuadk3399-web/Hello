import React, { useEffect, useMemo, useState } from "react"; import { QRCodeCanvas } from "qrcode.react"; import { jsPDF } from "jspdf";

const LS_KEY = "dcm_data_v4"; const SESSION_KEY = "dcm_session_v4"; const TRIAL_KEY = "dcm_trial_start_v4";

const emptyState = { clinic: { name: "", phone: "", address: "", upiId: "", upiName: "", googleBusinessURL: "" }, users: [], patients: [], appointments: [], invoices: [], staff: [], subscriptions: null };

const uid = (prefix = "id") => ${prefix}_${Date.now()}_${Math.random().toString(36).slice(2,8)}; const nowISO = () => new Date().toISOString(); const fmtDT = iso => { try { return iso ? new Date(iso).toLocaleString() : ""; } catch { return ""; } }; const monthKey = iso => { const d = iso ? new Date(iso) : new Date(); return ${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}; };

function loadState(){ try { return JSON.parse(localStorage.getItem(LS_KEY)) || { ...emptyState }; } catch { return { ...emptyState }; } } function saveState(s){ try { localStorage.setItem(LS_KEY, JSON.stringify(s)); } catch(e){ console.error(e); } }

export default function DentalClinicManager(){ const [state, setState] = useState(loadState); const [session, setSession] = useState(()=>{ try { const r = localStorage.getItem(SESSION_KEY); return r?JSON.parse(r):null; } catch { return null; } }); const [view, setView] = useState(session? 'dashboard':'auth'); const [range, setRange] = useState('today');

useEffect(()=> saveState(state), [state]); useEffect(()=> setView(session? 'dashboard':'auth'), [session]);

const [auth, setAuth] = useState({ clinic:'', doctor:'', phone:'', email:'' }); const [isRegister, setIsRegister] = useState(false);

const validPhone = p => /^[6-9]\d{9}$/.test((p||'').trim()); const validGmail = e => /@gmail.com\s*$/.test((e||'').trim());

function register(){ if(!auth.clinic.trim() || !auth.doctor.trim() || !auth.phone.trim() || !auth.email.trim()) return alert('All fields required'); if(!validPhone(auth.phone)) return alert('Phone must be 10 digits'); if(!validGmail(auth.email)) return alert("Email must end with @gmail.com"); const exists = (state.users||[]).find(u=>u.phone===auth.phone && u.clinic===auth.clinic); if(exists) return alert('User exists — login'); const u = { id: uid('user'), ...auth }; setState(s=>({ ...s, users:[...(s.users||[]), u], clinic:{ ...(s.clinic||{}), name: auth.clinic } })); try{ localStorage.setItem(SESSION_KEY, JSON.stringify(u)); }catch{} setSession(u); if(!localStorage.getItem(TRIAL_KEY)) localStorage.setItem(TRIAL_KEY, nowISO()); setIsRegister(false); alert('Registered'); }

function login(){ if(!auth.clinic.trim() || !auth.phone.trim()) return alert('Clinic & phone required'); if(!validPhone(auth.phone)) return alert('Phone must be 10 digits'); const u = (state.users||[]).find(x=>x.clinic===auth.clinic && x.phone===auth.phone); if(!u){ setIsRegister(true); return alert('Not found — register'); } try{ localStorage.setItem(SESSION_KEY, JSON.stringify(u)); }catch{} setSession(u); if(!localStorage.getItem(TRIAL_KEY)) localStorage.setItem(TRIAL_KEY, nowISO()); alert('Logged in'); }

function logout(){ localStorage.removeItem(SESSION_KEY); setSession(null); setView('auth'); }

const isLocked = useMemo(()=>{ if(!session) return true; const trial = localStorage.getItem(TRIAL_KEY); const now = new Date(); if(trial){ const end = new Date(trial); end.setMonth(end.getMonth()+1); if(now<=end) return false; } if(!state.subscriptions?.end) return true; return now > new Date(state.subscriptions.end); }, [session, state.subscriptions]);

function activateSubscription(months, price){ const start=new Date(); const end=new Date(); end.setMonth(end.getMonth()+months); setState(s=>({...s, subscriptions:{ start:start.toISOString(), end:end.toISOString(), months, price }})); alert('Subscription active'); }

function addPatient(p){ if(!p||!p.name||!p.phone) return alert('Name & phone required'); if(!validPhone(p.phone)) return alert('Phone invalid'); const rec={ id: uid('pat'), name: p.name.trim(), age: p.age||'', sex: p.sex||'', phone: p.phone.trim(), address: p.address||'', createdAt: nowISO() }; setState(s=>({...s, patients:[rec, ...(s.patients||[])]})); return rec; }

function addAppointment(a){ if(!a||!a.patientId||!a.dateTimeISO) return alert('Select patient & date/time'); const rec={ id: uid('app'), patientId: a.patientId, dateTimeISO: a.dateTimeISO, notes: a.notes||'', createdAt: nowISO() }; setState(s=>({...s, appointments:[rec, ...(s.appointments||[])]})); return rec; }

async function sendReminder(app){ const p = (state.patients||[]).find(x=>x.id===app.patientId); if(!p) return alert('Patient missing'); const msg = Reminder: ${p.name}, your appointment is on ${fmtDT(app.dateTimeISO)}; try{ const r = await fetch('/api/send-reminder',{ method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ phone: p.phone, message: msg, via: 'whatsapp' }) }); if(r.ok) return alert('Reminder sent'); throw new Error('server'); }catch(e){ const wa=https://wa.me/${(p.phone||'').replace(/[^0-9]/g,'')}?text=${encodeURIComponent(msg)}; window.open(wa,'_blank'); } }

function createInvoice({ patientId, items, method, upiId }){ if(!patientId) return alert('Select patient'); if(!items||items.length===0) return alert('Add items'); for(const it of items){ if(!it.treatment||!it.quantity||!it.price) return alert('Each item needs treatment, quantity, price'); } const p = (state.patients||[]).find(x=>x.id===patientId) || { name:'', phone:'' }; const total = items.reduce((s,i)=> s + (Number(i.quantity||0) * Number(i.price||0)), 0); const inv = { id: uid('inv'), patientId, patientName: p.name||'', phone: p.phone||'', items, total, method: method||'cash', upiId: method==='upi' ? (upiId || state.clinic.upiId || '') : '', paid:false, createdAt: nowISO() }; setState(s=>{ const tr = { ...(s.treatments||{}) }; items.forEach(it=>{ const name = it.treatment||'Unknown'; if(!tr[name]) tr[name]=[]; tr[name].push({ patientId, patientName: p.name||'', dateISO: inv.createdAt }); }); return {...s, invoices:[inv, ...(s.invoices||[])], treatments: tr }; }); return inv; }

function markPaid(id, paid){ setState(s=>({...s, invoices:(s.invoices||[]).map(iv=> iv.id===id ? {...iv, paid: !!paid} : iv )})); }

function canPrint(inv){ return !!inv && !!inv.paid; } function printInvoice(inv){ if(!canPrint(inv)) return alert('Cannot print unpaid invoice'); const rows = (inv.items||[]).map(it=><tr><td>${escapeHtml(it.treatment)}</td><td style="text-align:center">${it.quantity}</td><td style="text-align:right">₹${Number(it.price||0)}</td></tr>).join(''); const html = <!doctype html><html><body><h2>${escapeHtml(state.clinic.name||'Clinic')}</h2><div>${escapeHtml(state.clinic.address||'')} • ${escapeHtml(state.clinic.phone||'')}</div><hr/><div>Invoice: ${escapeHtml(inv.id)}</div><div>Patient: ${escapeHtml(inv.patientName||'')}</div><div>Date: ${escapeHtml(fmtDT(inv.createdAt))}</div><table style="width:100%;border-collapse:collapse"><thead><tr><th>Treatment</th><th>Qty</th><th style="text-align:right">Price</th></tr></thead><tbody>${rows}</tbody></table><h3 style="text-align:right">Total: ₹${Number(inv.total||0)}</h3></body></html>; const w = window.open('','_blank'); if(!w) return alert('Popup blocked'); w.document.write(html); w.document.close(); w.print(); }

async function sendInvoice(inv, { email=false, whatsapp=false }={}){ try{ await fetch('/api/send-invoice',{ method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ invoice: inv, clinic: state.clinic, toEmail: email?inv.email:null, toPhone: whatsapp?inv.phone:null }) }); alert('Sent via server'); }catch(e){ alert('Server unavailable — use manual sharing'); } }

function addStaff(s){ if(!s||!s.name||!s.phone) return alert('Name & phone required'); if(!validPhone(s.phone)) return alert('Phone invalid'); const rec = { id: uid('staff'), name: s.name.trim(), phone: s.phone.trim(), address: s.address||'', role: s.role||'others', salary: s.salary||'', workingDays: s.workingDays||'', times: s.times||'', leaveDays: s.leaveDays||'' }; setState(s=>({...s, staff:[rec, ...(s.staff||[])]})); return rec; } function updateStaff(id, s){ setState(st=>({...st, staff:(st.staff||[]).map(x=> x.id===id ? {...x, ...s} : x )})); } function deleteStaff(id){ if(!confirm('Delete staff?')) return; setState(st=>({...st, staff:(st.staff||[]).filter(x=>x.id!==id)})); }

function exportJSON(){ try{ const blob = new Blob([JSON.stringify(state,null,2)],{type:'application/json'}); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href=url; a.download=dcm_backup_${new Date().toISOString().slice(0,10)}.json; a.click(); URL.revokeObjectURL(url); }catch(e){ alert('Export failed'); } }

function importJSON(file){ const r = new FileReader(); r.onload = ()=>{ try{ const j = JSON.parse(r.result); setState({...emptyState, ...j}); alert('Imported'); }catch(e){ alert('Invalid JSON'); } }; r.readAsText(file); }

function exportPatientsPDF(){ try{ const doc = new jsPDF(); doc.setFontSize(12); doc.text(${state.clinic.name||'Clinic'} - Patients,10,12); let y=22; (state.patients||[]).forEach(p=>{ doc.text(${p.name||''} | ${p.phone||''} | ${p.age||''} | ${p.sex||''},10,y); y+=8; if(y>280){ doc.addPage(); y=20; } }); doc.save('patients.pdf'); }catch(e){ alert('PDF export failed'); } }

function resetApp(){ if(!confirm('Erase all app data? This cannot be undone.')) return; localStorage.removeItem(LS_KEY); localStorage.removeItem(SESSION_KEY); localStorage.removeItem(TRIAL_KEY); setState({...emptyState}); setSession(null); setView('auth'); }

const totalIncome = useMemo(()=> (state.invoices||[]).filter(i=>i.paid).reduce((s,i)=>s+Number(i.total||0),0), [state.invoices]); const treatmentsThisMonth = useMemo(()=>{ const key = monthKey(); const counts = {}; (state.invoices||[]).forEach(inv=>{ if(monthKey(inv.createdAt)!==key) return; (inv.items||[]).forEach(it=>{ const name = it.treatment||'Unknown'; counts[name] = (counts[name]||0) + Number(it.quantity||0); }); }); return counts; }, [state.invoices]);

if(!session) return ( <div style={{minHeight:'100vh',background:'#fff',display:'flex',alignItems:'center',justifyContent:'center',padding:20,fontFamily:'Inter, Arial, sans-serif'}}> <div style={{width:480,border:'1px solid #eef2f6',padding:22,borderRadius:12,boxShadow:'0 10px 30px rgba(2,6,23,0.06)'}}> <h2 style={{margin:0,marginBottom:12,color:'#123b7a'}}>Clinic Manager</h2> <div style={{display:'grid',gap:10}}> <input placeholder="Clinic name" value={auth.clinic} onChange={e=>setAuth(a=>({...a,clinic:e.target.value}))} style={inputStyle} /> <input placeholder="Doctor name" value={auth.doctor} onChange={e=>setAuth(a=>({...a,doctor:e.target.value}))} style={inputStyle} /> <input placeholder="Phone" value={auth.phone} onChange={e=>setAuth(a=>({...a,phone:e.target.value}))} style={inputStyle} /> {isRegister && <input placeholder="Email (must end with @gmail.com)" value={auth.email} onChange={e=>setAuth(a=>({...a,email:e.target.value}))} style={inputStyle} />} <div style={{display:'flex',gap:8,marginTop:6}}> <button onClick={isRegister?register:login} style={primaryBtn}>{isRegister? 'Register':'Login'}</button> <button onClick={()=>setIsRegister(r=>!r)} style={ghostBtn}>{isRegister? 'Already registered? Login':'New user? Register'}</button> </div> </div> </div> </div> );

return ( <div style={{minHeight:'100vh',background:'#fff',display:'flex',flexDirection:'column',fontFamily:'Inter, Arial, sans-serif'}}> <header style={{display:'flex',justifyContent:'space-between',alignItems:'center',padding:'14px 20px',borderBottom:'1px solid #f1f5f9'}}> <div style={{fontSize:18,fontWeight:700,color:'#123b7a'}}>{(state.clinic&&state.clinic.name) || (session&&session.clinic) || 'Your Clinic'}</div> <div style={{display:'flex',gap:10,alignItems:'center'}}> <button onClick={()=>setView('subscription')} style={subBtn}>Subscription</button> <button onClick={logout} style={logoutBtn}>Logout</button> </div> </header>

<main style={{flex:1,overflow:'auto',padding:20}}>
    {view==='dashboard' && (
      <section>
        <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
          <h2 style={{color:'#123b7a'}}>Dashboard</h2>
          <div style={{display:'flex',gap:8,alignItems:'center'}}>
            <label style={{color:'#666'}}>Range</label>
            <select value={range} onChange={e=>setRange(e.target.value)} style={inputSmall}><option value='today'>Today</option><option value='month'>This month</option><option value='year'>This year</option><option value='all'>All time</option></select>
          </div>
        </div>
        <div style={{display:'grid',gridTemplateColumns:'repeat(auto-fit,minmax(220px,1fr))',gap:12,marginTop:16}}>
          <div style={statCard}><div style={{color:'#666'}}>Total Patients</div><div style={{fontSize:20,fontWeight:700,marginTop:6}}>{(state.patients||[]).length}</div></div>
          <div style={statCard}><div style={{color:'#666'}}>Total Income (paid)</div><div style={{fontSize:20,fontWeight:700,marginTop:6}}>₹{totalIncome}</div></div>
          <div style={statCard}><div style={{color:'#666'}}>Treatments this month</div><div style={{fontSize:16,fontWeight:600,marginTop:8}}>{Object.keys(treatmentsThisMonth).length} types</div><div style={{color:'#666',marginTop:6}}>Use Treatments tab for details</div></div>
        </div>
      </section>
    )}

    {view==='patients' && <PatientsPanel patients={state.patients||[]} onAdd={addPatient} />}
    {view==='appointments' && <AppointmentsPanel patients={state.patients||[]} appointments={state.appointments||[]} onAdd={addAppointment} onReminder={sendReminder} locked={isLocked} />}
    {view==='invoices' && <InvoicesPanel patients={state.patients||[]} invoices={state.invoices||[]} onCreate={createInvoice} onMarkPaid={markPaid} onPrint={printInvoice} onSend={sendInvoice} clinic={state.clinic||{}} locked={isLocked} />}
    {view==='archive' && <ArchivePanel invoices={state.invoices||[]} clinic={state.clinic||{}} />}
    {view==='staff' && <StaffPanel staff={state.staff||[]} onAdd={addStaff} onUpdate={updateStaff} onDelete={deleteStaff} locked={isLocked} />}
    {view==='treatments' && <TreatmentsPanel counts={treatmentsThisMonth} />}
    {view==='settings' && <SettingsPanel clinic={state.clinic||{}} onSave={(c)=>setState(s=>({...s, clinic:c}))} />}
    {view==='backup' && <BackupPanel onExportJSON={exportJSON} onImportJSON={importJSON} onExportPDF={exportPatientsPDF} onReset={resetApp} />}
    {view==='subscription' && <SubscriptionPanel subs={state.subscriptions} onBuy={activateSubscription} isLocked={isLocked} />}
  </main>

  <footer style={{borderTop:'1px solid #f1f5f9',padding:'10px 12px',display:'flex',gap:8,flexWrap:'wrap',alignItems:'center',justifyContent:'center'}}>
    <Nav onClick={()=>setView('dashboard')} label='Dashboard' active={view==='dashboard'} />
    <Nav onClick={()=>setView('patients')} label='New Patient' active={view==='patients'} />
    <Nav onClick={()=>!isLocked&&setView('appointments')} label='Appointments' active={view==='appointments'} disabled={isLocked} />
    <Nav onClick={()=>!isLocked&&setView('invoices')} label='Invoices' active={view==='invoices'} disabled={isLocked} />
    <Nav onClick={()=>!isLocked&&setView('archive')} label='Archive' active={view==='archive'} disabled={isLocked} />
    <Nav onClick={()=>!isLocked&&setView('staff')} label='Staff & Salaries' active={view==='staff'} disabled={isLocked} />
    <Nav onClick={()=>!isLocked&&setView('treatments')} label='Treatments' active={view==='treatments'} disabled={isLocked} />
    <Nav onClick={()=>setView('backup')} label='Backup' active={view==='backup'} />
    <Nav onClick={()=>setView('settings')} label='Settings' active={view==='settings'} />
  </footer>
</div>

); }

const inputStyle = { padding:10,border:'1px solid #eef2f6',borderRadius:10,outline:'none' }; const inputSmall = { padding:8,border:'1px solid #eef2f6',borderRadius:8 }; const primaryBtn = { padding:'10px 14px',borderRadius:10,border:0,background:'linear-gradient(90deg,#d64545,#ff7b7b)',color:'#fff',cursor:'pointer' }; const ghostBtn = { padding:'10px 14px',borderRadius:10,border:'1px solid #eef2f6',background:'#fff',cursor:'pointer' }; const subBtn = { padding:'8px 12px',borderRadius:10,border:0,background:'linear-gradient(90deg,#0b63d9,#60a5fa)',color:'#fff',cursor:'pointer' }; const logoutBtn = { padding:'8px 12px',borderRadius:10,border:0,background:'linear-gradient(90deg,#ef4444,#fb7185)',color:'#fff',cursor:'pointer' }; const gradientRed = { padding:'8px 12px',borderRadius:10,border:0,background:'linear-gradient(90deg,#ef4444,#fb7185)',color:'#fff',cursor:'pointer' }; const statCard = { border:'1px solid #eef2f6',padding:16,borderRadius:12,background:'#fff',boxShadow:'0 6px 18px rgba(18,59,122,0.04)'};

function Nav({onClick,label,active,disabled}){ return (<button onClick={onClick} disabled={disabled} style={{padding:'8px 12px',borderRadius:10,border: active? '0':'1px solid #eef2f6',background: active? 'linear-gradient(90deg,#0b63d9,#60a5fa)':'#fff',color: active? '#fff':'#123b7a',cursor: disabled? 'not-allowed':'pointer' }}>{label}</button>); }

function PatientsPanel({patients,onAdd}){ const [f,setF] = useState({ name:'', age:'', sex:'male', phone:'', address:'' }); return (<div style={{maxWidth:900}}> <h3 style={{color:'#123b7a'}}>New Patient</h3> <div style={{display:'grid',gap:8}}> <input style={inputStyle} placeholder='Name' value={f.name} onChange={e=>setF({...f,name:e.target.value})} /> <input style={inputStyle} placeholder='Age' value={f.age} onChange={e=>setF({...f,age:e.target.value})} /> <select style={inputStyle} value={f.sex} onChange={e=>setF({...f,sex:e.target.value})}><option value='male'>Male</option><option value='female'>Female</option><option value='other'>Other</option></select> <input style={inputStyle} placeholder='Phone' value={f.phone} onChange={e=>setF({...f,phone:e.target.value})} /> <textarea style={{...inputStyle,minHeight:80}} placeholder='Address' value={f.address} onChange={e=>setF({...f,address:e.target.value})}></textarea> <div><button style={primaryBtn} onClick={()=>{ if(!f.name||!f.phone) return alert('Name & phone required'); if(!/^[6-9]\d{9}$/.test(f.phone)) return alert('Phone must be 10 digits'); onAdd(f); setF({ name:'', age:'', sex:'male', phone:'', address:'' }); }}>Save</button></div> </div>

<h4 style={{marginTop:18,color:'#123b7a'}}>Patients</h4>
<div style={{border:'1px solid #eef2f6',borderRadius:10,padding:8,maxHeight:340,overflow:'auto'}}>
  {(patients||[]).length===0 && <div style={{color:'#666'}}>No patients yet.</div>}
  {(patients||[]).map(p=>(<div key={p.id} style={{padding:8,borderBottom:'1px solid #f5f7fa'}}><div style={{fontWeight:700}}>{p.name||''}</div><div style={{color:'#666',fontSize:13}}>{p.phone||''} • {p.age||''} • {p.sex||''}</div></div>))}
</div>

  </div>);
}function AppointmentsPanel({patients,appointments,onAdd,onReminder,locked}){ const [patientId,setPatientId]=useState(''); const [dateTimeISO,setDateTimeISO]=useState(''); return (<div style={{maxWidth:1000}}> <h3 style={{color:'#123b7a'}}>Appointments</h3> <div style={{display:'grid',gridTemplateColumns:'2fr 2fr 1fr',gap:8,alignItems:'center'}}> <select style={inputSmall} value={patientId} onChange={e=>setPatientId(e.target.value)}><option value=''>Select patient</option>{(patients||[]).map(p=><option key={p.id} value={p.id}>{p.name||''} - {p.phone||''}</option>)}</select> <input type='datetime-local' style={inputSmall} value={dateTimeISO} onChange={e=>setDateTimeISO(e.target.value)} /> <button style={subBtn} onClick={()=>{ if(locked) return alert('Feature locked — subscribe'); if(!patientId||!dateTimeISO) return alert('Select patient & date/time'); onAdd({patientId,dateTimeISO}); setDateTimeISO(''); }}>Add</button> </div> <div style={{marginTop:12,border:'1px solid #eef2f6',borderRadius:10,padding:8,maxHeight:320,overflow:'auto'}}> {(appointments||[]).len
