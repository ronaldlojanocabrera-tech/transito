<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
<title>VehículoFlotante · UCuenca</title>
<link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=DM+Sans:ital,wght@0,300;0,400;0,500;0,600;0,700;1,400&family=DM+Mono:wght@400;500&display=swap" rel="stylesheet">

<!-- ╔══════════════════════════════════════════════════════════════╗
     ║  PASO 1: Ve a https://console.firebase.google.com           ║
     ║  Crea un proyecto → Agrega una app Web                      ║
     ║  Copia el firebaseConfig que te da y pégalo AQUÍ ABAJO      ║
     ╚══════════════════════════════════════════════════════════════╝ -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.0/firebase-app.js";
import { getAuth, GoogleAuthProvider, OAuthProvider,
         signInWithPopup, signOut, onAuthStateChanged }
  from "https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js";
import { getDatabase, ref, set, push, onValue, get, update, firebaseConfig , remove }
  from "https://www.gstatic.com/firebasejs/11.0.0/firebase-database.js";

/* ▼▼▼▼▼▼▼▼▼  PEGA TU firebaseConfig AQUÍ  ▼▼▼▼▼▼▼▼▼ */
const firebaseConfig = {
  apiKey: "AIzaSyDrXjSUos9qYBqn4_2uzblCiTzycpssA58",
  authDomain: "transito-86deb.firebaseapp.com",
  databaseURL: "https://transito-86deb-default-rtdb.firebaseio.com",
  projectId: "transito-86deb",
  storageBucket: "transito-86deb.firebasestorage.app",
  messagingSenderId: "11304287278",
  appId: "1:11304287278:web:114cc4f7191dbea2b433bf",
  measurementId: "G-GDCP13DBQX"
};
/* ▲▲▲▲▲▲▲▲▲  FIN DE firebaseConfig  ▲▲▲▲▲▲▲▲▲ */

const app  = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db   = getDatabase(app);

/* ─── REGLAS DE BASE DE DATOS (copia esto en Firebase → Database → Rules)
{
  "rules": {
    "projects": {
      "$pid": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    }
  }
}
─── */

// ─── NORMATIVA ECUATORIANA DE VEHÍCULOS (INEN / RTE) ───────────────
const VEHICLE_TYPES = [
  { id:'L1', label:'Motocicleta / Moto', ejes:2, icon:'🏍',
    desc:'2 ruedas, hasta 50cc o más',
    svg:`<svg viewBox="0 0 120 60" fill="none" xmlns="http://www.w3.org/2000/svg">
      <circle cx="20" cy="45" r="12" stroke="#c8401a" stroke-width="3"/>
      <circle cx="100" cy="45" r="12" stroke="#c8401a" stroke-width="3"/>
      <path d="M20 33 L45 20 L70 20 L85 33 L100 33" stroke="#c8401a" stroke-width="3" stroke-linecap="round"/>
      <path d="M45 20 L50 33" stroke="#c8401a" stroke-width="2"/>
      <circle cx="55" cy="20" r="5" stroke="#c8401a" stroke-width="2"/>
      <text x="60" y="58" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO L · 2 EJES</text>
    </svg>` },
  { id:'A1', label:'Automóvil / Taxi / SUV', ejes:2, icon:'🚗',
    desc:'Vehículo liviano hasta 3.5 t',
    svg:`<svg viewBox="0 0 160 70" fill="none" xmlns="http://www.w3.org/2000/svg">
      <rect x="10" y="30" width="140" height="25" rx="4" stroke="#1a6bc8" stroke-width="2.5"/>
      <path d="M30 30 L45 12 L115 12 L130 30" stroke="#1a6bc8" stroke-width="2.5" stroke-linecap="round"/>
      <circle cx="38" cy="55" r="10" stroke="#1a6bc8" stroke-width="2.5"/>
      <circle cx="122" cy="55" r="10" stroke="#1a6bc8" stroke-width="2.5"/>
      <rect x="55" y="15" width="50" height="15" rx="2" stroke="#1a6bc8" stroke-width="1.5" fill="rgba(26,107,200,0.1)"/>
      <text x="80" y="68" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO A · 2 EJES</text>
    </svg>` },
  { id:'B1', label:'Bus / Buseta urbana', ejes:2, icon:'🚌',
    desc:'Transporte público urbano ≤10 t',
    svg:`<svg viewBox="0 0 200 80" fill="none" xmlns="http://www.w3.org/2000/svg">
      <rect x="8" y="15" width="184" height="45" rx="5" stroke="#1a8a4a" stroke-width="2.5"/>
      <rect x="15" y="20" width="170" height="25" rx="2" stroke="#1a8a4a" stroke-width="1.5" fill="rgba(26,138,74,0.08)"/>
      <circle cx="40" cy="62" r="11" stroke="#1a8a4a" stroke-width="2.5"/>
      <circle cx="160" cy="62" r="11" stroke="#1a8a4a" stroke-width="2.5"/>
      <line x1="8" y1="40" x2="192" y2="40" stroke="#1a8a4a" stroke-width="1" stroke-dasharray="4,4"/>
      <text x="100" y="76" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO B · 2 EJES</text>
    </svg>` },
  { id:'C2', label:'Camión 2 ejes (C2)', ejes:2, icon:'🚚',
    desc:'Camión rígido de 2 ejes, hasta 18 t',
    svg:`<svg viewBox="0 0 220 80" fill="none" xmlns="http://www.w3.org/2000/svg">
      <rect x="8" y="25" width="150" height="35" rx="3" stroke="#8a3a1a" stroke-width="2.5"/>
      <rect x="158" y="15" width="54" height="45" rx="4" stroke="#8a3a1a" stroke-width="2.5"/>
      <rect x="165" y="22" width="40" height="20" rx="2" stroke="#8a3a1a" stroke-width="1.5" fill="rgba(138,58,26,0.08)"/>
      <circle cx="45" cy="63" r="11" stroke="#8a3a1a" stroke-width="2.5"/>
      <circle cx="110" cy="63" r="11" stroke="#8a3a1a" stroke-width="2.5"/>
      <circle cx="185" cy="63" r="11" stroke="#8a3a1a" stroke-width="2.5"/>
      <text x="110" y="77" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO C2 · 2 EJES</text>
    </svg>` },
  { id:'C3', label:'Camión 3 ejes (C3)', ejes:3, icon:'🚛',
    desc:'Camión rígido de 3 ejes, hasta 25 t',
    svg:`<svg viewBox="0 0 260 80" fill="none" xmlns="http://www.w3.org/2000/svg">
      <rect x="8" y="25" width="190" height="35" rx="3" stroke="#5a1a8a" stroke-width="2.5"/>
      <rect x="198" y="15" width="54" height="45" rx="4" stroke="#5a1a8a" stroke-width="2.5"/>
      <rect x="205" y="22" width="40" height="20" rx="2" stroke="#5a1a8a" stroke-width="1.5" fill="rgba(90,26,138,0.08)"/>
      <circle cx="40" cy="63" r="11" stroke="#5a1a8a" stroke-width="2.5"/>
      <circle cx="105" cy="63" r="11" stroke="#5a1a8a" stroke-width="2.5"/>
      <circle cx="150" cy="63" r="11" stroke="#5a1a8a" stroke-width="2.5"/>
      <circle cx="225" cy="63" r="11" stroke="#5a1a8a" stroke-width="2.5"/>
      <text x="130" y="77" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO C3 · 3 EJES</text>
    </svg>` },
  { id:'T3S2', label:'Tracto-camión (T3S2)', ejes:5, icon:'🚜',
    desc:'Semirremolque, 5 ejes, hasta 48 t',
    svg:`<svg viewBox="0 0 320 80" fill="none" xmlns="http://www.w3.org/2000/svg">
      <rect x="8" y="25" width="100" height="35" rx="3" stroke="#1a4a8a" stroke-width="2.5"/>
      <rect x="108" y="30" width="200" height="30" rx="3" stroke="#1a4a8a" stroke-width="2.5"/>
      <circle cx="35" cy="63" r="11" stroke="#1a4a8a" stroke-width="2.5"/>
      <circle cx="70" cy="63" r="11" stroke="#1a4a8a" stroke-width="2.5"/>
      <circle cx="155" cy="63" r="11" stroke="#1a4a8a" stroke-width="2.5"/>
      <circle cx="200" cy="63" r="11" stroke="#1a4a8a" stroke-width="2.5"/>
      <circle cx="285" cy="63" r="11" stroke="#1a4a8a" stroke-width="2.5"/>
      <line x1="108" y1="44" x2="108" y2="62" stroke="#1a4a8a" stroke-width="2" stroke-dasharray="3,3"/>
      <text x="160" y="77" font-size="7" fill="#8a7a6a" text-anchor="middle">TIPO T3S2 · 5 EJES · SEMIRREMOLQUE</text>
    </svg>` },
  { id:'BIC', label:'Bicicleta', ejes:2, icon:'🚲',
    desc:'No motorizado',
    svg:`<svg viewBox="0 0 120 65" fill="none" xmlns="http://www.w3.org/2000/svg">
      <circle cx="25" cy="45" r="14" stroke="#4a8a1a" stroke-width="2.5"/>
      <circle cx="95" cy="45" r="14" stroke="#4a8a1a" stroke-width="2.5"/>
      <path d="M25 45 L55 20 L80 20 L95 45" stroke="#4a8a1a" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
      <path d="M55 20 L60 45" stroke="#4a8a1a" stroke-width="2.5"/>
      <circle cx="60" cy="15" r="5" stroke="#4a8a1a" stroke-width="2"/>
      <text x="60" y="62" font-size="7" fill="#8a7a6a" text-anchor="middle">NO MOTORIZADO</text>
    </svg>` },
];

const ROLES = {
  conductor: { label:'Conductor', emoji:'🚗', color:'#c8401a',
    desc:'Maneja el vehículo al ritmo del tráfico. No registra datos.' },
  cronometro: { label:'Cronometrista', emoji:'⏱', color:'#1a6bc8',
    desc:'Registra el tiempo exacto de ida T₁ y vuelta T₂.' },
  C: { label:'Dir. Opuesta (C)', emoji:'↔', color:'#1a8a4a',
    desc:'Cuenta vehículos que vienen en sentido contrario.' },
  Rmas: { label:'Nos Rebasan (R⁺)', emoji:'⬆', color:'#8a3a1a',
    desc:'Cuenta autos que nos superan desde atrás.' },
  Rmenos: { label:'Rebasamos (R⁻)', emoji:'⬇', color:'#5a1a8a',
    desc:'Cuenta autos a los que nosotros pasamos.' },
};

