# Tested JavaScript snippets (paste into `javascript_tool`)

All snippets assume the page is already loaded.

## 1. Teacher ID map (assignments list, `?per_page=100`)
```js
const m={}; document.querySelectorAll('[data-teacher-id][data-teacher-name]')
  .forEach(b=>m[b.dataset.teacherId]=b.dataset.teacherName); m
```

## 2. Select chips and assign (teacher-assignments/<id>/)
`spec` = list of `[gradeWord, [section numbers], siteSubject]`. Submits only if every chip was found and free.
```js
const spec=[["رابع",[1,2,3,4],"الرياضيات"],["خامس",[1],"نشاط"]];
const norm=s=>s.replace(/\s+/g,'').replace('حامس','خامس').replace('أول','اول');
const want=new Set(); spec.forEach(([g,n,s])=>n.forEach(x=>want.add(norm(g+x)+'|'+s)));
const total=want.size, bad=[]; let sel=0;
document.querySelectorAll('td.ta-classroom-col').forEach(td=>{const c=norm(td.textContent);
  td.parentElement.querySelectorAll('button.ta-course-btn').forEach(b=>{
    const k=c+'|'+b.childNodes[0].textContent.trim();
    if(want.has(k)){want.delete(k); if(b.classList.contains('is-free')){b.click();sel++} else bad.push(k+' '+b.className)}})});
const ok=bad.length===0&&want.size===0;
if(ok) setTimeout(()=>document.getElementById('btn-assign').click(),100);
({teacher:(document.body.innerText.match(/إسنادات المعلم — [^\n]*/)||[''])[0], total, sel, ok, bad, missing:[...want]})
```
After ~3 s check: `document.body.innerText.match(/تم الإسناد[^\n]*|الحصص المسندة:\s*\d+/g)`.

Batch pattern per teacher in `browser_batch`: `navigate` → `javascript_tool` (above) → `computer wait 3`.

## 2b. Diff current assignments vs a PDF spec (print-teacher-assignments page)
`P` maps the site's full teacher name → spec list `[grade, [sections], subject]` (same format as §2).
```js
const norm=s=>s.replace(/\s+/g,'').replace('حامس','خامس').replace('أول','اول');
const cur={}; [...document.querySelectorAll('tr')].forEach(tr=>{const c=tr.cells; if(!c||c.length<5) return;
  const n=c[1].innerText.trim(); if(n==='المعلم') return;
  cur[n]=(c[2].innerText.match(/\[[^\]]+\]/g)||[]).map(x=>{const [s,cl]=x.slice(1,-1).split(' - '); return norm(cl)+'|'+s.trim();}).sort();});
const P={/* 'teacher full name': [["رابع",[1,2],"الرياضيات"], ...] */};
const exp=s=>s.flatMap(([g,n,sub])=>n.map(x=>norm(g+x)+'|'+sub)).sort();
const owner={}; Object.entries(P).forEach(([t,s])=>exp(s).forEach(k=>(owner[k]=owner[k]||[]).push(t)));
const diff={}; for(const t of Object.keys(cur)){ if(!P[t]){ if(cur[t].length) diff[t]='NOT IN PDF'; continue; }
  const w=exp(P[t]), h=cur[t], add=w.filter(x=>!h.includes(x)), rem=h.filter(x=>!w.includes(x)); if(add.length||rem.length) diff[t]={add,rem}; }
({diff, dupInPdf:Object.entries(owner).filter(([k,v])=>v.length>1), notInPdf:[...new Set(Object.values(cur).flat())].filter(k=>!owner[k])})
```

## 2c. Grade rule check (e.g. نشاط teacher must teach another subject in the same grade)
```js
const grade=cl=>cl.replace(/\s+/g,'').replace('حامس','خامس').replace(/\d+$/,'');
const T={}; [...document.querySelectorAll('tr')].forEach(tr=>{const c=tr.cells; if(!c||c.length<5) return; const n=c[1].innerText.trim(); if(n==='المعلم') return;
  T[n]=(c[2].innerText.match(/\[[^\]]+\]/g)||[]).map(x=>{const [s,cl]=x.slice(1,-1).split(' - '); return {s:s.trim(), cl, g:grade(cl)};});});
Object.entries(T).flatMap(([t,v])=>{const g=new Set(v.filter(i=>i.s!=='نشاط').map(i=>i.g)); return v.filter(i=>i.s==='نشاط'&&!g.has(i.g)).map(i=>t+' → نشاط '+i.cl);})
```

