---
layout: default
title: "Skat Scorekeeper"
permalink: /skat-scorekeeper/
---

<style>
body { font-family: monospace; margin:5px; font-size: clamp(12px, 3vw, 16px); }
.container { max-width:760px; margin:auto; overflow-x:auto; }

table { border-collapse:collapse; width:100%; table-layout:fixed; font-size: inherit; }

th, td {
  border:1px solid black;
  text-align:center;
  min-height:22px;
  font-size: clamp(10px, 2.5vw, 12px);
}

/* strong separator */
td:nth-child(12), th:nth-child(12){
  border-right:3px solid black;
}

/* between players */
.sep { border-right:2px solid black; }

/* round blocks */
.block td { border-bottom:3px solid black; }

/* dealer shading */
.dealer { background:#e0e0e0; }

/* highlights */
.current { background:#f0f0f0; font-weight:bold; }
.editing { background:#cfcfcf; }

.panel { border:1px solid black; padding:6px; margin-bottom:6px; }

.row-flex { display:flex; gap:2px; }
.btn {
  flex:1;
  border:1px solid black;
  padding:6px;
  text-align:center;
}
.btn.active { background:black; color:white; }

#names { display:flex; gap:8px; margin-top:8px; }
#names input { flex:1; min-width:0; }

.edit { cursor:pointer; }

button#applyBtn {
  cursor: pointer;
}

button#applyBtn.disabled,
button#applyBtn:disabled {
  background: #e0e0e0;
  color: #666;
  border-color: #999;
  cursor: not-allowed;
}

.spacer { height:6px; }

@media (max-width: 640px) {
  .row-flex { flex-wrap: wrap; }
  .btn { padding: 4px; min-width: 0; }
  #names { flex-direction: column; }
  #names input { width: 100%; }
  input#bid { width: 100%; max-width: 140px; }
}
</style>

<div class="container">

<div class="panel">

Players:
<div class="row-flex" id="playerBtns">
<div class="btn active" onclick="setPlayers(3,this)">3</div>
<div class="btn" onclick="setPlayers(4,this)">4</div>
</div>

<div id="names"></div>

<div class="spacer"></div>

Game:
<div class="row-flex" id="suitBtns"></div>

With / Without:
<div class="row-flex" id="modeBtns"></div>

Jacks:
<div class="row-flex" id="jackBtns"></div>

<div class="spacer"></div>

<div class="row-flex" id="multRow"></div>

<div class="spacer"></div>

Bid:
<input id="bid" style="width:60px">

<div class="spacer"></div>

Declarer:
<div class="row-flex" id="decl"></div>

Result:
<div class="row-flex" id="winloss"></div>

<button id="applyBtn" onclick="applyRow()">APPLY</button>

</div>

<table id="sheet"></table>

</div>

<script>

const baseVals={"♣":12,"♠":11,"♥":10,"♦":9,"Grand":24,"Null":23};

let playersCount=3;
let playerNames=["P1","P2","P3","P4"];
let state=[];
let rowIndex=0;
let editIndex=null;
let control={};

// ---------- PLAYERS ----------
function setPlayers(n,el){

  if(playersCount!==n){
    state=[];
    rowIndex=0;
    editIndex=null;
    control={};
  }

  playersCount=n;

  document.querySelectorAll("#playerBtns .btn").forEach(b=>b.classList.remove("active"));
  el.classList.add("active");

  buildTable();
  drawNames();
  buildDeclarer();
}

// ---------- INIT ----------
function init(){
  drawNames();
  buildControls();
  buildTable();
  highlightCurrent();
}

// ---------- NAMES ----------
function drawNames(){
  names.innerHTML = playerNames.slice(0,playersCount).map((n,i)=>
    `<input value="${n}" oninput="playerNames[${i}]=this.value;updateNames()">`
  ).join("");
}

function updateNames(){
  for(let i=0;i<playersCount;i++){
    document.getElementById("ph"+i).innerText=playerNames[i];
  }
  buildDeclarer();
}

// ---------- BUTTONS ----------
function makeRow(el,list,key){
  el.innerHTML="";
  list.forEach(v=>{
    let b=document.createElement("div");
    b.className="btn";
    b.innerText=v;

    b.onclick=()=>{
      [...el.children].forEach(x=>x.classList.remove("active"));
      b.classList.add("active");
      control[key]=v;
      updateApplyState();
    };

    el.appendChild(b);
  });
}

// ---------- CONTROLS ----------
function buildControls(){

  makeRow(suitBtns,["♣","♠","♥","♦","Grand","Null"],"suit");
  makeRow(modeBtns,["With","Without"],"mode");
  makeRow(jackBtns,["1","2","3","4"],"jacks");
  makeRow(winloss,["Win","Loss"],"win");

  multRow.innerHTML="";
  ["hand","sch","scha","schw","schwa","open"].forEach(k=>{
    let b=document.createElement("div");
    b.className="btn";
    b.innerText=k;

    b.onclick=()=>{
      const isActive = !b.classList.contains("active");
      control[k] = isActive;
      b.classList.toggle("active", isActive);
      syncSpecialFlags();
      refreshMultiRowButtons();
      updateApplyState();
    };

    multRow.appendChild(b);
  });
  updateApplyState();
}

function syncSpecialFlags(){
  if(control.open){
    control.hand = true;
    control.sch = true;
    control.scha = true;
    control.schw = true;
    control.schwa = true;
    return;
  }

  if(control.schwa){
    control.hand = true;
    control.sch = true;
    control.scha = true;
    control.schw = true;
    return;
  }

  if(control.schw){
    control.sch = true;
    return;
  }

  if(control.scha){
    control.hand = true;
    control.sch = true;
    return;
  }

  if(control.sch){
    control.hand = control.hand || false;
  }
}

function refreshMultiRowButtons(){
  Array.from(multRow.children).forEach(b=>{
    const key = b.innerText.trim();
    b.classList.toggle("active", !!control[key]);
  });
}

function updateApplyState(){
  const applyBtn = document.getElementById("applyBtn");
  if(!applyBtn) return;
  const ready = !!control.suit && !!control.jacks && !!control.win && typeof control.decl === 'number' && !isNaN(control.decl);
  applyBtn.disabled = !ready;
  applyBtn.classList.toggle("disabled", !ready);
}

function resetSelection(){
  control = {};
  document.querySelectorAll("#suitBtns .btn, #modeBtns .btn, #jackBtns .btn, #winloss .btn, #multRow .btn").forEach(b=>b.classList.remove("active"));
  const bidInput = document.getElementById("bid");
  if(bidInput) bidInput.value = "";
  buildDeclarer();
}

// ---------- TABLE ----------
function buildTable(){

  let h="<tr>";

  let cols=["No","Val","Mode","Jacks","H","S","SA","Z","ZA","O","+","-"];
  cols.forEach(c=>h+=`<th rowspan=2>${c}</th>`);

  for(let i=0;i<playersCount;i++){
    h+=`<th colspan=3 id="ph${i}">${playerNames[i]}</th>`;
  }

  h+=`<th rowspan=2></th></tr><tr>`;

  for(let i=0;i<playersCount;i++){
    h+=`<th>Score</th><th>Win</th><th class="sep">Loss</th>`;
  }

  h+="</tr>";

  let rounds=playersCount===4?48:36;

  for(let i=0;i<rounds;i++){

    let dealer=i%playersCount;
    let block=((i+1)%playersCount===0)?"block":"";

    h+=`<tr id="row${i}" class="${block}">\n    <td>${i+1}</td>`;

    for(let j=0;j<11;j++) h+="<td></td>";

    for(let j=0;j<playersCount;j++){
      let d=j===dealer?"dealer":"";
      h+=`<td class="${d}"></td><td class="${d}"></td><td class="sep ${d}"></td>`;
    }

    h+=`<td class="edit" onclick="editRow(${i})">✎</td></tr>`;
  }

  // ✅ SEGER TOTALS BACK
  h+=`\n<tr><td colspan=12>Game Points</td>\n${Array(playersCount).fill('<td colspan=3 class="gp"></td>').join("")}</tr>\n\n<tr><td colspan=12>Win / Loss</td>\n${Array(playersCount).fill('<td colspan=3 class="wl"></td>').join("")}</tr>\n\n<tr><td colspan=12>(W-L)*50</td>\n${Array(playersCount).fill('<td colspan=3 class="b1"></td>').join("")}</tr>\n\n<tr><td colspan=12>(Opp Loss)</td>\n${Array(playersCount).fill('<td colspan=3 class="b2"></td>').join("")}</tr>\n\n<tr><td colspan=12>Total</td>\n${Array(playersCount).fill('<td colspan=3 class="tot"></td>').join("")}</tr>\n`;

  sheet.innerHTML=h;
  buildDeclarer();
}

// ---------- DECLARER ----------
function buildDeclarer(rowIdx){

  decl.innerHTML="";
  let dealer=(rowIdx??rowIndex)%playersCount;
  let defaultDecl = Number.isFinite(control.decl) && control.decl!==dealer && control.decl<playersCount
    ? control.decl
    : (dealer===0?1:0);

  control.decl = defaultDecl;

  for(let i=0;i<playersCount;i++){

    let b=document.createElement("div");
    b.className="btn"+(i===dealer?" dealer":"")+(i===control.decl?" active":"");
    b.innerText=playerNames[i];

    if(i!==dealer){
      b.onclick=()=>{
        [...decl.children].forEach(x=>x.classList.remove("active"));
        b.classList.add("active");
        control.decl=i;
        if(editIndex!==null){
          let row=document.getElementById("row"+editIndex).children;
          for(let j=0;j<playersCount;j++){
            let idx=12+j*3;
            if(row[idx]) row[idx].innerText="";
            if(row[idx+1]) row[idx+1].innerText="";
            if(row[idx+2]) row[idx+2].innerText="";
          }
        }
        updateApplyState();
      };
    }

    decl.appendChild(b);
  }
  refreshMultiRowButtons();
  updateApplyState();
}

// ---------- VALUE ----------
function calcValue(){

  let base=baseVals[control.suit]||0;
  let mult=1+(parseInt(control.jacks)||1);

  ["hand","sch","scha","schw","schwa","open"].forEach(k=>{
    if(control[k]) mult++;
  });

  return base*mult;
}

// ---------- APPLY ----------
function applyRow(){

  let r=editIndex??rowIndex;
  let d=parseInt(control.decl);

  if(isNaN(d) || d===r%playersCount) return;

  let row=document.getElementById("row"+r).children;

  let total=calcValue();
  let bidValue=parseInt(document.getElementById("bid").value)||0;

  let win=(control.win==="Win" && total>=bidValue);
  let pts=win?total:-2*total;

  row[1].innerText=baseVals[control.suit]||"";
  row[2].innerText=control.mode==="With"?"W":control.mode==="Without"?"WO":"";
  row[3].innerText=control.jacks||"";

  row[4].innerText=control.hand?"X":"";
  row[5].innerText=control.sch?"X":"";
  row[6].innerText=control.scha?"X":"";
  row[7].innerText=control.schw?"X":"";
  row[8].innerText=control.schwa?"X":"";
  row[9].innerText=control.open?"X":"";

  row[10].innerText=win?total:"";
  row[11].innerText=!win?total*2:"";

  if(editIndex!==null){
    state[editIndex]={d,pts};
  } else {
    state.push({d,pts});
    rowIndex++;
  }

  editIndex=null;

  updateTotals();
  highlightCurrent();
  resetSelection();
}

// ---------- EDIT ----------
function editRow(i){
  if(i>=state.length) return;

  document.querySelectorAll(".editing").forEach(r=>r.classList.remove("editing"));

  editIndex=i;
  document.getElementById("row"+i).classList.add("editing");
  buildDeclarer(i);
}

// ---------- TOTALS ----------
function updateTotals(){

  let p=playersCount;
  let score=Array(p).fill(0);
  let w=Array(p).fill(0);
  let l=Array(p).fill(0);
  let opp=Array(p).fill(0);

  state.forEach((r,i)=>{
    if(typeof r.d !== 'number' || isNaN(r.d) || r.d<0 || r.d>=p) return;

    score[r.d]+=r.pts;

    if(r.pts>0) w[r.d]++;
    else{
      l[r.d]++;
      for(let j=0;j<p;j++) if(j!==r.d) opp[j]++;
    }

    let row=document.getElementById("row"+i).children;
    let idx=12+r.d*3;

    if(row[idx]) row[idx].innerText=score[r.d];
    if(row[idx+1]) row[idx+1].innerText=w[r.d];
    if(row[idx+2]) row[idx+2].innerText=l[r.d];
  });

  document.querySelectorAll(".gp").forEach((el,i)=>el.innerText=score[i]);
  document.querySelectorAll(".wl").forEach((el,i)=>el.innerText=`${w[i]} / ${l[i]}`);
  document.querySelectorAll(".b1").forEach((el,i)=>el.innerText=(w[i]-l[i])*50);
  document.querySelectorAll(".b2").forEach((el,i)=>el.innerText=opp[i]*(p===4?30:40));
  document.querySelectorAll(".tot").forEach((el,i)=>
    el.innerText=score[i]+(w[i]-l[i])*50+opp[i]*(p===4?30:40)
  );
}

// ---------- HIGHLIGHT ----------
function highlightCurrent(){
  document.querySelectorAll("tr").forEach(r=>r.classList.remove("current"));
  let row=document.getElementById("row"+rowIndex);
  if(row) row.classList.add("current");
}

init();

</script>
