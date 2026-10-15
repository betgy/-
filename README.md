<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<title>Мои смены</title>
<style>
:root{--blue:#2563eb;--bg:#f2f4f7;--card:#fff;--text:#111827;--muted:#6b7280;--line:#e5e7eb;--danger:#dc2626}
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
body{margin:0;background:var(--bg);color:var(--text);font-family:-apple-system,BlinkMacSystemFont,"SF Pro Text",Arial,sans-serif;padding-bottom:24px}
.wrap{max-width:520px;margin:0 auto;padding:16px 14px}
h1{font-size:22px;margin:6px 0 14px}
.card{background:var(--card);border-radius:16px;padding:16px;box-shadow:0 4px 14px rgba(15,23,42,.07);margin-bottom:14px}
h2{font-size:17px;margin:0 0 12px}
label{display:block;font-size:13px;color:var(--muted);margin:10px 0 6px;font-weight:600}
input{width:100%;padding:14px;border:1px solid var(--line);border-radius:12px;font-size:16px;background:#fff;min-height:48px}
input:focus{outline:none;border-color:var(--blue);box-shadow:0 0 0 3px rgba(37,99,235,.15)}
.grid{display:grid;grid-template-columns:1fr 1fr;gap:10px}
.full{grid-column:1/-1}
button{width:100%;min-height:50px;border:0;border-radius:14px;font-size:17px;font-weight:700;color:#fff;background:var(--blue);margin-top:14px;cursor:pointer}
button:active{transform:scale(.98)}
button.ghost{background:#eef2ff;color:var(--blue)}
.summary{position:sticky;top:0;z-index:5;background:linear-gradient(135deg,#2563eb,#1d4ed8);color:#fff;border-radius:18px;padding:16px;margin-bottom:14px;box-shadow:0 6px 18px rgba(37,99,235,.25)}
.summary .label{font-size:13px;opacity:.85}
.summary .amount{font-size:30px;font-weight:800;margin:4px 0 10px}
.stats{display:grid;grid-template-columns:1fr 1fr;gap:8px}
.stat{background:rgba(255,255,255,.16);border-radius:12px;padding:8px 10px}
.stat b{display:block;font-size:17px}
.stat span{font-size:12px;opacity:.85}
.shift{background:#fff;border-radius:16px;padding:14px;margin-bottom:10px;box-shadow:0 2px 8px rgba(15,23,42,.06);cursor:pointer}
.shift:active{background:#f8fafc}
.shift-top{display:flex;justify-content:space-between;align-items:start;gap:10px}
.date{font-weight:800;font-size:16px}
.weekday{font-size:13px;color:var(--muted);font-weight:600;margin-top:2px}
.pay{font-weight:800;color:#16a34a;font-size:17px;white-space:nowrap}
.address{margin-top:4px;color:var(--muted);font-size:14px}
.meta{display:flex;flex-wrap:wrap;gap:6px;margin-top:10px}
.chip{background:#f3f4f6;border-radius:999px;padding:6px 10px;font-size:12px;color:#374151}
.edit-hint{font-size:12px;color:var(--muted);margin-top:8px}
.empty{text-align:center;color:var(--muted);padding:26px 10px;background:#fff;border-radius:16px}
.actions{display:grid;grid-template-columns:1fr 1fr;gap:10px}
</style>
</head>
<body>
<div class="wrap">
  <h1>Мои смены</h1>

  <div class="summary">
    <div class="label">Зарплата за месяц</div>
    <div class="amount" id="total">0 ₽</div>
    <div class="stats">
      <div class="stat"><b id="count">0</b><span>смен</span></div>
      <div class="stat"><b id="hours">0</b><span>часов</span></div>
    </div>
  </div>

  <div class="card">
    <h2 id="formTitle">Добавить смену</h2>
    <div class="grid">
      <div class="full"><label>Дата</label><input type="date" id="date" onchange="showWeekday()"><div id="dateWeekday" class="weekday"></div></div>
      <div class="full"><label>Адрес</label><input id="address" placeholder="Улица, дом" autocomplete="off"></div>
      <div><label>Ставка, ₽/час</label><input type="number" id="rate" inputmode="decimal" min="0" step="0.01" placeholder="0"></div>
      <div><label>Оплата, ₽</label><input type="number" id="pay" inputmode="decimal" min="0" step="0.01" placeholder="авто"></div>
      <div><label>Начало</label><input type="time" id="start"></div>
      <div><label>Конец</label><input type="time" id="end"></div>
    </div>
    <div class="actions">
      <button id="saveBtn" onclick="saveShift()">Добавить</button>
      <button id="cancelBtn" class="ghost" onclick="cancelEdit()" style="display:none">Отмена</button>
    </div>
    <p style="font-size:12px;color:var(--muted);margin:10px 0 0">Если не указать оплату, она рассчитается: ставка × часы.</p>
  </div>

  <div class="card">
    <h2>Период</h2>
    <label>Месяц</label>
    <input type="month" id="month" onchange="render()">
  </div>

  <h2 style="margin:18px 4px 10px">Смены</h2>
  <div id="list"></div>
</div>

<script>
let shifts=JSON.parse(localStorage.getItem('shifts')||'[]');
let editingId=null;
const today=new Date().toISOString().slice(0,10);
const date=document.getElementById('date'),address_=document.getElementById('address'),rate_=document.getElementById('rate'),pay_=document.getElementById('pay'),start_=document.getElementById('start'),end_=document.getElementById('end'),month_=document.getElementById('month');
date.value=today;month_.value=today.slice(0,7);

const fmt=n=>new Intl.NumberFormat('ru-RU',{style:'currency',currency:'RUB',maximumFractionDigits:2}).format(n||0);
const weekdays=['воскресенье','понедельник','вторник','среда','четверг','пятница','суббота'];
function weekday(d){const dt=new Date(d+'T00:00:00');return isNaN(dt)?'':weekdays[dt.getDay()];}
function showWeekday(){document.getElementById('dateWeekday').textContent=weekday(date.value);}

function save(){localStorage.setItem('shifts',JSON.stringify(shifts));render();}

function hoursBetween(a,b){
  if(!a||!b)return 0;
  let [h1,m1]=a.split(':').map(Number),[h2,m2]=b.split(':').map(Number);
  let m=(h2*60+m2)-(h1*60+m1);
  if(m<0)m+=1440;
  return m/60;
}

function saveShift(){
  const d=date.value,address=address_.value.trim(),rate=parseFloat(rate_.value)||0,start=start_.value,end=end_.value;
  if(!d){alert('Выберите дату');return;}
  const hours=hoursBetween(start,end);
  let pay=pay_.value===''?+(rate*hours).toFixed(2):parseFloat(pay_.value);
  if(editingId){
    const s=shifts.find(x=>x.id===editingId);
    Object.assign(s,{date:d,address,rate,start,end,hours:+hours.toFixed(2),pay});
    cancelEdit();
  }else{
    shifts.push({id:Date.now(),date:d,address,rate,start,end,hours:+hours.toFixed(2),pay});
    clearForm();
  }
  save();window.scrollTo({top:0,behavior:'smooth'});
}

function editShift(id){
  const s=shifts.find(x=>x.id===id);
  if(!s)return;
  editingId=id;
  date.value=s.date;address_.value=s.address||'';rate_.value=s.rate||'';pay_.value=s.pay??'';start_.value=s.start||'';end_.value=s.end||'';
  showWeekday();
  document.getElementById('formTitle').textContent='Редактировать смену';
  document.getElementById('saveBtn').textContent='Сохранить';
  document.getElementById('cancelBtn').style.display='block';
  document.querySelector('.card').scrollIntoView({behavior:'smooth',block:'start'});
}

function cancelEdit(){
  editingId=null;clearForm();
  document.getElementById('formTitle').textContent='Добавить смену';
  document.getElementById('saveBtn').textContent='Добавить';
  document.getElementById('cancelBtn').style.display='none';
}

function clearForm(){
  address_.value='';rate_.value='';pay_.value='';start_.value='';end_.value='';
  date.value=today;showWeekday();
}

function del(id,event){
  event.stopPropagation();
  if(confirm('Удалить эту смену?')){shifts=shifts.filter(s=>s.id!==id);if(editingId===id)cancelEdit();save();}
}

function render(){
  const month=month_.value;
  const list=shifts.filter(s=>s.date.slice(0,7)===month).sort((a,b)=>b.date.localeCompare(a.date));
  let total=0,hours=0;
  const box=document.getElementById('list');
  box.innerHTML='';
  if(!list.length){box.innerHTML='<div class="empty">За этот месяц смен пока нет</div>';}
  list.forEach(s=>{
    total+=s.pay;hours+=s.hours;
    const d=s.date.split('-').reverse().join('.');
    const wd=weekday(s.date);
    box.innerHTML+=`<div class="shift" onclick="editShift(${s.id})">
      <div class="shift-top">
        <div>
          <div class="date">${d}</div>
          <div class="weekday">${wd}</div>
          <div class="address">${s.address||'Адрес не указан'}</div>
        </div>
        <div class="pay">${fmt(s.pay)}</div>
      </div>
      <div class="meta">
        <span class="chip">🕐 ${s.start||'—'}–${s.end||'—'}</span>
        <span class="chip">⏱ ${s.hours} ч</span>
        ${s.rate?`<span class="chip">₽ ${s.rate}/час</span>`:''}
        <button class="chip" style="border:0;color:var(--danger);font-weight:700" onclick="del(${s.id},event)">Удалить</button>
      </div>
      <div class="edit-hint">Нажмите на смену, чтобы изменить данные</div>
    </div>`;
  });
  document.getElementById('total').textContent=fmt(total);
  document.getElementById('count').textContent=list.length;
  document.getElementById('hours').textContent=hours.toFixed(2);
}

showWeekday();
render();
</script>
</body>
</html>