## 3. Completeness check (any teacher-assignments page)
```js
const all=[...document.querySelectorAll('button.ta-course-btn')];
({total:all.length, free:all.filter(b=>b.classList.contains('is-free'))
  .map(b=>b.closest('tr').querySelector('td').textContent.trim()+' | '+b.childNodes[0].textContent.trim())})
```

## 4. Timetable audit (table-preview)
Adjust `CLASS_RE` and add the school's closed-period rules.
```js
const CLASS_RE=/^(اول|ثاني|ثالث|رابع|خامس|حامس|سادس)/;
const rows=[...document.querySelectorAll('tr')].filter(tr=>CLASS_RE.test(tr.cells[0]?.innerText.trim()||''));
const days=['الأحد','الاثنين','الثلاثاء','الأربعاء','الخميس']; const NCLS=rows.length/5;
const G={},T={},LD={}; let empty=0;
rows.forEach((tr,ri)=>{const d=days[Math.floor(ri/NCLS)]; const cls=tr.cells[0].innerText.trim().replace(/\s+/g,'');
  [...tr.cells].slice(1).forEach((c,i)=>{const l=c.innerText.split('\n').map(s=>s.trim()).filter(Boolean);
    if(!l.length||l[0]==='—'){empty++;return;}
    G[d+'|'+cls+'|'+(i+1)]={subj:l[0],tch:l[1]}; T[l[1]+'|'+d+'|'+(i+1)]=cls; LD[l[1]+'|'+d]=(LD[l[1]+'|'+d]||0)+1;})});
window.__G=G; window.__T=T; window.__LD=LD;
const load=t=>days.map(d=>LD[t+'|'+d]||0).join('/');
const heavy=[...new Set(Object.keys(LD).map(k=>k.split('|')[0]))].filter(t=>days.some(d=>(LD[t+'|'+d]||0)>=6)).map(t=>t+' '+load(t));
({empty, heavy})
```

## 4b. Full post-generation audit (table-preview; fetches every teacher's assignments)
Finds dropped lessons, same-subject repeats, day loads and closure violations. Fill `ids` from §1, `DOUBLES` from تخصيص المواد (subjects with `even_classes`>0), and `closedRule` from §7b. Expected filled count = Σ class slots (e.g. 10×34 + 5×35 = 515).
```js
const ids=[/* teacher ids from §1 */]; const DOUBLES=['رياضيات','الرياضيات'];
const closedRule=(t,d,p)=>false; // e.g. (['TEACHER A','TEACHER B'].includes(t)&&p==='7')||(t==='TEACHER C'&&['الأربعاء','الخميس'].includes(d))
const P=async u=>new DOMParser().parseFromString(await fetch(u).then(r=>r.text()),'text/html');
const A={}; for(const id of ids){const d=await P('/pro/pro_dashboard/teacher-assignments/'+id+'/'); const name=(d.body.textContent.match(/إسنادات المعلم — ([^\n]*)/)?.[1]||id).trim();
 d.querySelectorAll('td.ta-classroom-col').forEach(td=>{const c=td.textContent.trim().replace(/\s+/g,' ');
  td.parentElement.querySelectorAll('button.ta-course-btn.is-self').forEach(b=>{A[c+'|'+b.childNodes[0].textContent.trim()]={t:name,n:+((b.textContent.match(/\((\d+)\)/)||[])[1]||0)};});});}
const days=['الأحد','الاثنين','الثلاثاء','الأربعاء','الخميس'];
const rows=[...document.querySelectorAll('tr')].filter(tr=>tr.cells.length>3&&/^(اول|ثاني|ثالث|رابع|خامس|سادس)/.test(tr.cells[0].innerText.trim())); const N=rows.length/5;
const G={}, empty=[]; rows.forEach((tr,ri)=>{const d=days[Math.floor(ri/N)]; const cls=tr.cells[0].innerText.trim().replace(/\s+/g,' ');
 [...tr.cells].slice(1).forEach((c,i)=>{ if(c.classList.contains('is-placeholder')) return;
  const l=c.innerText.split('\n').map(s=>s.trim()).filter(s=>s&&!/^\d+:\d+/.test(s));
  if(!l.length||l[0]==='—'){empty.push(d+' ح'+(i+1)+' '+cls);return;} G[d+'|'+cls+'|'+(i+1)]={s:l[0],t:l[1]||''};});});
window.__A=A; window.__G=G;
const cnt={}; Object.entries(G).forEach(([k,v])=>{const kk=k.split('|')[1]+'|'+v.s; cnt[kk]=(cnt[kk]||0)+1;});
const dropped=Object.entries(A).filter(([k,v])=>(cnt[k]||0)!==v.n).map(([k,v])=>k+' '+v.t+' need '+v.n+' got '+(cnt[k]||0));
const C={}; Object.entries(G).forEach(([k,v])=>{const [d,cls,p]=k.split('|'); (C[cls+'|'+d+'|'+v.s]=C[cls+'|'+d+'|'+v.s]||[]).push(+p);});
const repeats=Object.entries(C).filter(([k,ps])=>ps.length>1&&!(DOUBLES.includes(k.split('|')[2])&&ps.length===2&&Math.abs(ps[1]-ps[0])===1)).map(([k,ps])=>k+' ح'+ps.join('+'));
const LD={}; Object.entries(G).forEach(([k,v])=>{const d=k.split('|')[0]; if(!v.t) return; (LD[v.t]=LD[v.t]||{})[d]=(LD[v.t][d]||0)+1;});
const loads=Object.entries(LD).map(([t,o])=>{const a=days.map(d=>o[d]||0); return t+' '+a.join('/')+(Math.max(...a)>=6?' ⚠6':'');});
const closedHits=Object.entries(G).filter(([k,v])=>{const [d,,p]=k.split('|'); return closedRule(v.t,d,p);}).map(([k,v])=>k+' '+v.s+' '+v.t);
({filled:Object.keys(G).length, emptyN:empty.length, dropped, repeatsN:repeats.length, repeats:repeats.slice(0,40), closedHits, loads})
```
Teacher names in `A` are full names; in `G` they are nicknames — compare loads by nickname, dropped lessons by class×subject.