// ─── STATE ───────────────────────────────────────────────────────────
let STATE = {
  user: null,
  currentProject: null,
  myRole: null,
  unsubscribes: [],
};

// ─── AUTH ─────────────────────────────────────────────────────────────
onAuthStateChanged(auth, user => {
  STATE.user = user;
  if (user) {
    showPage('projects');
    loadProjects();
  } else {
    showPage('login');
  }
});

window.loginGoogle = async () => {
  try {
    const provider = new GoogleAuthProvider();
    await signInWithPopup(auth, provider);
  } catch(e) { showToast('Error al iniciar sesión: ' + e.message, 'error'); }
};

window.loginApple = async () => {
  try {
    const provider = new OAuthProvider('apple.com');
    provider.addScope('email');
    provider.addScope('name');
    await signInWithPopup(auth, provider);
  } catch(e) { showToast('Error Apple: ' + e.message + ' (activa Apple en Firebase Auth)', 'error'); }
};

window.doLogout = async () => {
  STATE.unsubscribes.forEach(fn => fn());
  STATE.unsubscribes = [];
  STATE.currentProject = null;
  STATE.myRole = null;
  await signOut(auth);
};

// ─── PAGES ────────────────────────────────────────────────────────────
function showPage(name) {
  document.querySelectorAll('.page').forEach(p => p.classList.add('hidden'));
  const el = document.getElementById('page-' + name);
  if (el) el.classList.remove('hidden');
}

// ─── PROJECTS ─────────────────────────────────────────────────────────
function loadProjects() {
  const u = STATE.user;
  if (!u) return;
  const el = document.getElementById('projects-list');
  el.innerHTML = '<div class="loading-spinner"></div>';

  const projRef = ref(db, 'projects');
  const unsub = onValue(projRef, snap => {
    const data = snap.val() || {};
    const projs = Object.entries(data).filter(([,v]) =>
      v.members && v.members[u.uid]
    );
    if (!projs.length) {
      el.innerHTML = `<div class="empty-state">
        <div style="font-size:3rem;margin-bottom:1rem">📋</div>
        <div style="font-weight:600;margin-bottom:.4rem">Sin proyectos aún</div>
        <div style="color:var(--ink3);font-size:.85rem">Crea uno nuevo o pide al líder que te invite</div>
      </div>`;
      return;
    }
    el.innerHTML = projs.map(([id, p]) => {
      const memberCount = Object.keys(p.members || {}).length;
      const myRole = p.members?.[u.uid]?.role;
      const runCount = Object.keys(p.runs || {}).length;
      return `
      <div class="proj-card" onclick="openProject('${id}')">
        <div class="proj-card-top">
          <div class="proj-name">${p.name}</div>
          <div class="proj-date mono">${p.date || '—'}</div>
        </div>
        <div class="proj-avenue">${p.avenue}</div>
        <div class="proj-meta">
          <span>📏 ${p.length}m</span>
          <span>👥 ${memberCount}/5 miembros</span>
          <span>🔄 ${runCount} registros</span>
          ${myRole ? `<span class="role-tag" style="background:${ROLES[myRole]?.color}20;color:${ROLES[myRole]?.color};border:1px solid ${ROLES[myRole]?.color}40">${ROLES[myRole]?.emoji} ${ROLES[myRole]?.label}</span>` : '<span class="role-tag-warn">Sin rol asignado</span>'}
        </div>
      </div>`;
    }).join('');
  });
  STATE.unsubscribes.push(unsub);
}

window.showCreateProject = () => {
  document.getElementById('modal-create').classList.remove('hidden');
};

window.createProject = async () => {
  const name   = document.getElementById('cp-name').value.trim();
  const avenue = document.getElementById('cp-avenue').value.trim();
  const length = parseFloat(document.getElementById('cp-length').value) || 366;
  const dir1   = document.getElementById('cp-dir1').value || 'Norte → Sur';
  const dir2   = document.getElementById('cp-dir2').value || 'Sur → Norte';
  const runs   = parseInt(document.getElementById('cp-runs').value) || 12;
  const date   = document.getElementById('cp-date').value;

  if (!name || !avenue) { showToast('Completa nombre y avenida', 'error'); return; }

  const u = STATE.user;
  const projRef = ref(db, 'projects');
  const newRef = push(projRef);
  await set(newRef, {
    name, avenue, length, dir1, dir2, runs, date,
    createdBy: u.uid,
    createdAt: Date.now(),
    members: {
      [u.uid]: {
        name: u.displayName || u.email,
        email: u.email,
        photo: u.photoURL || '',
        role: null,
        joinedAt: Date.now(),
      }
    },
    runs: {}
  });

  closeModal('modal-create');
  showToast('Proyecto creado', 'success');
};

// ─── OPEN PROJECT ─────────────────────────────────────────────────────
window.openProject = async (pid) => {
  const snap = await get(ref(db, `projects/${pid}`));
  if (!snap.exists()) return;
  STATE.currentProject = { id: pid, ...snap.val() };
  const u = STATE.user;
  const myMember = STATE.currentProject.members?.[u.uid];
  STATE.myRole = myMember?.role || null;

  // Join project if not member
  if (!myMember) {
    await set(ref(db, `projects/${pid}/members/${u.uid}`), {
      name: u.displayName || u.email,
      email: u.email,
      photo: u.photoURL || '',
      role: null,
      joinedAt: Date.now(),
    });
  }

  if (!STATE.myRole) {
    showRolePicker(pid);
  } else {
    enterFieldView(pid);
  }
};

// ─── ROLE PICKER ──────────────────────────────────────────────────────
function showRolePicker(pid) {
  showPage('roles');
  document.getElementById('role-project-name').textContent =
    STATE.currentProject.name;

  // Show current members with roles
  const membersRef = ref(db, `projects/${pid}/members`);
  const unsub = onValue(membersRef, snap => {
    const members = snap.val() || {};
    const taken = Object.values(members).map(m => m.role).filter(Boolean);

    document.getElementById('role-members-live').innerHTML =
      Object.values(members).map(m => {
        const r = ROLES[m.role];
        return `<div class="member-live">
          <div class="ml-avatar" style="background:${r?.color||'#888'}">${(m.name||'?')[0].toUpperCase()}</div>
          <div>
            <div style="font-weight:600;font-size:.85rem">${m.name||'Anónimo'}</div>
            <div style="font-size:.72rem;color:var(--ink3)">${r ? r.emoji+' '+r.label : 'Sin rol'}</div>
          </div>
        </div>`;
      }).join('');

    document.getElementById('role-cards').innerHTML =
      Object.entries(ROLES).map(([key, r]) => {
        const isTaken = taken.includes(key) &&
          members[STATE.user.uid]?.role !== key;
        return `<div class="role-card ${isTaken ? 'role-taken' : ''}" 
          onclick="${isTaken ? '' : `selectRole('${pid}','${key}')`}">
          <div class="rc-emoji">${r.emoji}</div>
          <div class="rc-name">${r.label}</div>
          <div class="rc-desc">${r.desc}</div>
          ${isTaken ? '<div class="rc-taken-badge">Ocupado</div>' : ''}
          <div class="rc-color-bar" style="background:${r.color}"></div>
        </div>`;
      }).join('');
  });
  STATE.unsubscribes.push(unsub);
}

window.selectRole = async (pid, role) => {
  const u = STATE.user;
  await update(ref(db, `projects/${pid}/members/${u.uid}`), { role });
  STATE.myRole = role;
  STATE.currentProject.members = STATE.currentProject.members || {};
  STATE.currentProject.members[u.uid] = {
    ...STATE.currentProject.members[u.uid], role
  };
  showToast(`Rol asignado: ${ROLES[role].label}`, 'success');
  setTimeout(() => enterFieldView(pid), 600);
};

// ─── FIELD VIEW ───────────────────────────────────────────────────────
function enterFieldView(pid) {
  showPage('field');
  const role = STATE.myRole;
  const R = ROLES[role] || {};
  const p = STATE.currentProject;

  // Header
  document.getElementById('field-proj-name').textContent = p.name;
  document.getElementById('field-role-badge').textContent = `${R.emoji} ${R.label}`;
  document.getElementById('field-role-badge').style.background = R.color + '25';
  document.getElementById('field-role-badge').style.color = R.color;
  document.getElementById('field-role-badge').style.borderColor = R.color + '60';

  // Direction buttons
  document.getElementById('fd-dir1').textContent = '↓ ' + p.dir1;
  document.getElementById('fd-dir2').textContent = '↑ ' + p.dir2;

  // Role-specific panel
  buildRolePanel(role, pid);

  // Live team panel
  listenTeam(pid);

  // Live runs feed
  listenRuns(pid);
}

// ─── DIRECTION ────────────────────────────────────────────────────────
let activeDir = 1;
window.setDir = (d) => {
  activeDir = d;
  document.getElementById('fd-dir1').className =
    'dir-btn' + (d===1 ? ' dir-btn-active' : '');
  document.getElementById('fd-dir2').className =
    'dir-btn' + (d===2 ? ' dir-btn-active' : '');
  // refresh panel counts
  Object.keys(curCounts).forEach(k => curCounts[k] = 0);
  refreshCountDisplays();
};

// ─── ROLE PANELS ──────────────────────────────────────────────────────
let curCounts = {};
let timerState = { running:false, start:null, elapsed:0, interval:null, phase:1, t1:null };

