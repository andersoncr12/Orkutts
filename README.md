<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Orkut 3.0</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<style>
body{margin:0;font-family:Arial;background:#dbe7ff}
header{background:#3b5998;color:#fff;padding:12px;text-align:center;font-size:22px;font-weight:bold}

/* LOGIN / APP */
#login,#app{display:none;padding:15px;max-width:900px;margin:auto}
.card{background:#fff;border-radius:8px;padding:15px;margin-bottom:10px}

input,textarea,button,select{
width:100%;padding:10px;margin:5px 0;border-radius:6px;border:1px solid #ccc
}
button{background:#3b5998;color:#fff;border:none}

/* CHAT */
#chat{width:100%}
.msg{background:#f1f1f1;padding:8px;border-radius:6px;margin:6px 0}
.msg img,.msg video{max-width:100%;border-radius:6px;margin-top:5px}

/* ONLINE MINIMIZADO */
#onlineBox{
position:fixed;
top:60px;
right:10px;
z-index:999;
}

#toggleOnline{
padding:8px 14px;
border-radius:20px;
font-size:14px;
}

#online{
display:none;
width:180px;
max-height:300px;
overflow-y:auto;
margin-top:5px;
}

#online .user{
display:flex;
align-items:center;
gap:8px;
background:#eaf1ff;
padding:6px;
border-radius:6px;
margin-bottom:5px;
font-size:13px;
}

#online img{
width:32px;
height:32px;
border-radius:50%;
object-fit:contain;
background:#eee;
}

#notif{color:red;font-size:13px}
</style>
</head>

<body>

<header>💙 Orkut 3.0</header>

<!-- LOGIN -->
<div id="login" class="card">
<h3>Entrar</h3>
<input id="nome" placeholder="Seu nome">
<input id="senha" type="password" placeholder="Senha">
<input id="foto" type="file" accept="image/*">
<button onclick="entrar()">Entrar</button>
</div>

<!-- APP -->
<div id="app">

<!-- ONLINE MINIMIZADO -->
<div id="onlineBox">
<button id="toggleOnline" onclick="toggleOnline()">🟢 Online</button>
<div id="online" class="card">
<h4>Online agora</h4>
<div id="listaOnline"></div>
</div>
</div>

<!-- CHAT -->
<div id="chat" class="card">
<h4>Chat <span id="notif"></span></h4>
<select id="destino"></select>
<div id="msgs"></div>
<textarea id="texto" placeholder="Mensagem"></textarea>
<input type="file" id="file" accept="image/*,video/*">
<button onclick="enviar()">Enviar</button>
<button onclick="sair()">Sair</button>
</div>

</div>

<!-- FIREBASE -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-storage-compat.js"></script>

<script>
const firebaseConfig={
apiKey:"AIzaSyCUb2VNq9AETlzhE4w0eSwRPA4yBzrVC68",
authDomain:"meu-50495.firebaseapp.com",
projectId:"meu-50495",
storageBucket:"meu-50495.firebasestorage.app",
messagingSenderId:"941036013080",
appId:"1:941036013080:web:61281776f379d87867a56d"
};

firebase.initializeApp(firebaseConfig);
const auth=firebase.auth();
const db=firebase.firestore();
const storage=firebase.storage();

let me="",chatUser="";

/* LOGIN */
auth.onAuthStateChanged(u=>{
if(u){
login.style.display="none";
app.style.display="block";
me=u.uid;
online();
usuarios();
mensagens();
}else{
login.style.display="block";
app.style.display="none";
}
});

function entrar(){
const n=nome.value.trim().toLowerCase();
const s=senha.value;
if(!n||!s)return alert("Preencha tudo");

const email=n+"@orkut.local";

auth.signInWithEmailAndPassword(email,s)
.catch(()=>{
auth.createUserWithEmailAndPassword(email,s).then(c=>{
const uid=c.user.uid;
const f=foto.files[0];
if(f){
storage.ref("fotos/"+uid).put(f).then(r=>{
r.ref.getDownloadURL().then(url=>{
db.collection("users").doc(uid).set({nome:n.split(" ")[0],foto:url});
});
});
}else{
db.collection("users").doc(uid).set({nome:n.split(" ")[0],foto:""});
}
});
});
}

/* ONLINE */
function online(){
db.collection("online").doc(me).set({t:Date.now()});
db.collection("users").doc(me).get().then(d=>{
db.collection("online").doc(me).update(d.data());
});

window.onbeforeunload=()=>db.collection("online").doc(me).delete();

db.collection("online").onSnapshot(s=>{
listaOnline.innerHTML="";
s.forEach(d=>{
listaOnline.innerHTML+=`
<div class="user">
<img src="${d.data().foto||'https://via.placeholder.com/40'}">
${d.data().nome}
</div>`;
});
});
}

function toggleOnline(){
online.style.display =
online.style.display==="block" ? "none" : "block";
}

/* USUÁRIOS */
function usuarios(){
db.collection("users").onSnapshot(s=>{
destino.innerHTML="";
s.forEach(d=>{
if(d.id!==me)
destino.innerHTML+=`<option value="${d.id}">${d.data().nome}</option>`;
});
});
}

/* CHAT */
function enviar(){
chatUser=destino.value;
const f=file.files[0];
const t=texto.value;
if(!f&&!t)return;

if(f){
storage.ref("midia/"+Date.now()+f.name).put(f).then(r=>{
r.ref.getDownloadURL().then(url=>{
salvar({media:url,type:f.type,text:t});
});
});
}else salvar({text:t});

texto.value="";
file.value="";
}

function salvar(o){
db.collection("msgs").add({from:me,to:chatUser,...o,t:Date.now()});
}

function mensagens(){
db.collection("msgs").orderBy("t").onSnapshot(s=>{
msgs.innerHTML="";
s.forEach(d=>{
const m=d.data();
if(m.from===me||m.to===me){
let html=m.text||"";
if(m.media){
if(m.type.includes("image"))html+=`<img src="${m.media}">`;
if(m.type.includes("video"))html+=`<video controls src="${m.media}"></video>`;
}
msgs.innerHTML+=`<div class="msg">${html}</div>`;
if(m.to===me){
notif.innerText="Nova mensagem!";
setTimeout(()=>notif.innerText="",2000);
}
}
});
});
}

function sair(){
db.collection("online").doc(me).delete();
auth.signOut();
}
</script>

</body>
</html>
 # Orkutts