## 4c. Capacity check: assignment periods per class vs slots (read-only, before generating)
Run on any teacher-assignments page (every class/subject chip is listed there, whoever owns it), then compare with each class's slots (workdays settings; or count non-`is-placeholder` cells per class on table-preview).
```js
const tot={}, free=[]; document.querySelectorAll('td.ta-classroom-col').forEach(td=>{const c=td.textContent.trim().replace(/\s+/g,' ');
 td.parentElement.querySelectorAll('button.ta-course-btn').forEach(b=>{tot[c]=(tot[c]||0)+(+((b.textContent.match(/\((\d+)\)/)||[])[1]||0)); if(b.classList.contains('is-free')) free.push(c+'|'+b.childNodes[0].textContent.trim());});});
({tot, free})
```
Slots per class from table-preview: `rows.forEach((tr,ri)=>slots[cls]+=[...tr.cells].slice(1).filter(c=>!c.classList.contains('is-placeholder')).length)`. Any class with `tot > slots` will lose (tot − slots) lessons silently; `tot < slots` leaves free periods (fine if they land at the end of a day).

## 5. Swap search (run after the audit; same class, heavy day → light day)
```js
const G=__G,T=__T,LD=__LD; const days=['الأحد','الاثنين','الثلاثاء','الأربعاء','الخميس'];
const nP=d=>['الأربعاء','الخميس'].includes(d)?6:7;               // periods per day — adjust
const closed=(t,d,p)=>false;                                   // fill from teacher-customize pages
const fixedAct=(cls,s)=>s==='نشاط'&&/^(اول|ثاني|ثالث)/.test(cls); // pinned subjects — adjust
const isDouble=(d,cls,p)=>{const x=G[d+'|'+cls+'|'+p];return [p-1,p+1].some(q=>{const y=G[d+'|'+cls+'|'+q];return y&&y.subj===x.subj&&y.tch===x.tch});};
const load=t=>days.map(d=>LD[t+'|'+d]||0);
const cnt=(d,cls,s,ex)=>Object.keys(G).filter(k=>k.startsWith(d+'|'+cls+'|')&&k!==ex&&G[k].subj===s).length;
const weekly=(cls,s)=>Object.keys(G).filter(k=>k.split('|')[1]===cls&&G[k].subj===s).length;
const dupBad=(d,cls,s,ex)=>cnt(d,cls,s,ex)+1>Math.ceil(weekly(cls,s)/5);
function search(A,src){const out=[],L=load(A),si=days.indexOf(src);
 for(const [k,v] of Object.entries(G)){const [d,cls,p]=k.split('|');
  if(d!==src||v.tch!==A||fixedAct(cls,v.subj)||isDouble(d,cls,+p)) continue;
  days.forEach((td,ti)=>{ if(td===src||L[ti]>=5) return;
   for(let q=1;q<=nP(td);q++){const tk=td+'|'+cls+'|'+q,b=G[tk];
    if(!b||!b.tch||b.tch===A||b.subj===v.subj||fixedAct(cls,b.subj)||isDouble(td,cls,q)) continue;
    if(T[A+'|'+td+'|'+q]||closed(A,td,q)||T[b.tch+'|'+src+'|'+p]||closed(b.tch,src,+p)) continue;
    const issues=[]; if(dupBad(td,cls,v.subj,tk)) issues.push('dup '+v.subj); if(dupBad(src,cls,b.subj,k)) issues.push('dup '+b.subj);
    const bl=load(b.tch), nb=bl.map((x,i)=>i===si?x+1:i===ti?x-1:x);
    if(nb[si]>5) issues.push(b.tch+' '+nb[si]+'@'+src); if(nb[ti]<4&&bl[ti]>=4) issues.push(b.tch+' low@'+td);
    out.push({swap:src+' ح'+p+' '+cls+' '+v.subj+' ⇄ '+td+' ح'+q+' '+b.subj+'/'+b.tch,
      newA:L.map((x,i)=>i===si?x-1:i===ti?x+1:x).join('/'), newB:nb.join('/'), issues:issues.join(', ')});}});}
 return out.sort((a,b)=>a.issues.length-b.issues.length);}
search('TEACHER SHORT NAME','الاثنين').slice(0,8)
```
To consider splitting a double, drop the `isDouble` filter on the source lesson and report the split as a cost.

