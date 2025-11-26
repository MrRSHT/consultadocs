# README

Proyecto: ConsultaDocs — Aplicación web para realizar consultas sobre documentos PDF alojados en URLs administradas por un panel de administración.

Descripción general
- Los usuarios pueden realizar consultas sin necesidad de registro.
- Las respuestas se generan a partir de los documentos PDF registrados por el administrador.
- El administrador puede ingresar con credenciales (usuario: **RshMaule**, clave: **Maule2025@**) y agregar, editar o eliminar URLs de documentos.
- El sistema descarga los PDFs, extrae el texto y permite búsquedas avanzadas.
- Si se agrega la variable de entorno `OPENAI_API_KEY`, el sistema puede generar respuestas mejoradas utilizando IA.

Estructura del proyecto:
- package.json
- server.js
- data.json
- /public/index.html (interfaz del usuario)
- /public/admin.html (panel administrador)
- /public/main.js
- /public/admin.js
- .gitignore

A continuación se incluye el código completo del proyecto:

--- package.json ---
{
  "name": "consultadocs",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "node-fetch": "^2.6.7",
    "pdf-parse": "^1.1.1",
    "lunr": "^2.3.9",
    "body-parser": "^1.20.2",
    "express-session": "^1.17.3",
    "cors": "^2.8.5",
    "morgan": "^1.10.0"
  }
}

--- .gitignore ---
node_modules/
documents/
data.json
.env

--- data.json ---
{
  "documents": []
}

--- server.js ---
const express = require('express');
const fetch = require('node-fetch');
const fs = require('fs');
const path = require('path');
const pdf = require('pdf-parse');
const lunr = require('lunr');
const bodyParser = require('body-parser');
const session = require('express-session');
const cors = require('cors');
const morgan = require('morgan');

const APP_PORT = process.env.PORT || 3000;
const ADMIN_USER = process.env.ADMIN_USER || 'RshMaule';
const ADMIN_PASS = process.env.ADMIN_PASS || 'Maule2025@';
const OPENAI_KEY = process.env.OPENAI_API_KEY || '';

const DATA_FILE = path.join(__dirname, 'data.json');
const DOCS_DIR = path.join(__dirname, 'documents');
if (!fs.existsSync(DOCS_DIR)) fs.mkdirSync(DOCS_DIR);
if (!fs.existsSync(DATA_FILE)) fs.writeFileSync(DATA_FILE, JSON.stringify({documents:[]}, null, 2));

let rawData = JSON.parse(fs.readFileSync(DATA_FILE));
let documents = rawData.documents; // [{id, url, title, file, text, updated}]
let idx = null;

function saveData(){
  fs.writeFileSync(DATA_FILE, JSON.stringify({documents}, null, 2));
}

async function fetchAndParsePDF(url, id){
  try{
    const res = await fetch(url);
    if (!res.ok) throw new Error('Failed to fetch');
    const buffer = await res.buffer();
    const data = await pdf(buffer);
    const text = (data && data.text) ? data.text.replace(/\s+/g, ' ') : '';
    const filename = path.join(DOCS_DIR, `${id}.pdf`);
    fs.writeFileSync(filename, buffer);
    return {text, file: filename};
  }catch(e){
    console.error('PDF fetch/parse error', e);
    throw e;
  }
}

function buildIndex(){
  idx = lunr(function(){n
    this.ref('id');
    this.field('title');
    this.field('text');
    const that = this;
    documents.forEach(doc => {
      that.add({id: doc.id, title: doc.title || '', text: doc.text || ''});
    });
  });
}

// small helper to create snippet
function snippet(text, query, radius=120){
  const i = text.toLowerCase().indexOf(query.toLowerCase());
  if (i === -1) return text.slice(0, radius) + (text.length>radius ? '...' : '');
  const start = Math.max(0, i - Math.floor(radius/2));
  const s = text.slice(start, start + radius);
  return (start>0? '...':'') + s + (start + radius < text.length ? '...' : '');
}

// Build initial index
buildIndex();