function buildRolePanel(role, pid) {
  const panel = document.getElementById('role-panel');

  if (role === 'conductor') {
    panel.innerHTML = `
      <div class="conductor-view">
        <div class="cv-icon">🚗</div>
        <h2>Modo Conductor</h2>
        <p>Tu trabajo es conducir al ritmo natural del tráfico.<br>
        No necesitas registrar datos — el equipo se encarga.</p>
        <div class="cv-instruction">
          <div>✦ Mantén la velocidad promedio del flujo</div>
          <div>✦ Sigue la ruta definida del tramo</div>
          <div>✦ El cronometrista te indicará inicio y fin</div>
        </div>
        <div class="live-feed-section" style="margin-top:2rem">
          <div class="lf-title">📊 Lo que registra tu equipo en tiempo real</div>
          <div id="conductor-live-feed"></div>
        </div>
      </div>`;
    listenConductorFeed(pid);
    return;
  }

  if (role === 'cronometro') {
    panel.innerHTML = `
      <div class="timer-panel">
        <div class="tp-instruction">
          Corrida <strong id="run-num-disp">—</strong> · 
          <span id="timer-phase-label" style="color:var(--accent)">Esperando inicio</span>
        </div>
        <div class="big-timer" id="big-timer">00:00.0</div>
        <div class="timer-btns">
          <button class="tbtn tbtn-start" id="tbtn-start" onclick="timerAction('${pid}')">
            ▶ Iniciar T₁
          </button>
          <button class="tbtn tbtn-reset" onclick="timerReset()">↺ Reset</button>
        </div>
        <div class="timer-log" id="timer-log">
          <div class="tl-empty">Los tiempos registrados aparecerán aquí</div>
        </div>
        <div class="formula-box">
          T₁ (ida): <strong id="t1-disp">—</strong> min &nbsp;·&nbsp;
          T₂ (vuelta): <strong id="t2-disp">—</strong> min<br>
          v = d/t̄ donde d = <strong>${STATE.currentProject.length}m</strong>
        </div>
      </div>`;
    initTimer(pid);
    return;
  }

  if (role === 'C') {
    panel.innerHTML = buildCounterPanel('C', pid,
      '↔ Dirección Opuesta',
      '#1a8a4a',
      'Cuenta TODOS los vehículos que vienen en sentido contrario',
      true);
    initCounterVehicles('C', pid);
    return;
  }

  if (role === 'Rmas') {
    panel.innerHTML = buildCounterPanel('Rmas', pid,
      '⬆ Nos Rebasan (R⁺)',
      '#8a3a1a',
      'Cuenta los vehículos que vienen de atrás y NOS superan',
      false);
    initCounterVehicles('Rmas', pid);
    return;
  }

  if (role === 'Rmenos') {
    panel.innerHTML = buildCounterPanel('Rmenos', pid,
      '⬇ Rebasamos (R⁻)',
      '#5a1a8a',
      'Cuenta los vehículos a los que NOSOTROS superamos',
      false);
    initCounterVehicles('Rmenos', pid);
    return;
  }
}

// ─── COUNTER PANEL HTML ───────────────────────────────────────────────
function buildCounterPanel(role, pid, title, color, desc, showAllTypes) {
  const vTypes = showAllTypes ? VEHICLE_TYPES : VEHICLE_TYPES.slice(0, 5);
  return `
  <div class="counter-panel">
    <div class="cp-header" style="border-color:${color}">
      <div class="cp-title" style="color:${color}">${title}</div>
      <div class="cp-desc">${desc}</div>
    </div>

    <div class="vehicle-selector">
      <div class="vs-label">Tipo de vehículo detectado:</div>
      <div class="vs-grid" id="vs-grid-${role}">
        ${vTypes.map(v => `
          <div class="vs-card ${v.id === 'A1' ? 'vs-selected' : ''}" 
               id="vsc-${role}-${v.id}"
               onclick="selectVehicle('${role}','${v.id}')">
            <div class="vsc-svg">${v.svg}</div>
            <div class="vsc-label">${v.label}</div>
            <div class="vsc-ejes">INEN · ${v.ejes} eje${v.ejes>1?'s':''}</div>
          </div>`).join('')}
      </div>
    </div>

    <div class="big-counter-area">
      <div class="bca-label">Total esta corrida</div>
      <div class="bca-value" id="bca-${role}">0</div>
      <div class="bca-btns">
        <button class="bca-btn-add" onclick="quickCount('${role}','${pid}')">
          ＋ REGISTRAR
        </button>
        <button class="bca-btn-sub" onclick="undoCount('${role}','${pid}')">
          ↩ Deshacer
        </button>
      </div>
    </div>

    <div class="breakdown-grid" id="breakdown-${role}">
      ${VEHICLE_TYPES.map(v => `
        <div class="bg-item">
          <span>${v.icon}</span>
          <span class="bg-label">${v.label.split('/')[0].trim()}</span>
          <span class="bg-count mono" id="bgc-${role}-${v.id}">0</span>
        </div>`).join('')}
    </div>

    <div class="session-log" id="log-${role}">
      <div class="sl-title">📋 Registros de esta sesión</div>
      <div id="log-entries-${role}">
        <div style="color:var(--ink3);font-size:.82rem;padding:.5rem">Aún sin registros</div>
      </div>
    </div>
  </div>`;
}

// ─── VEHICLE SELECTION ────────────────────────────────────────────────
let selectedVehicle = { C:'A1', Rmas:'A1', Rmenos:'A1' };

window.selectVehicle = (role, vid) => {
  selectedVehicle[role] = vid;
  document.querySelectorAll(`#vs-grid-${role} .vs-card`).forEach(el => {
    el.classList.remove('vs-selected');
  });
  const el = document.getElementById(`vsc-${role}-${vid}`);
  if (el) el.classList.add('vs-selected');
};

function initCounterVehicles(role, pid) {
  curCounts[role] = 0;
  if (!window._sessionCounts) window._sessionCounts = {};
  window._sessionCounts[role] = {};
  VEHICLE_TYPES.forEach(v => { window._sessionCounts[role][v.id] = 0; });
}

window.quickCount = async (role, pid) => {
  const vid = selectedVehicle[role] || 'A1';
  const vLabel = VEHICLE_TYPES.find(v => v.id === vid)?.label || vid;
  curCounts[role] = (curCounts[role] || 0) + 1;
  if (!window._sessionCounts) window._sessionCounts = {};
  if (!window._sessionCounts[role]) window._sessionCounts[role] = {};
  window._sessionCounts[role][vid] = (window._sessionCounts[role][vid] || 0) + 1;

  // Update display
  document.getElementById(`bca-${role}`).textContent = curCounts[role];
  const bgEl = document.getElementById(`bgc-${role}-${vid}`);
  if (bgEl) bgEl.textContent = window._sessionCounts[role][vid];

  // Animate
  const btn = document.querySelector(`#role-panel .bca-btn-add`);
  btn.style.transform = 'scale(0.94)';
  setTimeout(() => btn.style.transform = '', 120);

  // Save to Firebase with timestamp
  const runRef = push(ref(db, `projects/${pid}/events`));
  await set(runRef, {
    type: role,
    vehicleType: vid,
    vehicleLabel: vLabel,
    direction: activeDir,
    timestamp: Date.now(),
    userId: STATE.user.uid,
    userName: STATE.user.displayName || STATE.user.email,
  });

  // Add to local log
  addLogEntry(role, vid, vLabel);
};

window.undoCount = async (role, pid) => {
  const vid = selectedVehicle[role] || 'A1';
  curCounts[role] = Math.max(0, (curCounts[role] || 0) - 1);
  if (window._sessionCounts?.[role]?.[vid])
    window._sessionCounts[role][vid] = Math.max(0, window._sessionCounts[role][vid] - 1);

  document.getElementById(`bca-${role}`).textContent = curCounts[role];
  const bgEl = document.getElementById(`bgc-${role}-${vid}`);
  if (bgEl) bgEl.textContent = window._sessionCounts[role][vid] || 0;
  showToast('Desecho el último conteo', 'info');
};

function addLogEntry(role, vid, vLabel) {
  const container = document.getElementById(`log-entries-${role}`);
  if (!container) return;
  const now = new Date();
  const timeStr = now.toTimeString().slice(0,8);
  const vt = VEHICLE_TYPES.find(v => v.id === vid);
  const entry = document.createElement('div');
  entry.className = 'log-entry new-entry';
  entry.innerHTML = `
    <span class="le-time mono">${timeStr}</span>
    <span class="le-icon">${vt?.icon || '🚗'}</span>
    <span class="le-label">${vLabel}</span>
    <span class="le-dir">Dir ${activeDir}</span>`;
  // Remove "sin registros" placeholder
  const empty = container.querySelector('[style*="color:var(--ink3)"]');
  if (empty) empty.remove();
  container.prepend(entry);
  setTimeout(() => entry.classList.remove('new-entry'), 400);
  // Keep max 20 entries
  while (container.children.length > 20) container.lastChild.remove();
}

function refreshCountDisplays() {
  ['C','Rmas','Rmenos'].forEach(role => {
    const el = document.getElementById(`bca-${role}`);
    if (el) el.textContent = curCounts[role] || 0;
    VEHICLE_TYPES.forEach(v => {
      const bg = document.getElementById(`bgc-${role}-${v.id}`);
      if (bg) bg.textContent = window._sessionCounts?.[role]?.[v.id] || 0;
    });
  });
}

// ─── TIMER ────────────────────────────────────────────────────────────
function initTimer(pid) {
  timerState = { running:false, start:null, elapsed:0, interval:null, phase:1, t1:null };
  updateRunNumDisplay(pid);
}

async function updateRunNumDisplay(pid) {
  const snap = await get(ref(db, `projects/${pid}/runs`));
  const count = snap.exists() ? Object.keys(snap.val()).length : 0;
  const el = document.getElementById('run-num-disp');
  if (el) el.textContent = count + 1;
}

