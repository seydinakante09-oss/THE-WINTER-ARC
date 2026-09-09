const KEY="habitquest-v1";
const todayKey=()=>new Date().toISOString().slice(0,10);
const defaultData=()=>({
  habits:[
    {id:1,name:"Drink water",category:"Health",xp:15},
    {id:2,name:"Read for 20 minutes",category:"Learning",xp:20},
    {id:3,name:"Plan tomorrow",category:"Productivity",xp:15},
    {id:4,name:"Write 3 things I'm grateful for",category:"Mindfulness",xp:20}
  ],
  completions:{},
  xp:0
});
let data=JSON.parse(localStorage.getItem(KEY)||"null")||defaultData();
function save(){localStorage.setItem(KEY,JSON.stringify(data))}
function dateKey(d=new Date()){return d.toISOString().slice(0,10)}
function done(id,key=todayKey()){return !!(data.completions[key]||[]).includes(id)}
function toggle(id){
  const k=todayKey(); data.completions[k] ||= [];
  const arr=data.completions[k], i=arr.indexOf(id), habit=data.habits.find(h=>h.id===id);
  if(i>=0){arr.splice(i,1);data.xp=Math.max(0,data.xp-habit.xp);toast("Habit unchecked")}
  else{arr.push(id);data.xp+=habit.xp;toast(`+${habit.xp} XP — nice work!`)}
  save();render();
}
function level(){return Math.floor(data.xp/100)+1}
function levelProgress(){return data.xp%100}
function currentStreak(){
  let s=0,d=new Date();
  while(true){let k=dateKey(d), count=(data.completions[k]||[]).length;
    if(count===data.habits.length && data.habits.length){s++;d.setDate(d.getDate()-1)}
    else break;
  }
  return s;
}
function completionFor(k){return data.habits.length?Math.round(((data.completions[k]||[]).length/data.habits.length)*100):0}
function bestCompletion(){
  const vals=Object.keys(data.completions).map(completionFor);
  return vals.length?Math.max(...vals):0
}
function greeting(){const h=new Date().getHours();return h<12?"Good morning 👋":h<18?"Good afternoon 👋":"Good evening 👋"}
function render(){
  document.getElementById("greeting").textContent=greeting();
  const total=data.habits.length, doneToday=(data.completions[todayKey()]||[]).length, pct=total?Math.round(doneToday/total*100):0;
  document.getElementById("score").textContent=pct+"%";document.querySelector(".score-ring").style.setProperty("--p",pct+"%");
  document.getElementById("scoreText").textContent=`${doneToday} of ${total} habits done`;
  document.getElementById("streak").textContent=currentStreak()+" days";
  document.getElementById("xp").textContent=data.xp+" XP";document.getElementById("level").textContent=level();
  document.getElementById("best").textContent=bestCompletion()+"%";
  document.getElementById("sideLevel").textContent=level();document.getElementById("sideXP").textContent=data.xp+" XP";
  document.getElementById("sideProgress").style.width=levelProgress()+"%";
  document.getElementById("habitList").innerHTML=data.habits.length?data.habits.map(h=>`
    <div class="card habit-item">
      <button class="check ${done(h.id)?"done":""}" onclick="toggle(${h.id})">${done(h.id)?"✓":""}</button>
      <div class="habit-main"><div class="habit-name">${esc(h.name)}</div><div class="habit-meta">${esc(h.category)}</div></div>
      <span class="xp-badge">+${h.xp} XP</span>
    </div>`).join(""):`<div class="card habit-item"><div class="habit-main"><div class="habit-name">No habits yet</div><div class="habit-meta">Add your first habit to start your quest.</div></div></div>`;
  document.getElementById("manageList").innerHTML=data.habits.map(h=>`
    <div class="card manage-item"><div class="manage-icon">✦</div><div class="habit-main"><div class="habit-name">${esc(h.name)}</div><div class="habit-meta">${esc(h.category)} · +${h.xp} XP</div></div>
    <button class="delete" onclick="removeHabit(${h.id})">Delete</button></div>`).join("");
  renderCalendar();renderStats();
}
function removeHabit(id){if(!confirm("Delete this habit?"))return;data.habits=data.habits.filter(h=>h.id!==id);Object.keys(data.completions).forEach(k=>data.completions[k]=(data.completions[k]||[]).filter(x=>x!==id));save();render();toast("Habit deleted")}
function esc(s){return s.replace(/[&<>"']/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#039;"}[c]))}
function renderCalendar(){
 const el=document.getElementById("calendar"), now=new Date(), y=now.getFullYear(),m=now.getMonth(),first=new Date(y,m,1).getDay(),days=new Date(y,m+1,0).getDate();
 let html=`<div class="calendar-title"><h3>${now.toLocaleString(undefined,{month:"long",year:"numeric"})}</h3></div><div class="calendar-grid">`;
 ["Sun","Mon","Tue","Wed","Thu","Fri","Sat"].forEach(x=>html+=`<div class="dow">${x}</div>`);
 for(let i=0;i<first;i++)html+="<div></div>";
 for(let d=1;d<=days;d++){const dt=new Date(y,m,d),k=dateKey(dt),pct=completionFor(k),isToday=k===todayKey();html+=`<div class="day ${isToday?"today":""}"><div class="num">${d}</div>${pct?`<span class="dot" title="${pct}% complete"></span>`:""}</div>`}
 html+="</div>";el.innerHTML=html;
}
function renderStats(){
 const keys=Object.keys(data.completions), total=keys.reduce((a,k)=>a+(data.completions[k]||[]).length,0);
 document.getElementById("totalCompletions").textContent=total;
 document.getElementById("avgCompletion").textContent=keys.length?Math.round(keys.map(completionFor).reduce((a,b)=>a+b,0)/keys.length)+"%":"0%";
 let best=0,run=0,d=new Date(); for(let i=0;i<365;i++){if(completionFor(dateKey(d))===100){run++;best=Math.max(best,run)}else run=0;d.setDate(d.getDate()-1)}
 document.getElementById("bestStreak").textContent=best;
 const chart=document.getElementById("chart");let arr=[];for(let i=6;i>=0;i--){let d=new Date();d.setDate(d.getDate()-i);arr.push({d,p:completionFor(dateKey(d))})}
 chart.innerHTML=arr.map(x=>`<div class="bar-wrap"><div class="bar" style="height:${Math.max(5,x.p*1.65)}px" title="${x.p}%"></div><div class="bar-label">${x.d.toLocaleDateString(undefined,{weekday:"short"})}</div></div>`).join("");
}
function openModal(){document.getElementById("modal").classList.remove("hidden");setTimeout(()=>document.getElementById("habitName").focus(),50)}
function closeModal(){document.getElementById("modal").classList.add("hidden")}
function toast(msg){const t=document.getElementById("toast");t.textContent=msg;t.classList.remove("hidden");t.classList.add("show");clearTimeout(window.tt);window.tt=setTimeout(()=>t.classList.add("hidden"),1800)}
document.querySelectorAll(".nav-item").forEach(b=>b.onclick=()=>showView(b.dataset.view));
document.querySelectorAll("[data-view-link]").forEach(b=>b.onclick=()=>showView(b.dataset.viewLink));
function showView(v){document.querySelectorAll(".view").forEach(x=>x.classList.add("hidden"));document.getElementById(v+"View").classList.remove("hidden");document.querySelectorAll(".nav-item").forEach(x=>x.classList.toggle("active",x.dataset.view===v))}
document.getElementById("openModal").onclick=openModal;document.getElementById("openModal2").onclick=openModal;
document.getElementById("closeModal").onclick=closeModal;document.getElementById("closeModal2").onclick=closeModal;
document.getElementById("habitForm").onsubmit=e=>{e.preventDefault();const h={id:Date.now(),name:document.getElementById("habitName").value.trim(),category:document.getElementById("habitCategory").value,xp:Math.max(5,Math.min(100,+document.getElementById("habitXP").value||20))};if(!h.name)return;data.habits.push(h);save();e.target.reset();document.getElementById("habitXP").value=20;closeModal();render();toast("Habit created — your quest begins!")}
document.getElementById("resetBtn").onclick=()=>{if(confirm("Reset all demo progress?")){data=defaultData();save();render();toast("Demo reset")}}
render();