## 6. Manual-edit cell coordinates (edit page)
```js
const td=[...document.querySelectorAll('td[data-cell-day]')].find(td=>td.dataset.cellDay==='الاثنين'&&td.dataset.cellClass==='1'&&td.dataset.cellClassroom.replace(/\s+/g,'')==='ثالث2');
td.scrollIntoView({block:'center'}); await new Promise(r=>setTimeout(r,500));
const b=td.querySelector('.swap-button').getBoundingClientRect(); const f=1475/innerWidth; // 1475 = screenshot frame width
({visibleDays:[...new Set([...document.querySelectorAll('td[data-cell-day]')].filter(t=>t.offsetParent).map(t=>t.dataset.cellDay))], x:(b.x+b.width/2)*f, y:(b.y+b.height/2)*f})
```
After selecting a source, the target cell shows an orange ⇄ icon in its header — take a screenshot to locate it.

## 7. Teacher availability grid (teacher-customize)
```js
const t=[...document.querySelectorAll('table')].find(t=>/الأحد/.test(t.innerText));
[...t.querySelectorAll('tr')].filter(r=>/الأحد|الاثنين|الثلاثاء|الأربعاء|الخميس/.test(r.innerText))
 .map(r=>r.cells[0].innerText.trim()+': '+[...r.querySelectorAll('input')].map(i=>i.checked?'✓':'✗').join(' '))
```
Toggle with `input.click()`, then real-click `احفظ أوقات العمل المتاحة`, reload, re-run.

## 7b. Read every teacher's closed periods without opening pages (any page)
Uses the `checked` attribute in the server HTML (unchecked = closed). `ids` = teacher id map from §1.
```js
const days=['الأحد','الاثنين','الثلاثاء','الأربعاء','الخميس']; const CL={};
for(const [id,n] of Object.entries(ids)){ const h=await fetch('/pro/pro_dashboard/teacher-customize/'+id+'/').then(r=>r.text());
  const d=new DOMParser().parseFromString(h,'text/html'); const tb=[...d.querySelectorAll('table')].find(x=>/الأحد/.test(x.textContent)); if(!tb) continue;
  [...tb.querySelectorAll('tr')].forEach(r=>{const day=r.cells[0]?.textContent.trim(); if(!days.includes(day)) return;
    [...r.querySelectorAll('input[type=checkbox]')].forEach((cb,i)=>{ if(!cb.hasAttribute('checked')) CL[n+'|'+day+'|'+(i+1)]=1; });}); }
CL
```
The same fetch pattern reads `input[name=nickname]` / `input[name=max_classes]` for all teachers.