window.timerAction = async (pid) => {
  if (!timerState.running) {
    // START
    timerState.running = true;
    timerState.start = Date.now() - timerState.elapsed;
    timerState.interval = setInterval(() => {
      timerState.elapsed = Date.now() - timerState.start;
      const s = timerState.elapsed / 1000;
      const m = Math.floor(s / 60).toString().padStart(2, '0');
      const sec = (s % 60).toFixed(1).padStart(4, '0');
      const el = document.getElementById('big-timer');
      if (el) { el.textContent = `${m}:${sec}`; el.classList.add('timer-running'); }
    }, 100);
    const btn = document.getElementById('tbtn-start');
    if (btn) { btn.textContent = '⏹ Detener'; btn.className = 'tbtn tbtn-stop'; }
    const lbl = document.getElementById('timer-phase-label');
    if (lbl) lbl.textContent = timerState.phase === 1 ? 'Midiendo T₁ (ida)...' : 'Midiendo T₂ (vuelta)...';
  } else {
    // STOP
    clearInterval(timerState.interval);
    timerState.running = false;
    const elapsed = timerState.elapsed;
    const mins = elapsed / 1000 / 60;

    const el = document.getElementById('big-timer');
    if (el) { el.textContent = '00:00.0'; el.classList.remove('timer-running'); }
    timerState.elapsed = 0;

    if (timerState.phase === 1) {
      timerState.t1 = mins;
      document.getElementById('t1-disp').textContent = mins.toFixed(4);
      timerState.phase = 2;
      const btn = document.getElementById('tbtn-start');
      if (btn) { btn.textContent = '▶ Iniciar T₂'; btn.className = 'tbtn tbtn-start'; }
      const lbl = document.getElementById('timer-phase-label');
      if (lbl) lbl.textContent = 'T₁ registrado ✓ — Ahora regresa';
      showToast(`T₁ = ${mins.toFixed(4)} min registrado`, 'success');
      addTimerLog(`T₁ ida — Dir ${activeDir}`, mins);

      // Publish T1 live to Firebase
      await set(ref(db, `projects/${pid}/liveTimer`), {
        t1: mins, t2: null, dir: activeDir,
        by: STATE.user.displayName || STATE.user.email,
        ts: Date.now()
      });

    } else {
      const t2 = mins;
      document.getElementById('t2-disp').textContent = t2.toFixed(4);
      addTimerLog(`T₂ vuelta — Dir ${activeDir}`, t2);
      showToast(`T₁+T₂ = ${(timerState.t1 + t2).toFixed(4)} min ✓`, 'success');

      // Save full run to Firebase
      const runData = {
        t1: timerState.t1, t2, tTotal: timerState.t1 + t2,
        direction: activeDir,
        by: STATE.user.displayName || STATE.user.email,
        timestamp: Date.now(),
        date: new Date().toISOString(),
      };
      const runRef = push(ref(db, `projects/${pid}/runs`));
      await set(runRef, runData);

      // Update live timer
      await update(ref(db, `projects/${pid}/liveTimer`), { t2, tTotal: timerState.t1 + t2 });

      timerState.phase = 1;
      timerState.t1 = null;
      const btn = document.getElementById('tbtn-start');
      if (btn) { btn.textContent = '▶ Iniciar T₁'; btn.className = 'tbtn tbtn-start'; }
      document.getElementById('t1-disp').textContent = '—';
      document.getElementById('t2-disp').textContent = '—';
      const lbl = document.getElementById('timer-phase-label');
      if (lbl) lbl.textContent = 'Corrida guardada ✓ — Listo para la siguiente';
      updateRunNumDisplay(pid);
    }
  }
};

window.timerReset = () => {
  clearInterval(timerState.interval);
  timerState = { running:false, start:null, elapsed:0, interval:null, phase:1, t1:null };
  const el = document.getElementById('big-timer');
  if (el) { el.textContent = '00:00.0'; el.classList.remove('timer-running'); }
  const btn = document.getElementById('tbtn-start');
  if (btn) { btn.textContent = '▶ Iniciar T₁'; btn.className = 'tbtn tbtn-start'; }
  document.getElementById('t1-disp').textContent = '—';
  document.getElementById('t2-disp').textContent = '—';
};

function addTimerLog(label, mins) {
  const log = document.getElementById('timer-log');
  if (!log) return;
  const empty = log.querySelector('.tl-empty');
  if (empty) empty.remove();
  const div = document.createElement('div');
  div.className = 'tl-entry';
  div.innerHTML = `
    <span class="tl-label">${label}</span>
    <span class="tl-val mono">${mins.toFixed(4)} min</span>
    <span class="tl-val mono">${(mins * 60).toFixed(2)} seg</span>`;
  log.prepend(div);
}

// ─── LIVE TEAM LISTENER ───────────────────────────────────────────────
function listenTeam(pid) {
  const membersRef = ref(db, `projects/${pid}/members`);
  const unsub = onValue(membersRef, snap => {
    const members = snap.val() || {};
    const el = document.getElementById('live-team');
    if (!el) return;
    el.innerHTML = Object.entries(members).map(([uid, m]) => {
      const r = ROLES[m.role];
      const isMe = uid === STATE.user?.uid;
      return `<div class="lt-member ${isMe ? 'lt-me' : ''}">
        <div class="lt-avatar" style="background:${r?.color||'#888'}">${(m.name||'?')[0].toUpperCase()}</div>
        <div class="lt-info">
          <div class="lt-name">${m.name||'Anónimo'}${isMe?' (tú)':''}</div>
          <div class="lt-role">${r ? r.emoji+' '+r.label : 'Sin rol'}</div>
        </div>
        <div class="lt-dot" style="background:${r?.color||'#888'}"></div>
      </div>`;
    }).join('');
  });
  STATE.unsubscribes.push(unsub);
}

// ─── LIVE RUNS FEED ───────────────────────────────────────────────────
function listenRuns(pid) {
  const eventsRef = ref(db, `projects/${pid}/events`);
  const unsub = onValue(eventsRef, snap => {
    const events = snap.val() || {};
    const el = document.getElementById('global-feed');
    if (!el) return;
    const sorted = Object.values(events).sort((a,b) => b.timestamp - a.timestamp).slice(0, 30);
    if (!sorted.length) {
      el.innerHTML = '<div style="color:var(--ink3);font-size:.82rem;padding:1rem;text-align:center">Sin eventos aún</div>';
      return;
    }
    el.innerHTML = sorted.map(ev => {
      const role = ROLES[ev.type];
      const t = new Date(ev.timestamp).toTimeString().slice(0,8);
      const vt = VEHICLE_TYPES.find(v => v.id === ev.vehicleType);
      return `<div class="gf-entry">
        <span class="gf-time mono">${t}</span>
        <span class="gf-icon" title="${ev.type}">${role?.emoji||'?'}</span>
        <span class="gf-veh">${vt?.icon||''} ${ev.vehicleLabel||'—'}</span>
        <span class="gf-dir">Dir ${ev.direction}</span>
        <span class="gf-user">${(ev.userName||'?').split(' ')[0]}</span>
      </div>`;
    }).join('');
  });
  STATE.unsubscribes.push(unsub);

  // Runs counter
  const runsRef = ref(db, `projects/${pid}/runs`);
  const unsub2 = onValue(runsRef, snap => {
    const runs = snap.val() || {};
    const count = Object.keys(runs).length;
    const el = document.getElementById('runs-count');
    if (el) el.textContent = count;

    // Update live stats
    computeLiveStats(runs, pid);
  });
  STATE.unsubscribes.push(unsub2);
}

function computeLiveStats(runs, pid) {
  // Get all events for counting
  get(ref(db, `projects/${pid}/events`)).then(snap => {
    const events = snap.val() || {};
    const p = STATE.currentProject;

    [1,2].forEach(dir => {
      const dirRuns = Object.values(runs).filter(r => r.direction === dir);
      const dirEvents = Object.values(events).filter(e => e.direction === dir);

      if (!dirRuns.length) return;
      const totT = dirRuns.reduce((a,r) => a + (r.tTotal||0), 0);
      const C = dirEvents.filter(e => e.type === 'C').length;
      const Rp = dirEvents.filter(e => e.type === 'Rmas').length;
      const Rm = dirEvents.filter(e => e.type === 'Rmenos').length;

      const q = totT > 0 ? (C + Rp - Rm) / totT : 0; // veh/min
      const tAvg = totT / dirRuns.length;
      const v = tAvg > 0 ? (p.length / 1000) / (tAvg / 60) : 0;
      const k = v > 0 ? (q * 60) / v : 0;

      const suffix = dir === 1 ? '1' : '2';
      const qEl = document.getElementById(`stat-q-${suffix}`);
      const vEl = document.getElementById(`stat-v-${suffix}`);
      const kEl = document.getElementById(`stat-k-${suffix}`);
      if (qEl) qEl.textContent = (q * 60).toFixed(1);
      if (vEl) vEl.textContent = v.toFixed(1);
      if (kEl) kEl.textContent = k.toFixed(2);
    });
  });
}

function listenConductorFeed(pid) {
  const eventsRef = ref(db, `projects/${pid}/events`);
  onValue(eventsRef, snap => {
    const events = snap.val() || {};
    const el = document.getElementById('conductor-live-feed');
    if (!el) return;
    const sorted = Object.values(events).sort((a,b) => b.timestamp - a.timestamp).slice(0,15);
    const C = sorted.filter(e => e.type==='C').length;
    const Rp = sorted.filter(e => e.type==='Rmas').length;
    const Rm = sorted.filter(e => e.type==='Rmenos').length;
    el.innerHTML = `
      <div class="clf-stats">
        <div class="clf-stat"><div class="clf-val">${C}</div><div class="clf-lbl">C opuestos</div></div>
        <div class="clf-stat"><div class="clf-val" style="color:var(--accent2)">${Rp}</div><div class="clf-lbl">R⁺ nos rebasan</div></div>
        <div class="clf-stat"><div class="clf-val" style="color:var(--accent)">${Rm}</div><div class="clf-lbl">R⁻ rebasamos</div></div>
      </div>`;
  });
}