const app = express();
app.use(morgan('dev'));
app.use(cors());
app.use(bodyParser.json({limit: '10mb'}));
app.use(bodyParser.urlencoded({extended:true}));
app.use(session({secret:'consultadocs-secret', resave:false, saveUninitialized:true}));
app.use(express.static(path.join(__dirname, 'public')));

// Admin login
app.post('/api/admin/login', (req, res) => {
  const {user, pass} = req.body;
  if (user === ADMIN_USER && pass === ADMIN_PASS){
    req.session.admin = true;
    return res.json({ok:true});
  }
  return res.status(401).json({ok:false, error:'credenciales inválidas'});
});

app.post('/api/admin/logout', (req,res)=>{ req.session.destroy(()=>res.json({ok:true}));});

function requireAdmin(req, res, next){
  if (req.session && req.session.admin) return next();
  return res.status(403).json({ok:false, error:'no autorizado'});
}

// Add document URL
app.post('/api/admin/add', requireAdmin, async (req, res) => {
  const {url, title} = req.body;
  if (!url) return res.status(400).json({ok:false, error:'url requerida'});
  const id = 'doc_' + Date.now();
  try{
    const parsed = await fetchAndParsePDF(url, id);
    const doc = {id, url, title: title || url, file: parsed.file, text: parsed.text, updated: new Date().toISOString()};
    documents.push(doc);
    saveData();
    buildIndex();
    return res.json({ok:true, doc});
  }catch(e){
    return res.status(500).json({ok:false, error: e.message});
  }
});

// List docs
app.get('/api/admin/list', requireAdmin, (req,res)=>{
  res.json({ok:true, documents: documents.map(d=>({id:d.id, title:d.title, url:d.url, updated:d.updated}))});
});

// Remove doc
app.post('/api/admin/remove', requireAdmin, (req,res)=>{
  const {id} = req.body;
  documents = documents.filter(d=>d.id !== id);
  saveData();
  buildIndex();
  res.json({ok:true});
});

// Search (public)
app.post('/api/search', async (req, res) => {
  const {q, topk} = req.body;
  if (!q) return res.status(400).json({ok:false, error:'q requerida'});
  try{
    if (!idx) buildIndex();
    const results = idx.search(q + '*').slice(0, topk || 5);
    const out = results.map(r=>{
      const d = documents.find(x=>x.id === r.ref);
      return {id:d.id, title:d.title, url:d.url, score: r.score, snippet: snippet(d.text || '', q)};
    });

    // Optional: if OPENAI_KEY provided, call OpenAI to generate a concise answer (this part is optional and uses fetch)
    if (OPENAI_KEY && out.length>0){
      // Form a prompt using top snippets
      const prompt = `Eres un asistente. Responde brevemente a la consulta: "${q}" usando exclusivamente la información de estos extractos:\n\n` + out.map(o=>`- ${o.title}: ${o.snippet}`).join('\n\n') + '\n\nRespuesta:';
      try{
        const openaiRes = await fetch('https://api.openai.com/v1/chat/completions',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+OPENAI_KEY},body: JSON.stringify({model:'gpt-4o-mini',messages:[{role:'user',content:prompt}],max_tokens:300})});
        const j = await openaiRes.json();
        const text = j.choices && j.choices[0] && j.choices[0].message ? j.choices[0].message.content : ''; 
        return res.json({ok:true, results: out, answer: text});
      }catch(e){
        console.warn('openai call failed', e);
        return res.json({ok:true, results: out});
      }
    }

    return res.json({ok:true, results: out});
  }catch(e){
    console.error(e);
    res.status(500).json({ok:false, error:e.message});
  }
});

app.listen(APP_PORT, ()=>console.log('Server started on port', APP_PORT));