## 7c. Edit-page grid keyed by teacher id (handles duplicate nicknames)
```js
const G={}, LOCK={}, OFF=new Set();
document.querySelectorAll('td[data-cell-day]').forEach(td=>{const k=td.dataset.cellDay+'|'+td.dataset.cellClassroom.replace(/\s+/g,'')+'|'+td.dataset.cellClass;
  if(td.dataset.isOffCell==='true') OFF.add(k); const o=td.querySelector('select').selectedOptions[0]; const txt=(o?.text||'').trim();
  if(!txt||/^-+$/.test(txt)){G[k]=null;return;} const [tch,subj]=txt.split(' - ').map(x=>x.trim()); G[k]={t:o.id,tch,subj};
  const l=td.querySelector('.btn-lock-cell'); LOCK[k]=!!(l&&!l.classList.contains('hidden'));});
```

## 7d. Substitution schedule audit (edit-wc-table)
```js
const tb=[...document.querySelectorAll('table')].find(t=>/الأحد/.test(t.innerText)); const days=['الأحد','الاثنين','الثلاثاء','الأربعاء','الخميس']; const grid={}, cnt={};
[...tb.querySelectorAll('tr')].forEach(r=>{const d=r.cells[0]?.innerText.trim(); if(!days.includes(d)) return;
  grid[d]=[...r.cells].slice(1).map((c,i)=>{const n=[...c.querySelectorAll('.teacher-nickname')].map(e=>e.textContent.trim()); n.forEach(x=>cnt[x]=(cnt[x]||0)+1); return 'ح'+(i+1)+': '+(n.join(' + ')||'—');});});
({grid, cnt})
```
To set wizard step-3 quotas: `const set=Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set; set.call(input,'0'); input.dispatchEvent(new Event('input',{bubbles:true})); input.dispatchEvent(new Event('change',{bubbles:true}));`

## 8. Quality summary (table-quality)
```js
await new Promise(r=>setTimeout(r,4000));
[...document.querySelectorAll('*')].filter(e=>/عنصر بحاجة إلى مراجعة/.test(e.textContent)&&e.children.length<=2&&e.textContent.length<60).forEach(e=>e.click());
await new Promise(r=>setTimeout(r,1500));
const t=document.body.innerText.replace(/\n{2,}/g,'\n');
({score:t.slice(t.indexOf('درجة الجودة')-6,t.indexOf('درجة الجودة')+20), weak:t.match(/الأضعف[^\n]*/)?.[0]})
```
Then slice sections like `توزيع متوازن على أيام الأسبوع\nتبقى` for named teachers.

## 9. Solver constraint states (solver-priorities)
```js
const out=[]; document.querySelectorAll('input[type=checkbox]').forEach(cb=>{let p=cb;
 for(let i=0;i<8;i++){p=p.parentElement; if(!p) break; const t=p.innerText||'';
  if(/منخفض/.test(t)){const first=t.split('\n').map(s=>s.trim()).filter(Boolean)[0];
   if(!out.some(o=>o.startsWith(first))) out.push(first+': '+(cb.checked?'ON':'OFF')); break;}}}); out
```
The selected level is the level button with a non-transparent background colour.

## 9b. Set constraint levels (solver-priorities), then real-click `احفظ`
```js
if(!/ACCOUNT NAME/.test(document.body.innerText)) throw 'wrong account';
const want={'المادة مرة واحدة في اليوم':'عالي','توزيع متوازن على أيام الأسبوع':'عالي','عدالة الحصص الأخيرة':'منخفض','توازن نصفَي اليوم':'منخفض'};
const card=name=>{for(const cb of document.querySelectorAll('input[type=checkbox]')){let p=cb; for(let i=0;i<8;i++){p=p.parentElement; if(!p) break;
  if(/منخفض/.test(p.innerText||'')){ if(p.innerText.split('\n').map(s=>s.trim()).filter(Boolean)[0]===name) return p; break; }}} return null;};
const done=[]; for(const [n,l] of Object.entries(want)){const c=card(n); if(!c){done.push(n+' NOT FOUND');continue;}
  [...c.querySelectorAll('button')].find(b=>b.innerText.trim()===l).click(); done.push(n+' → '+l); await new Promise(r=>setTimeout(r,400));} done
```
Cards re-order after each change (raised → top), which is why each card is looked up fresh. Then real-click `احفظ` in the sticky bar (top-left), reload, verify with §9.