// ─── EXPORT ───────────────────────────────────────────────────────────
window.exportData = async () => {
  const pid = STATE.currentProject.id;
  const [runsSnap, eventsSnap] = await Promise.all([
    get(ref(db, `projects/${pid}/runs`)),
    get(ref(db, `projects/${pid}/events`)),
  ]);
  const p = STATE.currentProject;
  const runs = runsSnap.val() || {};
  const events = eventsSnap.val() || {};

  let csv = `PROYECTO:,${p.name}\nAVENIDA:,${p.avenue}\nLONGITUD (m):,${p.length}\n\n`;
  csv += `CORRIDAS (T₁ + T₂)\n`;
  csv += `ID,Dirección,T₁(min),T₂(min),T_total(min),Por,Fecha\n`;
  Object.entries(runs).forEach(([id, r]) => {
    const dn = r.direction===1 ? p.dir1 : p.dir2;
    csv += `${id},${dn},${r.t1?.toFixed(4)||0},${r.t2?.toFixed(4)||0},${r.tTotal?.toFixed(4)||0},${r.by},${r.date}\n`;
  });

  csv += `\nEVENTOS DE CONTEO\n`;
  csv += `Hora,Tipo,Vehículo,Ejes,Dirección,Por\n`;
  Object.values(events).sort((a,b)=>a.timestamp-b.timestamp).forEach(e => {
    const vt = VEHICLE_TYPES.find(v=>v.id===e.vehicleType);
    csv += `${new Date(e.timestamp).toISOString()},${e.type},${e.vehicleLabel},${vt?.ejes||'-'},Dir ${e.direction},${e.userName}\n`;
  });

  // Summary
  [1,2].forEach(dir => {
    const dirRuns = Object.values(runs).filter(r=>r.direction===dir);
    const dirEventsArr = Object.values(events).filter(e=>e.direction===dir);
    if (!dirRuns.length) return;
    const totT = dirRuns.reduce((a,r)=>a+(r.tTotal||0),0);
    const C = dirEventsArr.filter(e=>e.type==='C').length;
    const Rp = dirEventsArr.filter(e=>e.type==='Rmas').length;
    const Rm = dirEventsArr.filter(e=>e.type==='Rmenos').length;
    const q = totT>0?(C+Rp-Rm)/totT:0;
    const tAvg=totT/dirRuns.length;
    const v=tAvg>0?(p.length/1000)/(tAvg/60):0;
    const k=v>0?(q*60)/v:0;
    const dn=dir===1?p.dir1:p.dir2;
    csv+=`\nRESUMEN ${dn}\n`;
    csv+=`q(veh/h),${(q*60).toFixed(3)}\nv(km/h),${v.toFixed(3)}\nk(veh/km),${k.toFixed(4)}\n`;
    csv+=`C total,${C}\nR+ total,${Rp}\nR- total,${Rm}\n`;
  });

  const blob = new Blob([csv], {type:'text/csv;charset=utf-8;'});
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = `${p.name.replace(/\s+/g,'_')}_campo.csv`;
  a.click();
  showToast('CSV exportado correctamente', 'success');
};

// ─── UTILS ────────────────────────────────────────────────────────────
window.showToast = showToast;
function showToast(msg, type='info') {
  const el = document.getElementById('toast');
  el.textContent = msg;
  el.className = `toast toast-${type} show`;
  setTimeout(() => el.classList.remove('show'), 3000);
}

window.openModal = (id) => document.getElementById(id).classList.remove('hidden');
window.closeModal = (id) => document.getElementById(id).classList.add('hidden');

window.openTab = (id) => {
  document.querySelectorAll('.field-tab-btn').forEach(b => b.classList.remove('ftb-active'));
  document.querySelectorAll('.field-tab-pane').forEach(p => p.classList.add('hidden'));
  document.getElementById('tab-'+id).classList.remove('hidden');
  event.currentTarget.classList.add('ftb-active');
};

window.goBack = () => {
  STATE.unsubscribes.forEach(fn => fn());
  STATE.unsubscribes = [];
  showPage('projects');
  loadProjects();
};

window.openTab = (id) => {
  document.querySelectorAll('.field-tab-btn').forEach(b => b.classList.remove('ftb-active'));
  document.querySelectorAll('.field-tab-pane').forEach(p => p.classList.add('hidden'));
  document.getElementById('tab-'+id)?.classList.remove('hidden');
};

// Set today's date on load
document.getElementById('cp-date').value = new Date().toISOString().split('T')[0];
</script>

<style>
:root {
  --bg:#f4f1ea; --paper:#fdfcf8; --card:#ffffff;
  --ink:#1c1712; --ink2:#3d342a; --ink3:#7a6e62;
  --border:#ddd5c8; --border2:#c9bfb0;
  --accent:#c8401a; --accent2:#1a6bc8; --accent3:#1a8a4a;
  --shadow:0 1px 8px rgba(28,23,18,.07);
  --shadow-md:0 4px 20px rgba(28,23,18,.11);
  --shadow-lg:0 12px 48px rgba(28,23,18,.16);
  --r:10px;
}
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
html,body{height:100%;font-family:'DM Sans',sans-serif;background:var(--bg);color:var(--ink);font-size:16px}
.hidden{display:none!important}
.page{min-height:100vh}
.mono{font-family:'DM Mono',monospace}