--- public/index.html ---
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>ConsultaDocs - Buscador</title>
  <style>
    body{font-family: Arial, sans-serif; max-width:800px; margin:40px auto; padding:0 10px}
    input, button{padding:8px; font-size:16px}
    .result{border-bottom:1px solid #ddd; padding:10px 0}
    .title{font-weight:600}
  </style>
</head>
<body>
  <h1>ConsultaDocs</h1>
  <p>Escribe tu consulta y obtén respuestas basadas en los documentos registrados por el administrador.</p>
  <input id="q" placeholder="Escribe tu pregunta" style="width:70%" />
  <button id="search">Buscar</button>
  <div id="results"></div>

  <script src="/main.js"></script>
</body>
</html>

--- public/admin.html ---
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>ConsultaDocs - Admin</title>
  <style>body{font-family: Arial; max-width:900px; margin:20px auto}</style>
</head>
<body>
  <h1>Admin — ConsultaDocs</h1>
  <div id="login">
    <h3>Ingreso Administrador</h3>
    <input id="user" placeholder="Usuario" value="RshMaule" />
    <input id="pass" placeholder="Clave" value="Maule2025@" type="password" />
    <button id="loginBtn">Entrar</button>
  </div>

  <div id="panel" style="display:none">
    <h3>Agregar Documento (URL de PDF)</h3>
    <input id="url" placeholder="https://...archivo.pdf" style="width:60%" />
    <input id="title" placeholder="Título (opcional)" />
    <button id="add">Agregar</button>

    <h3>Listado de Documentos</h3>
    <div id="docs"></div>
    <button id="logout">Cerrar Sesión</button>
  </div>

  <script src="/admin.js"></script>
</body>
</html>

--- public/main.js ---
async function doSearch(){
  const q = document.getElementById('q').value;
  if (!q) return alert('Escribe algo');
  const res = await fetch('/api/search', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({q, topk:5})});
  const j = await res.json();
  const cont = document.getElementById('results'); cont.innerHTML='';
  if (!j.ok) return cont.innerText = 'Error: ' + (j.error||'');
  if (j.answer) {
    const a = document.createElement('div'); a.className='answer'; a.innerHTML = '<h3>Respuesta (generada)</h3><div>'+j.answer+'</div>'; cont.appendChild(a);
  }
  j.results.forEach(r=>{
    const d = document.createElement('div'); d.className='result';
    d.innerHTML = `<div class="title">${r.title}</div><div>${r.snippet}</div><div style="font-size:12px;color:#666">Fuente: ${r.url}</div>`;
    cont.appendChild(d);
  });
}

document.getElementById('search').addEventListener('click', doSearch);

--- public/admin.js ---
async function login(){
  const user = document.getElementById('user').value;
  const pass = document.getElementById('pass').value;
  const res = await fetch('/api/admin/login',{method:'POST',headers:{'Content-Type':'application/json'},body: JSON.stringify({user, pass})});
  if (res.ok){ document.getElementById('login').style.display='none'; document.getElementById('panel').style.display='block'; loadDocs(); }
  else alert('Credenciales inválidas');
}
async function loadDocs(){
  const res = await fetch('/api/admin/list');
  const j = await res.json();
  if (!j.ok) return alert('No autorizado');
  const el = document.getElementById('docs'); el.innerHTML='';
  j.documents.forEach(d=>{
    const div = document.createElement('div'); div.innerHTML = `<b>${d.title}</b> <span style="color:#666">(${d.url})</span> <button onclick="remove('${d.id}')">Eliminar</button>`;
    el.appendChild(div);
  });
}
async function add(){
  const url = document.getElementById('url').value;
  const title = document.getElementById('title').value;
  const res = await fetch('/api/admin/add',{method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({url, title})});
  const j = await res.json();
  if (!j.ok) return alert('Error: '+(j.error||'')); alert('Documento agregado'); loadDocs();
}
async function remove(id){
  if (!confirm('Eliminar?')) return;
  const res = await fetch('/api/admin/remove',{method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({id})});
  const j = await res.json();
  if (!j.ok) return alert('Error'); loadDocs();
}

document.getElementById('loginBtn').addEventListener('click', login);
document.getElementById('add').addEventListener('click', add);
document.getElementById('logout').addEventListener('click', async ()=>{ await fetch('/api/admin/logout',{method:'POST'}); location.reload(); });

--- FIN DEL PROYECTO ---
