<!DOCTYPE html>
<html lang="fr"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes"><meta name="robots" content="noindex">
<title>Mon journal de trading</title>
<style>
:root{--bg:#f6f5f1;--card:#fff;--tx:#1d1d1b;--mu:#76746d;--bd:#e4e2da;--g:#0f8a5f;--r:#c8372d;--ac:#b8860b;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#141413;--card:#1e1e1c;--tx:#efede6;--mu:#9a978d;--bd:#33322e;--g:#3ccf91;--r:#ff6b5e;--ac:#e0b040}}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--tx);font:15px/1.4 system-ui,-apple-system,sans-serif;padding:16px;max-width:900px;margin:auto}
h1{font:600 22px Georgia,serif;margin:0 0 10px}h2{font-size:13px;margin:0 0 10px;color:var(--mu);text-transform:uppercase;letter-spacing:.04em}
.card{background:var(--card);border:1px solid var(--bd);border-radius:12px;padding:14px;margin-bottom:12px}
.grid{display:grid;grid-template-columns:repeat(2,1fr);gap:10px}@media(min-width:600px){.grid{grid-template-columns:repeat(4,1fr)}}
.k{background:var(--card);border:1px solid var(--bd);border-radius:12px;padding:12px}.k small{color:var(--mu);display:block;font-size:12px}.k b{font-size:20px}
.g{color:var(--g)}.r{color:var(--r)}input,select{width:100%;background:var(--bg);color:var(--tx);border:1px solid var(--bd);border-radius:8px;padding:10px;font:inherit;margin:4px 0 10px}
button{background:var(--ac);color:#fff;border:0;border-radius:8px;padding:10px 16px;font:inherit;cursor:pointer}button.s{background:transparent;color:var(--mu);border:1px solid var(--bd)}
label{font-size:13px;color:var(--mu)}.row{display:flex;gap:8px;flex-wrap:wrap;align-items:center;margin-bottom:12px}.row select{width:auto;margin:0}
.sc{overflow-x:auto}table{border-collapse:collapse;width:100%;font-size:13px;white-space:nowrap}th,td{padding:6px 8px;text-align:right;border-bottom:1px solid var(--bd)}th:first-child,td:first-child{text-align:left}th{color:var(--mu)}
canvas{width:100%;height:200px;display:block}.bar{display:flex;align-items:center;gap:8px;margin:5px 0;font-size:13px}.bar i{display:block;height:12px;border-radius:3px}.bar span:first-child{width:34px;color:var(--mu)}
.n{color:var(--mu);font-size:13px}.err{color:var(--r);font-size:13px}
</style></head><body>
<h1>Mon journal de trading</h1>
<div class="row"><button id="sync">Synchroniser</button><button class="s" id="cfg">Réglages</button><select id="sym"></select><select id="per"><option value="0">Tout</option><option value="1">Aujourd’hui</option><option value="7">7 jours</option><option value="30">30 jours</option></select></div>
<p class="n" id="msg"></p>
<div class="card" id="set" hidden><h2>Connexion MetaApi</h2>
<label>Jeton d’accès (token)</label><input id="tok" type="password" autocomplete="off">
<label>Identifiant du compte</label><input id="acc" autocomplete="off">
<label>Région du compte</label><select id="reg" style="width:100%;margin:4px 0 10px"><option>new-york</option><option>london</option><option>singapore</option></select>
<label>Importer l’historique depuis le</label><input id="since" type="date">
<button id="saveb">Enregistrer</button><button class="s" id="wipe" style="margin-left:8px">Effacer les données</button>
<p class="n">Ces informations restent uniquement sur ton téléphone.</p></div>
<div id="out"></div>
<script>
const $=s=>document.querySelector(s),LS=(k,v)=>{try{if(v===undefined)return JSON.parse(localStorage.getItem(k));localStorage.setItem(k,JSON.stringify(v))}catch(e){return null}};
let D=LS('deals')||[],C=LS('cfg')||{reg:'new-york'};
const fmt=n=>(n>=0?'+':'')+n.toFixed(2);
function setup(){$('#tok').value=C.tok||'';$('#acc').value=C.acc||'';$('#reg').value=C.reg||'new-york';$('#since').value=C.since||new Date(Date.now()-90*864e5).toISOString().slice(0,10);if(!C.tok||!C.acc)$('#set').hidden=false}
$('#cfg').onclick=()=>$('#set').hidden=!$('#set').hidden;
$('#saveb').onclick=()=>{C={tok:$('#tok').value.trim(),acc:$('#acc').value.trim(),reg:$('#reg').value,since:$('#since').value};LS('cfg',C);$('#set').hidden=true;sync()};
$('#wipe').onclick=()=>{if(confirm('Effacer les données locales du journal ?')){D=[];LS('deals',D);LS('last',null);render()}};
$('#sync').onclick=sync;$('#sym').onchange=$('#per').onchange=()=>render();
async function sync(){
 if(!C.tok||!C.acc){$('#set').hidden=false;return}
 const m=$('#msg');m.className='n';m.textContent='Synchronisation…';
 const base=`https://mt-client-api-v1.${C.reg}.agiliumtrade.ai/users/current/accounts/${C.acc}/history-deals/time/`;
 const last=LS('last');const start=last?new Date(new Date(last).getTime()-2*864e5):new Date((C.since||'2020-01-01')+'T00:00:00Z');
 const end=new Date(Date.now()+864e5);let off=0,add=0;
 try{for(;;){
  const r=await fetch(`${base}${start.toISOString()}/${end.toISOString()}?offset=${off}&limit=1000`,{headers:{'auth-token':C.tok,Accept:'application/json'}});
  if(!r.ok)throw new Error(r.status===401?'Jeton refusé (401). Vérifie ton token.':r.status===404?'Compte introuvable ou pas encore déployé (404).':'Erreur MetaApi '+r.status);
  const a=await r.json();
  for(const d of a){if(!/^DEAL_TYPE_(BUY|SELL)$/.test(d.type))continue;
   if(!D.some(x=>x.i===d.id)){D.push({i:d.id,p:d.positionId,e:d.entryType,t:d.type,s:d.symbol,v:d.volume,pr:d.price,pf:d.profit||0,c:d.commission||0,sw:d.swap||0,tm:d.time});add++}}
  if(a.length<1000)break;off+=1000}
  LS('deals',D);LS('last',new Date().toISOString());m.textContent=add+' nouveau(x) deal(s) importé(s). Dernière synchro : '+new Date().toLocaleString('fr-FR',{dateStyle:'short',timeStyle:'short'})}
 catch(e){m.className='err';m.textContent=/Failed to fetch|NetworkError|Load failed/i.test(e.message)?'Connexion impossible : vérifie la région du compte, ta connexion, ou dis-moi si le navigateur bloque l’appel (CORS).':e.message}
 render()}
function trades(){const P={};
 for(const d of D){const p=P[d.p]||(P[d.p]={id:d.p,sym:d.s,vol:0,com:0,sw:0,pr:0,to:0,tc:0,po:0,pc:0,type:'',out:0});
  p.com+=d.c;p.sw+=d.sw;p.pr+=d.pf;const t=new Date(d.tm).getTime();
  if(d.e==='DEAL_ENTRY_IN'){p.to=t;p.po=d.pr;p.vol=d.v;p.type=d.t==='DEAL_TYPE_BUY'?'buy':'sell'}
  else{p.out=1;p.tc=Math.max(p.tc,t);p.pc=d.pr;if(!p.type)p.type=d.t==='DEAL_TYPE_BUY'?'sell':'buy';if(!p.vol)p.vol=d.v;if(!p.to)p.to=t}}
 return Object.values(P).filter(p=>p.out)}
function render(){
 const sy=$('#sym'),cur=sy.value,all=trades(),syms=[...new Set(all.map(t=>t.sym))];
 sy.innerHTML='<option value="">Tous les symboles</option>'+syms.map(s=>`<option ${s===cur||(!cur&&s.startsWith('XAUUSD'))?'selected':''}>${s}</option>`).join('');
 const S=sy.value,P=+$('#per').value,lim=P?Date.now()-P*864e5:0;
 const L=all.filter(t=>(!S||t.sym===S)&&t.tc>=lim).sort((a,b)=>a.tc-b.tc).map(t=>({...t,net:t.pr+t.com+t.sw}));
 const o=$('#out');if(!L.length){o.innerHTML='<div class="card n">Aucune position. Appuie sur « Synchroniser ».</div>';return}
 const W=L.filter(t=>t.net>0),Ls=L.filter(t=>t.net<0),gp=W.reduce((a,t)=>a+t.net,0),gl=-Ls.reduce((a,t)=>a+t.net,0),net=gp-gl;
 let eq=0,pk=0,dd=0;const curve=L.map(t=>{eq+=t.net;pk=Math.max(pk,eq);dd=Math.max(dd,pk-eq);return eq});
 const wr=W.length/L.length*100,aw=W.length?gp/W.length:0,al=Ls.length?gl/Ls.length:0;
 const K=(a,b,c)=>`<div class="k"><small>${a}</small><b class="${c||''}">${b}</b></div>`;
 const days=['Dim','Lun','Mar','Mer','Jeu','Ven','Sam'],dw=days.map(()=>0);L.forEach(t=>dw[new Date(t.tc).getDay()]+=t.net);const mx=Math.max(...dw.map(Math.abs))||1;
 const side=k=>{const a=L.filter(t=>t.type===k);return a.length?`${a.length} trades · ${(a.filter(t=>t.net>0).length/a.length*100).toFixed(0)} % · ${fmt(a.reduce((s,t)=>s+t.net,0))}`:'–'};
 o.innerHTML=`<div class="grid">${K('Profit net',fmt(net),net>=0?'g':'r')}${K('Winrate',wr.toFixed(1)+' %')}${K('Trades',L.length+' ('+W.length+'G / '+Ls.length+'P)')}${K('Profit factor',gl?(gp/gl).toFixed(2):'∞')}${K('Gain moyen',fmt(aw),'g')}${K('Perte moyenne',fmt(-al),'r')}${K('Espérance / trade',fmt(net/L.length),net>=0?'g':'r')}${K('Drawdown max','-'+dd.toFixed(2),'r')}</div>
 <div class="card" style="margin-top:12px"><h2>Courbe de capital</h2><canvas id="cv"></canvas></div>
 <div class="card"><h2>Profit par jour</h2>${[1,2,3,4,5,6,0].map(i=>`<div class="bar"><span>${days[i]}</span><i style="width:${Math.abs(dw[i])/mx*60}%;background:var(--${dw[i]>=0?'g':'r'})"></i><span style="width:auto;color:var(--tx)">${fmt(dw[i])}</span></div>`).join('')}</div>
 <div class="card"><h2>Achats vs ventes</h2><p style="margin:4px 0">Buy : ${side('buy')}</p><p style="margin:4px 0">Sell : ${side('sell')}</p></div>
 <div class="card"><h2>Positions</h2><div class="sc"><table><tr><th>Clôture</th><th>Type</th><th>Lots</th><th>Entrée</th><th>Sortie</th><th>Net</th></tr>${[...L].reverse().map(t=>`<tr><td>${new Date(t.tc).toLocaleString('fr-FR',{dateStyle:'short',timeStyle:'short'})}</td><td>${t.type}</td><td>${t.vol}</td><td>${t.po}</td><td>${t.pc}</td><td class="${t.net>=0?'g':'r'}">${fmt(t.net)}</td></tr>`).join('')}</table></div></div>`;
 const c=$('#cv'),r=devicePixelRatio||1,w=c.clientWidth,h=200;c.width=w*r;c.height=h*r;const x=c.getContext('2d');x.scale(r,r);
 const cs=getComputedStyle(document.documentElement),v=[0,...curve],mn=Math.min(...v),M=Math.max(...v),sp=(M-mn)||1;
 const X=i=>10+i/(v.length-1||1)*(w-20),Y=q=>h-14-(q-mn)/sp*(h-28);
 x.strokeStyle=cs.getPropertyValue('--bd');x.beginPath();x.moveTo(0,Y(0));x.lineTo(w,Y(0));x.stroke();
 x.strokeStyle=cs.getPropertyValue(net>=0?'--g':'--r');x.lineWidth=2;x.beginPath();v.forEach((q,i)=>i?x.lineTo(X(i),Y(q)):x.moveTo(X(i),Y(q)));x.stroke()}
setup();render();if(C.tok&&C.acc)sync();
</script></body></html>