/* ─ BUTTONS ─ */
.btn{display:inline-flex;align-items:center;gap:.5rem;padding:.65rem 1.3rem;border-radius:8px;border:none;cursor:pointer;font-family:'DM Sans',sans-serif;font-weight:600;font-size:.9rem;transition:all .18s;white-space:nowrap;text-decoration:none}
.btn-primary{background:var(--accent);color:#fff}.btn-primary:hover{background:#a83316;transform:translateY(-1px);box-shadow:0 4px 16px rgba(200,64,26,.3)}
.btn-ink{background:var(--ink);color:#fff}.btn-ink:hover{background:#2c241c}
.btn-ghost{background:transparent;border:1.5px solid var(--border);color:var(--ink)}.btn-ghost:hover{border-color:var(--ink)}
.btn-google{background:#fff;color:#3c4043;border:1.5px solid var(--border);gap:.75rem}.btn-google:hover{background:#f8f8f8;box-shadow:var(--shadow-md)}
.btn-apple{background:#000;color:#fff}.btn-apple:hover{background:#1a1a1a}
.btn-sm{padding:.4rem .9rem;font-size:.82rem}
.btn-lg{padding:.85rem 2rem;font-size:1.02rem}
.btn-block{width:100%;justify-content:center}

/* ─ INPUTS ─ */
input,select{background:var(--paper);border:1.5px solid var(--border);border-radius:8px;padding:.6rem .9rem;font-family:'DM Sans',sans-serif;font-size:.9rem;color:var(--ink);width:100%;transition:border-color .15s,box-shadow .15s}
input:focus,select:focus{outline:none;border-color:var(--accent);box-shadow:0 0 0 3px rgba(200,64,26,.1)}
label{font-size:.73rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3);display:block;margin-bottom:.35rem}
.fg{display:flex;flex-direction:column;gap:.3rem}
.g2{display:grid;grid-template-columns:1fr 1fr;gap:.9rem}
.g3{display:grid;grid-template-columns:1fr 1fr 1fr;gap:.9rem}
@media(max-width:600px){.g2,.g3{grid-template-columns:1fr}}

/* ─ CARD ─ */
.card{background:var(--card);border:1px solid var(--border);border-radius:var(--r);padding:1.4rem;box-shadow:var(--shadow)}

/* ══════════════════════════════════
   LOGIN
══════════════════════════════════ */
#page-login{display:flex;min-height:100vh}
.login-left{
  flex:1;background:var(--ink);color:#fff;
  padding:3.5rem;display:flex;flex-direction:column;justify-content:space-between;
  position:relative;overflow:hidden;min-height:100vh;
}
.login-left::before{content:'';position:absolute;right:-120px;top:-120px;width:380px;height:380px;border-radius:50%;background:rgba(200,64,26,.14);pointer-events:none}
.login-left::after{content:'';position:absolute;left:-60px;bottom:-60px;width:220px;height:220px;border-radius:50%;background:rgba(200,64,26,.08);pointer-events:none}
.ll-logo{font-family:'Bebas Neue',sans-serif;font-size:4rem;line-height:.9;position:relative;z-index:1}
.ll-logo span{color:var(--accent)}
.ll-tagline{font-size:.95rem;color:rgba(255,255,255,.5);line-height:1.7;margin-top:.75rem;max-width:320px;position:relative;z-index:1}
.ll-method{display:inline-flex;align-items:center;gap:.5rem;margin-top:1.5rem;font-family:'DM Mono',monospace;font-size:.68rem;letter-spacing:.08em;color:rgba(255,255,255,.3);border:1px solid rgba(255,255,255,.1);padding:.3rem .7rem;border-radius:4px;position:relative;z-index:1}
.authors-section{position:relative;z-index:1}
.as-label{font-family:'DM Mono',monospace;font-size:.65rem;letter-spacing:.1em;text-transform:uppercase;color:rgba(255,255,255,.3);margin-bottom:.8rem}
.author-row{display:flex;align-items:center;gap:.7rem;padding:.55rem 0;border-bottom:1px solid rgba(255,255,255,.06)}
.author-row:last-child{border-bottom:none}
.ar-av{width:34px;height:34px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Bebas Neue',sans-serif;font-size:.95rem;color:#fff;flex-shrink:0}
.ar-name{font-weight:600;font-size:.85rem}
.ar-role{font-size:.72rem;color:rgba(255,255,255,.4);margin-top:.05rem}
.ar-num{margin-left:auto;font-family:'DM Mono',monospace;font-size:.6rem;color:rgba(255,255,255,.2);border:1px solid rgba(255,255,255,.1);padding:.15rem .4rem;border-radius:3px}
.ucuenca{font-family:'DM Mono',monospace;font-size:.65rem;color:rgba(255,255,255,.25);margin-top:1.5rem;letter-spacing:.04em}

.login-right{width:460px;flex-shrink:0;background:var(--paper);padding:4rem 3rem;display:flex;flex-direction:column;justify-content:center}
@media(max-width:860px){#page-login{flex-direction:column}.login-left{padding:2.5rem 2rem;min-height:auto}.login-right{width:100%;padding:2.5rem 2rem}}
.lr-title{font-family:'Bebas Neue',sans-serif;font-size:2.2rem;margin-bottom:.35rem}
.lr-sub{font-size:.88rem;color:var(--ink3);margin-bottom:2rem;line-height:1.6}
.auth-btns{display:flex;flex-direction:column;gap:.75rem}
.auth-divider{display:flex;align-items:center;gap:.75rem;margin:.5rem 0;color:var(--ink3);font-size:.82rem}
.auth-divider::before,.auth-divider::after{content:'';flex:1;height:1px;background:var(--border)}
.firebase-note{background:rgba(200,64,26,.05);border:1px solid rgba(200,64,26,.15);border-radius:8px;padding:1rem;margin-top:1.5rem;font-size:.78rem;color:var(--ink2);line-height:1.6}
.firebase-note strong{color:var(--accent)}
.fn-steps{margin-top:.5rem;display:flex;flex-direction:column;gap:.25rem}
.fn-step{display:flex;gap:.5rem;align-items:flex-start}

/* ══════════════════════════════════
   PROJECTS
══════════════════════════════════ */
#page-projects{background:var(--bg)}
.pp-header{background:var(--ink);padding:1rem 2rem;display:flex;align-items:center;justify-content:space-between;gap:1rem}
.pp-logo{font-family:'Bebas Neue',sans-serif;font-size:1.7rem;color:#fff}
.pp-logo span{color:var(--accent)}
.pp-user{display:flex;align-items:center;gap:.6rem}
.ppu-photo{width:34px;height:34px;border-radius:50%;object-fit:cover;border:2px solid rgba(255,255,255,.2)}
.ppu-name{color:rgba(255,255,255,.8);font-size:.85rem;font-weight:500}
.pp-body{max-width:1100px;margin:0 auto;padding:2rem}
.pp-top{display:flex;align-items:center;justify-content:space-between;margin-bottom:1.5rem;flex-wrap:wrap;gap:1rem}
.pp-title{font-family:'Bebas Neue',sans-serif;font-size:1.8rem}
.projects-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(300px,1fr));gap:1rem}
.proj-card{background:var(--card);border:1px solid var(--border);border-radius:var(--r);padding:1.25rem;cursor:pointer;transition:all .18s;position:relative;overflow:hidden}
.proj-card::before{content:'';position:absolute;left:0;top:0;width:4px;height:100%;background:var(--accent)}
.proj-card:hover{border-color:var(--accent);transform:translateY(-2px);box-shadow:var(--shadow-md)}
.proj-name{font-family:'Bebas Neue',sans-serif;font-size:1.2rem;margin-bottom:.2rem}
.proj-avenue{font-size:.85rem;color:var(--ink3);margin-bottom:.75rem}
.proj-date{font-size:.7rem;color:var(--ink3)}
.proj-meta{display:flex;flex-wrap:wrap;gap:.5rem;font-size:.72rem;color:var(--ink3)}
.role-tag{padding:.2rem .5rem;border-radius:4px;font-weight:600;font-size:.68rem}
.role-tag-warn{padding:.2rem .5rem;border-radius:4px;background:rgba(245,158,11,.1);color:#b45309;border:1px solid rgba(245,158,11,.3);font-size:.68rem}
.empty-state{text-align:center;padding:4rem 1rem;color:var(--ink3)}
.loading-spinner{width:36px;height:36px;border:3px solid var(--border);border-top-color:var(--accent);border-radius:50%;animation:spin .8s linear infinite;margin:3rem auto}
@keyframes spin{to{transform:rotate(360deg)}}

/* ══════════════════════════════════
   ROLE PICKER
══════════════════════════════════ */
#page-roles{background:var(--bg)}
.rp-header{background:var(--ink);padding:1rem 2rem;display:flex;align-items:center;gap:1rem}
.rp-back{background:none;border:none;color:rgba(255,255,255,.6);cursor:pointer;font-size:1.2rem;padding:.25rem}
.rp-back:hover{color:#fff}
.rp-title{font-family:'Bebas Neue',sans-serif;font-size:1.5rem;color:#fff}
.rp-body{max-width:900px;margin:0 auto;padding:2rem}
.rp-proj{font-size:.85rem;color:var(--ink3);margin-bottom:.35rem}
.rp-h{font-family:'Bebas Neue',sans-serif;font-size:2rem;margin-bottom.5rem}
.rp-sub{font-size:.9rem;color:var(--ink3);margin-bottom:2rem;line-height:1.6}
.role-cards-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:1rem;margin-bottom:2rem}
.role-card{background:var(--card);border:2px solid var(--border);border-radius:var(--r);padding:1.25rem;cursor:pointer;transition:all .2s;position:relative;overflow:hidden;text-align:center}
.role-card:hover:not(.role-taken){border-color:var(--ink);transform:translateY(-2px);box-shadow:var(--shadow-md)}
.role-taken{opacity:.45;cursor:not-allowed}
.rc-emoji{font-size:2rem;margin-bottom:.5rem}
.rc-name{font-weight:700;font-size:.9rem;margin-bottom:.35rem}
.rc-desc{font-size:.73rem;color:var(--ink3);line-height:1.4}
.rc-color-bar{position:absolute;bottom:0;left:0;right:0;height:3px}
.rc-taken-badge{position:absolute;top:.5rem;right:.5rem;background:var(--border);color:var(--ink3);font-size:.6rem;font-weight:700;text-transform:uppercase;padding:.15rem .4rem;border-radius:3px}
.live-members-bar{background:var(--card);border:1px solid var(--border);border-radius:var(--r);padding:1rem 1.25rem;margin-bottom:1.5rem}
.lmb-title{font-size:.72rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3);margin-bottom:.75rem}
.members-live-list{display:flex;flex-wrap:wrap;gap:.6rem}
.member-live{display:flex;align-items:center;gap:.5rem;background:var(--bg);border:1px solid var(--border);border-radius:50px;padding:.3rem .75rem .3rem .4rem;font-size:.82rem}
.ml-avatar{width:26px;height:26px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Bebas Neue',sans-serif;font-size:.72rem;color:#fff;flex-shrink:0}

/* ══════════════════════════════════
   FIELD VIEW
══════════════════════════════════ */
#page-field{background:var(--bg);min-height:100vh}
.field-header{background:var(--ink);padding:.85rem 1.5rem;display:flex;align-items:center;gap:1rem;position:sticky;top:0;z-index:100;flex-wrap:wrap}
.fh-logo{font-family:'Bebas Neue',sans-serif;font-size:1.4rem;color:#fff}
.fh-logo span{color:var(--accent)}
.fh-project{font-size:.75rem;color:rgba(255,255,255,.45);font-family:'DM Mono',monospace}
.fh-role-badge{padding:.25rem .7rem;border-radius:50px;font-size:.75rem;font-weight:600;margin-left:.25rem}
.fh-right{margin-left:auto;display:flex;align-items:center;gap:.6rem}

.field-body{display:grid;grid-template-columns:1fr 280px;gap:1.5rem;max-width:1300px;margin:0 auto;padding:1.5rem}
@media(max-width:900px){.field-body{grid-template-columns:1fr}}

/* Direction buttons */
.dir-bar{display:flex;gap:.6rem;margin-bottom:1.5rem;flex-wrap:wrap;align-items:center}
.dir-btn{padding:.55rem 1.1rem;border-radius:7px;border:2px solid var(--border);background:var(--card);font-weight:600;font-size:.88rem;cursor:pointer;transition:all .15s}
.dir-btn-active{background:var(--accent);color:#fff;border-color:var(--accent)}
.dir-label{font-size:.75rem;color:var(--ink3);margin-left:auto}

/* Counter panel */
.counter-panel{}
.cp-header{border-left:4px solid;padding-left:1rem;margin-bottom:1.5rem}
.cp-title{font-family:'Bebas Neue',sans-serif;font-size:1.5rem}
.cp-desc{font-size:.83rem;color:var(--ink3);margin-top:.2rem}

/* Vehicle selector */
.vehicle-selector{margin-bottom:1.5rem}
.vs-label{font-size:.75rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3);margin-bottom:.75rem}
.vs-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:.65rem}
.vs-card{background:var(--card);border:2px solid var(--border);border-radius:var(--r);padding:.75rem;cursor:pointer;transition:all .15s;text-align:center}
.vs-card:hover{border-color:var(--ink)}
.vs-selected{border-color:var(--accent);background:rgba(200,64,26,.04)}
.vsc-svg{width:100%;margin-bottom:.5rem}
.vsc-svg svg{width:100%;height:auto;max-height:50px}
.vsc-label{font-weight:600;font-size:.78rem;line-height:1.3}
.vsc-ejes{font-family:'DM Mono',monospace;font-size:.62rem;color:var(--ink3);margin-top:.2rem}

/* Big counter */
.big-counter-area{text-align:center;padding:2rem 1rem;background:var(--bg);border-radius:var(--r);border:2px dashed var(--border);margin-bottom:1rem}
.bca-label{font-size:.75rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3);margin-bottom:.5rem}
.bca-value{font-family:'Bebas Neue',sans-serif;font-size:7rem;line-height:1;color:var(--ink);margin:.25rem 0}
.bca-btns{display:flex;gap:.75rem;justify-content:center;margin-top:1rem;flex-wrap:wrap}
.bca-btn-add{
  padding:1.1rem 2.5rem;background:var(--accent);color:#fff;border:none;
  border-radius:10px;font-family:'Bebas Neue',sans-serif;font-size:1.4rem;
  cursor:pointer;transition:all .12s;letter-spacing:.04em;
}
.bca-btn-add:hover{background:#a83316;transform:scale(1.03)}
.bca-btn-add:active{transform:scale(.97)}
.bca-btn-sub{
  padding:.75rem 1.5rem;background:var(--bg);color:var(--ink2);
  border:2px solid var(--border);border-radius:10px;
  font-weight:600;font-size:.9rem;cursor:pointer;transition:all .12s;
}
.bca-btn-sub:hover{border-color:var(--ink)}

/* Breakdown */
.breakdown-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(130px,1fr));gap:.5rem;margin-bottom:1rem}
.bg-item{display:flex;align-items:center;gap:.5rem;background:var(--card);border:1px solid var(--border);border-radius:7px;padding:.5rem .75rem;font-size:.8rem}
.bg-label{flex:1;color:var(--ink2)}
.bg-count{font-weight:700;min-width:24px;text-align:right;color:var(--accent)}

/* Session log */
.session-log{background:var(--card);border:1px solid var(--border);border-radius:var(--r);overflow:hidden}
.sl-title{padding:.6rem 1rem;background:var(--bg);border-bottom:1px solid var(--border);font-size:.72rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3)}
.log-entry{display:flex;align-items:center;gap:.6rem;padding:.5rem 1rem;border-bottom:1px solid var(--border);font-size:.8rem;transition:background .2s}
.log-entry:last-child{border-bottom:none}
.new-entry{background:rgba(200,64,26,.06)}
.le-time{color:var(--ink3);min-width:65px;font-size:.72rem}
.le-icon{font-size:1rem}
.le-label{flex:1;color:var(--ink2)}
.le-dir{font-size:.7rem;color:var(--ink3);background:var(--bg);padding:.1rem .35rem;border-radius:3px}

/* Timer panel */
.timer-panel{}
.tp-instruction{font-size:.85rem;color:var(--ink3);margin-bottom:.75rem}
.big-timer{font-family:'Bebas Neue',sans-serif;font-size:6rem;text-align:center;letter-spacing:.02em;line-height:1;color:var(--ink);padding:.5rem 0;background:var(--bg);border-radius:var(--r);border:2px solid var(--border)}
.timer-running{color:var(--accent);border-color:var(--accent);animation:timerGlow 1.2s ease-in-out infinite}
@keyframes timerGlow{0%,100%{box-shadow:none}50%{box-shadow:0 0 24px rgba(200,64,26,.25)}}
.timer-btns{display:flex;gap:.75rem;justify-content:center;margin:1rem 0;flex-wrap:wrap}
.tbtn{padding:.8rem 2rem;border:none;border-radius:8px;font-family:'DM Sans',sans-serif;font-weight:700;font-size:1rem;cursor:pointer;transition:all .15s}
.tbtn-start{background:var(--accent3);color:#fff}.tbtn-start:hover{background:#157a40}
.tbtn-stop{background:var(--accent);color:#fff}.tbtn-stop:hover{background:#a83316}
.tbtn-reset{background:var(--bg);color:var(--ink2);border:2px solid var(--border)}.tbtn-reset:hover{border-color:var(--ink)}
.timer-log{background:var(--bg);border:1px solid var(--border);border-radius:var(--r);padding:.5rem;margin-top:.5rem;min-height:60px;max-height:200px;overflow-y:auto}
.tl-empty{font-size:.82rem;color:var(--ink3);padding:.5rem;text-align:center}
.tl-entry{display:flex;align-items:center;gap:.75rem;padding:.45rem .5rem;border-bottom:1px solid var(--border);font-size:.82rem}
.tl-entry:last-child{border-bottom:none}
.tl-label{flex:1;color:var(--ink2)}
.tl-val{color:var(--accent)}
.formula-box{background:var(--bg);border-left:3px solid var(--accent);border-radius:0 6px 6px 0;padding:.75rem 1rem;font-family:'DM Mono',monospace;font-size:.78rem;color:var(--ink2);margin-top:1rem;line-height:1.7}
.formula-box strong{color:var(--accent)}

/* Conductor view */
.conductor-view{text-align:center;padding:2rem 1rem}
.cv-icon{font-size:4rem;margin-bottom:1rem}
.conductor-view h2{font-family:'Bebas Neue',sans-serif;font-size:2rem;margin-bottom:.5rem}
.conductor-view p{color:var(--ink3);line-height:1.7;margin-bottom:1.5rem}
.cv-instruction{background:var(--bg);border-radius:var(--r);padding:1.25rem;text-align:left;display:flex;flex-direction:column;gap:.6rem;font-size:.88rem;border:1px solid var(--border)}
.clf-stats{display:grid;grid-template-columns:1fr 1fr 1fr;gap:.75rem;margin-top:.75rem}
.clf-stat{background:var(--card);border:1px solid var(--border);border-radius:8px;padding:.75rem;text-align:center}
.clf-val{font-family:'Bebas Neue',sans-serif;font-size:2.2rem;color:var(--ink)}
.clf-lbl{font-size:.7rem;color:var(--ink3);margin-top:.15rem}
.lf-title{font-size:.75rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em;color:var(--ink3);margin-bottom:.75rem}

/* Sidebar */
.field-sidebar{}
.sidebar-section{background:var(--card);border:1px solid var(--border);border-radius:var(--r);overflow:hidden;margin-bottom:1rem}
.ss-title{padding:.6rem 1rem;background:var(--ink);color:rgba(255,255,255,.7);font-size:.68rem;font-weight:600;text-transform:uppercase;letter-spacing:.06em}
.live-team{display:flex;flex-direction:column;gap:.35rem;padding:.75rem}
.lt-member{display:flex;align-items:center;gap:.6rem;padding:.4rem .5rem;border-radius:6px;font-size:.82rem}
.lt-me{background:var(--bg)}
.lt-avatar{width:28px;height:28px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-family:'Bebas Neue',sans-serif;font-size:.75rem;color:#fff;flex-shrink:0}
.lt-info{flex:1}
.lt-name{font-weight:600;font-size:.82rem}
.lt-role{font-size:.68rem;color:var(--ink3)}
.lt-dot{width:7px;height:7px;border-radius:50%;flex-shrink:0}

/* Global feed */
.global-feed{display:flex;flex-direction:column;overflow-y:auto;max-height:320px}
.gf-entry{display:flex;align-items:center;gap:.4rem;padding:.42rem .75rem;border-bottom:1px solid var(--border);font-size:.75rem}
.gf-entry:last-child{border-bottom:none}
.gf-time{color:var(--ink3);min-width:58px;font-size:.68rem}
.gf-icon{font-size:.85rem}
.gf-veh{flex:1;color:var(--ink2)}
.gf-dir{font-size:.65rem;color:var(--ink3);background:var(--bg);padding:.1rem .3rem;border-radius:3px}
.gf-user{font-size:.65rem;color:var(--ink3);min-width:50px;text-align:right}

/* Stats mini */
.stats-mini{display:grid;grid-template-columns:1fr 1fr;gap:.5rem;padding:.75rem}
.sm-item{background:var(--bg);border-radius:7px;padding:.6rem .75rem;text-align:center}
.sm-val{font-family:'Bebas Neue',sans-serif;font-size:1.6rem;color:var(--ink)}
.sm-lab{font-size:.62rem;color:var(--ink3);text-transform:uppercase;letter-spacing:.04em}

/* Runs count */
.runs-count-badge{background:var(--accent);color:#fff;border-radius:50px;padding:.2rem .65rem;font-size:.78rem;font-weight:700;font-family:'DM Mono',monospace}

/* MODAL */
.modal-overlay{position:fixed;inset:0;background:rgba(28,23,18,.55);backdrop-filter:blur(3px);z-index:200;display:flex;align-items:center;justify-content:center;padding:1rem}
.modal{background:var(--card);border-radius:14px;padding:2rem;max-width:540px;width:100%;box-shadow:var(--shadow-lg);animation:slideUp .25s ease}
@keyframes slideUp{from{transform:translateY(14px);opacity:0}to{transform:translateY(0);opacity:1}}
.modal-title{font-family:'Bebas Neue',sans-serif;font-size:1.7rem;margin-bottom:1.25rem}
.modal-body{display:flex;flex-direction:column;gap:.85rem}
.modal-actions{display:flex;gap:.75rem;justify-content:flex-end;margin-top:1.5rem;flex-wrap:wrap}

/* TOAST */
.toast{position:fixed;bottom:1.5rem;right:1.5rem;z-index:9999;padding:.85rem 1.2rem;border-radius:10px;font-size:.88rem;font-weight:500;max-width:320px;box-shadow:var(--shadow-lg);transform:translateY(80px);opacity:0;transition:all .28s ease;pointer-events:none}
.toast.show{transform:translateY(0);opacity:1}
.toast-success{background:#1a8a4a;color:#fff}
.toast-error{background:#c8401a;color:#fff}
.toast-info{background:var(--ink);color:#fff}
</style>
</head>
<body>

<!-- ════════════════════════════════
     PÁGINA: LOGIN
════════════════════════════════ -->
<div id="page-login" class="page">
  <div class="login-left">
    <div>
      <div class="ll-logo">Vehículo<br><span>Flotante</span></div>
      <p class="ll-tagline">Sistema colaborativo de recolección de datos para el Método del Observador Moviéndose. 5 personas · tiempo real · campo.</p>
      <div class="ll-method">✦ GARBER & HOEL · GREENSHIELDS · UCuenca</div>
    </div>

    <div class="authors-section">
      <div class="as-label">Autores · Facultad de Ingeniería Civil</div>

      <div class="author-row">
        <div class="ar-av" style="background:#c8401a">CE</div>
        <div><div class="ar-name">Christian Espinoza</div><div class="ar-role">Conductor / Cronometrista</div></div>
        <div class="ar-num">ROL 1</div>
      </div>
      <div class="author-row">
        <div class="ar-av" style="background:#1a6bc8">GT</div>
        <div><div class="ar-name">Gabriela Torres</div><div class="ar-role">Conteo Dir. Opuesta (C)</div></div>
        <div class="ar-num">ROL 2</div>
      </div>
      <div class="author-row">
        <div class="ar-av" style="background:#1a8a4a">DM</div>
        <div><div class="ar-name">Diana Morillo</div><div class="ar-role">Vehículos Rebasados (R⁺)</div></div>
        <div class="ar-num">ROL 3</div>
      </div>
      <div class="author-row">
        <div class="ar-av" style="background:#8a3a1a">JV</div>
        <div><div class="ar-name">Josué Vera</div><div class="ar-role">Vehículos que Rebasan (R⁻)</div></div>
        <div class="ar-num">ROL 4</div>
      </div>
      <div class="author-row">
        <div class="ar-av" style="background:#5a1a8a">AP</div>
        <div><div class="ar-name">Analí Pérez</div><div class="ar-role">Registro y Coordinación</div></div>
        <div class="ar-num">ROL 5</div>
      </div>

      <div class="ucuenca">🏛 Universidad de Cuenca · Ecuador · 2025</div>
    </div>
  </div>

  <div class="login-right">
    <div class="lr-title">Iniciar Sesión</div>
    <p class="lr-sub">Usa tu cuenta de Google o Apple. Tus datos se sincronizan con el equipo en tiempo real.</p>

    <div class="auth-btns">
      <button class="btn btn-google btn-lg btn-block" onclick="loginGoogle()">
        <svg width="18" height="18" viewBox="0 0 24 24"><path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/><path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/><path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l3.66-2.84z"/><path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/></svg>
        Continuar con Google
      </button>

      <div class="auth-divider">o</div>

      <button class="btn btn-apple btn-lg btn-block" onclick="loginApple()">
        <svg width="18" height="18" viewBox="0 0 814 1000" fill="white"><path d="M788.1 340.9c-5.8 4.5-108.2 62.2-108.2 190.5 0 148.4 130.3 200.9 134.2 202.2-.6 3.2-20.7 71.9-68.7 141.9-42.8 61.6-87.5 123.1-155.5 123.1s-85.5-39.5-164-39.5c-76.5 0-103.7 40.8-165.9 40.8s-105-57.8-155.5-127.4C46 790.7 0 663 0 541.8c0-194.3 126.4-297.5 250.8-297.5 66.1 0 121.2 43.4 162.7 43.4 39.5 0 101.1-46 176.3-46 28.5 0 130.9 2.6 198.3 99.2zm-234-181.5c31.1-36.9 53.1-88.1 53.1-139.3 0-7.1-.6-14.3-1.9-20.1-50.6 1.9-110.8 33.7-147.1 75.8-28.5 32.4-55.1 83.6-55.1 135.5 0 7.8 1.3 15.6 1.9 18.1 3.2.6 8.4 1.3 13.6 1.3 45.4 0 102.5-30.4 135.5-71.3z"/></svg>
        Continuar con Apple
      </button>
    </div>

    <div class="firebase-note">
      <strong>⚙ Configuración requerida</strong> — Este archivo necesita Firebase para funcionar.
      <div class="fn-steps">
        <div class="fn-step"><span>1.</span><span>Ve a <strong>console.firebase.google.com</strong> → crea proyecto gratis</span></div>
        <div class="fn-step"><span>2.</span><span>Authentication → habilita <strong>Google</strong> y <strong>Apple</strong></span></div>
        <div class="fn-step"><span>3.</span><span>Realtime Database → crea base de datos</span></div>
        <div class="fn-step"><span>4.</span><span>Configuración del proyecto → copia <strong>firebaseConfig</strong></span></div>
        <div class="fn-step"><span>5.</span><span>Pégalo en el HTML donde dice <strong>REEMPLAZA</strong></span></div>
      </div>
    </div>
  </div>
</div>

<!-- ════════════════════════════════
     PÁGINA: PROYECTOS
════════════════════════════════ -->
<div id="page-projects" class="page hidden">
  <div class="pp-header">
    <div class="pp-logo">Vehículo<span>Flotante</span></div>
    <div class="pp-user">
      <img id="pp-photo" class="ppu-photo" src="" alt="" onerror="this.style.display='none'">
      <span class="ppu-name" id="pp-name">—</span>
      <button class="btn btn-ghost btn-sm" style="color:rgba(255,255,255,.7);border-color:rgba(255,255,255,.2)" onclick="doLogout()">Salir</button>
    </div>
  </div>

  <div class="pp-body">
    <div class="pp-top">
      <div>
        <div class="pp-title">Mis Proyectos</div>
        <div style="font-size:.82rem;color:var(--ink3)">Selecciona uno o crea una nueva práctica</div>
      </div>
      <button class="btn btn-primary" onclick="showCreateProject()">✦ Nuevo Proyecto</button>
    </div>
    <div class="projects-grid" id="projects-list">
      <div class="loading-spinner"></div>
    </div>
  </div>
</div>

<!-- ════════════════════════════════
     PÁGINA: SELECCIÓN DE ROL
════════════════════════════════ -->
<div id="page-roles" class="page hidden">
  <div class="rp-header">
    <button class="rp-back" onclick="goBack()">←</button>
    <div class="rp-title">Selecciona tu Rol</div>
  </div>
  <div class="rp-body">
    <div class="rp-proj">Proyecto:</div>
    <div style="font-family:'Bebas Neue',sans-serif;font-size:1.6rem;margin-bottom:.4rem" id="role-project-name">—</div>
    <p class="rp-sub">Cada integrante escoge un rol diferente. Una vez asignado, accedes a tu vista de campo personalizada.</p>

    <div class="live-members-bar">
      <div class="lmb-title">👥 Equipo conectado ahora</div>
      <div class="members-live-list" id="role-members-live"></div>
    </div>

    <div class="role-cards-grid" id="role-cards"></div>
  </div>
</div>

<!-- ════════════════════════════════
     PÁGINA: CAMPO (vista de rol)
════════════════════════════════ -->
<div id="page-field" class="page hidden">

  <div class="field-header">
    <div>
      <div class="fh-logo">Vehículo<span>Flotante</span></div>
      <div class="fh-project" id="field-proj-name">—</div>
    </div>
    <span class="fh-role-badge" id="field-role-badge">—</span>
    <div class="fh-right">
      <span class="runs-count-badge"><span id="runs-count">0</span> corridas</span>
      <button class="btn btn-ghost btn-sm" style="color:rgba(255,255,255,.7);border-color:rgba(255,255,255,.2)" onclick="exportData()">↓ CSV</button>
      <button class="btn btn-ghost btn-sm" style="color:rgba(255,255,255,.7);border-color:rgba(255,255,255,.2)" onclick="goBack()">← Salir</button>
    </div>
  </div>

  <div class="field-body">
    <!-- MAIN PANEL -->
    <div>
      <!-- Direction selector -->
      <div class="dir-bar">
        <button class="dir-btn dir-btn-active" id="fd-dir1" onclick="setDir(1)">↓ Norte → Sur</button>
        <button class="dir-btn" id="fd-dir2" onclick="setDir(2)">↑ Sur → Norte</button>
        <span class="dir-label">Dirección activa del vehículo</span>
      </div>

      <!-- Role-specific panel -->
      <div id="role-panel"></div>
    </div>

    <!-- SIDEBAR -->
    <div class="field-sidebar">
      <!-- Team live -->
      <div class="sidebar-section">
        <div class="ss-title">👥 Equipo en campo</div>
        <div class="live-team" id="live-team"></div>
      </div>

      <!-- Live stats -->
      <div class="sidebar-section">
        <div class="ss-title">📊 Estadísticas en vivo</div>
        <div style="padding:.6rem .75rem;font-size:.68rem;color:var(--ink3);border-bottom:1px solid var(--border)">Dirección 1</div>
        <div class="stats-mini">
          <div class="sm-item"><div class="sm-val" id="stat-q-1">—</div><div class="sm-lab">q veh/h</div></div>
          <div class="sm-item"><div class="sm-val" id="stat-v-1">—</div><div class="sm-lab">v km/h</div></div>
          <div class="sm-item"><div class="sm-val" id="stat-k-1">—</div><div class="sm-lab">k veh/km</div></div>
          <div class="sm-item"><div class="sm-val" id="runs-count-1">—</div><div class="sm-lab">corridas</div></div>
        </div>
        <div style="padding:.6rem .75rem;font-size:.68rem;color:var(--ink3);border-top:1px solid var(--border);border-bottom:1px solid var(--border)">Dirección 2</div>
        <div class="stats-mini">
          <div class="sm-item"><div class="sm-val" id="stat-q-2">—</div><div class="sm-lab">q veh/h</div></div>
          <div class="sm-item"><div class="sm-val" id="stat-v-2">—</div><div class="sm-lab">v km/h</div></div>
          <div class="sm-item"><div class="sm-val" id="stat-k-2">—</div><div class="sm-lab">k veh/km</div></div>
          <div class="sm-item"><div class="sm-val" id="runs-count-2">—</div><div class="sm-lab">corridas</div></div>
        </div>
      </div>

      <!-- Global event feed -->
      <div class="sidebar-section">
        <div class="ss-title">⚡ Actividad del equipo</div>
        <div class="global-feed" id="global-feed">
          <div style="color:var(--ink3);font-size:.82rem;padding:1rem;text-align:center">Sin eventos aún</div>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- ════════════════════════════════
     MODAL: CREAR PROYECTO
════════════════════════════════ -->
<div class="modal-overlay hidden" id="modal-create">
  <div class="modal">
    <div class="modal-title">Nueva Práctica</div>
    <div class="modal-body">
      <div class="fg"><label>Nombre del proyecto</label><input id="cp-name" placeholder="Ej: Av. Pumapungo · Oct 2025"></div>
      <div class="fg"><label>Avenida / Tramo</label><input id="cp-avenue" placeholder="Ej: Av. Paseo de los Cañaris"></div>
      <div class="g2">
        <div class="fg"><label>Longitud del tramo (m)</label><input id="cp-length" type="number" value="366" min="50"></div>
        <div class="fg"><label>Corridas planeadas (por dir.)</label><input id="cp-runs" type="number" value="12" min="3"></div>
      </div>
      <div class="g2">
        <div class="fg"><label>Dirección 1</label><input id="cp-dir1" value="Norte → Sur"></div>
        <div class="fg"><label>Dirección 2</label><input id="cp-dir2" value="Sur → Norte"></div>
      </div>
      <div class="fg"><label>Fecha de práctica</label><input id="cp-date" type="date"></div>
    </div>
    <div class="modal-actions">
      <button class="btn btn-ghost" onclick="closeModal('modal-create')">Cancelar</button>
      <button class="btn btn-primary" onclick="createProject()">Crear Proyecto →</button>
    </div>
  </div>
</div>

<!-- TOAST -->
<div id="toast" class="toast"></div>

<script>
// Update header photo & name on auth change
import("https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js").then(m => {
  // Already handled in module above via onAuthStateChanged
});

// Patch pp-photo and pp-name when auth state resolves
document.addEventListener('DOMContentLoaded', () => {
  // These get updated when onAuthStateChanged fires in the module
});

// Expose auth state update to module scope
window._updateUserHeader = (user) => {
  if (!user) return;
  const photo = document.getElementById('pp-photo');
  if (photo && user.photoURL) {
    photo.src = user.photoURL;
    photo.style.display = 'block';
  }
  const name = document.getElementById('pp-name');
  if (name) name.textContent = user.displayName || user.email;
};
</script>

<!-- Patch onAuthStateChanged to also update header -->
<script type="module">
import { getAuth, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js";
// The main module already initializes; this just hooks the header update
const auth2 = getAuth();
onAuthStateChanged(auth2, user => {
  if (user && window._updateUserHeader) window._updateUserHeader(user);
});
</script>

</body>
</html>
