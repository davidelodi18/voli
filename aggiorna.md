# Aggiornamento voli dell'app "Voli" — istruzioni per Claude

Davide ha toccato "🤖 Chiedi a Claude" nell'app Voli (https://davidelodi18.github.io/voli/).
Il messaggio indica quali viaggi aggiornare (id). Scrivi tutto in italiano, con poche parole tra un passo e l'altro.

## Regole
- Usa **Claude in Chrome** (strumenti `mcp__claude-in-chrome__*`). Caricali con **una sola** ToolSearch:
  `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__tabs_close_mcp`.
  Se Chrome non risponde, digli in una riga di accendere/svegliare il Mac con Chrome aperto e fermati.
- Apri una scheda nuova (`tabs_create_mcp`) e chiudila alla fine. Niente login, captcha o banner cookie.
- **Una chiamata `javascript_tool` alla volta**, mai in parallelo: la pagina cambia sito spesso.
- L'output di `javascript_tool` viene troncato oltre ~1000 caratteri e bloccato se contiene URL: gli snippet qui sotto restituiscono solo testo compatto.
- Se una chiamata dice "browser-internal or unparseable URL" (succede se il link di ritorno è troppo lungo) o "another site", aspetta 3 s, chiama `tabs_context_mcp` per vedere dove è la scheda e riprova.

## Passi

**1. Avvio.** Naviga la scheda a `https://davidelodi18.github.io/voli/`, poi esegui nella pagina:
```js
await new Promise(r=>setTimeout(r,3000)); await voliScan.start(['ID1','ID2'])
```
(metti gli id del messaggio; `[]` = tutti i viaggi attivi). Restituisce l'elenco delle tratte **in ordine** (es. `MXP-ARN BGY-ARN ...`) e apre da sola la prima su Google Flights.

**2. Per ogni tratta, in ordine** (la scheda è su Google Flights):
- Chiamata A: incolla lo **snippet A** qui sotto così com'è. Attende i risultati, legge i voli di andata e la prima combinazione. Restituisce `w… n sel … | combo | elenco prezzi`.
  - Se restituisce `NESSUN RISULTATO`, esegui subito `_send('ORIG-DEST')` e passa alla tratta successiva.
- Chiamata B (solo se `sel` ha più di un indice): `await _doCombo(_sel.slice(1,4))`
- Chiamata C: `const c=await _doCombo(_sel.slice(4)); c+' | '+_send('ORIG-DEST')` (con il codice della tratta, es. `'MXP-ARN'`).
- `_send` torna all'app, che salva i dati e **apre da sola la tratta successiva** dopo ~1 s. Aspetta 4 s e ripeti dalla chiamata A. Non eseguire codice nella pagina dell'app tra una tratta e l'altra.
- Se una tratta si blocca due volte (o la pagina dell'app resta con un link lunghissimo e non risponde), torna all'app (`navigate` a `https://davidelodi18.github.io/voli/`), esegui `voliScan.skip('ORIG-DEST'); voliScan.next()` e prosegui.

**3. Salvataggio.** Dopo l'ultima tratta la scheda resta sull'app. Esegui:
```js
await new Promise(r=>setTimeout(r,3000)); const s=await voliScan.save(); voliScan.clear(); s
```
Restituisce per ogni viaggio l'itinerario migliore e, per tratta: prezzo a persona, ▲/▼ rispetto all'ultima rilevazione, numero di combinazioni (`c`) e quante con voli diretti in entrambe le direzioni (`dir`). `!` = nessun itinerario comodo, è il minimo assoluto.

**4. Resoconto** breve a Davide: per ogni viaggio l'itinerario più conveniente (compagnia, orari, prezzo), le tratte senza voli, le variazioni. Evidenzia cali ≥ 10% o sotto il prezzo obiettivo. Chiudi la scheda.

## Snippet A (pagina Google Flights)
```js
window._sleep=ms=>new Promise(r=>setTimeout(r,ms));
window._P=(l)=>{const num=s=>Number(String(s).replace(/\./g,''));const m=l.match(/di ([\d.]+) euro/);const total=m?num(m[1]):null;let stops=/diretto|senza scali/i.test(l)?0:((l.match(/con (\d+) scal/)||[])[1]|0);const am=l.match(/(?:scal[oi] con|diretto con|senza scali con) ([^.]+)\./);const hh=[...l.matchAll(/alle ore (\d{1,2}:\d{2})/g)].map(x=>x[1]);const dm=l.match(/Durata totale ([^.]+)\./);const dur=dm?dm[1].trim():'';const mins=((dur.match(/(\d+) h/)||[])[1]|0)*60+((dur.match(/(\d+) min/)||[])[1]|0);let via=[...l.matchAll(/nella città di ([^,.]+)/g)].map(x=>x[1].trim());if(!via.length)via=[...l.matchAll(/presso Aeroporto di ([^,.]+)/g)].map(x=>x[1].trim());const change=/Effettua qui il cambio/.test(l),sep=/Biglietti separati/.test(l);const dd=[...l.matchAll(/il giorno [^,]+, ([^ ]+ \d+)/g)].map(x=>x[1]);const night=dd.length>=2&&dd[0]!==dd[1];return {total,stops,airline:am?am[1].trim():'',dep:hh[0]||'',arr:hh[1]||'',dur,via:via.join(', '),change,sep,night,ok:stops<=1&&mins<=720&&!change&&!night,mins,label:l};};
window._labs=(excl)=>{const s=new Set();document.querySelectorAll('[aria-label]').forEach(e=>{const a=e.getAttribute('aria-label');if(a.includes('prezzo totale')&&a.includes('Partenza da')&&!(excl&&excl.has(a)))s.add(a)});return [...s];};
window._list=(labs)=>{const seen=new Set();return labs.map(_P).filter(x=>x.total).sort((a,b)=>a.total-b.total).filter(x=>{const k=x.total+x.airline+x.dep+x.arr;if(seen.has(k))return false;seen.add(k);return true;});};
window._tab=async()=>{const tab=[...document.querySelectorAll('[role=tab]')].find(e=>e.textContent.includes('Più economico'));if(tab){tab.click();await _sleep(4000);}return !!tab;};
window._doCombo=async(idxs)=>{const out=[];for(const i of idxs){const a=_and[i];const find=()=>[...document.querySelectorAll('[aria-label]')].find(e=>e.getAttribute('aria-label')===a.label);let el=find();if(!el){await _tab();el=find();}if(!el){out.push(i+':nf');continue;}el.click();await _sleep(6000);const r=_list(_labs(_andLabels)).slice(0,6);_combo[i]=r;const ok=r.find(x=>x.ok);out.push(i+':'+r.length+(r[0]?'/'+r[0].total+(ok?'/'+ok.total:''):''));history.back();await _sleep(4500);if(!_labs().some(l=>_andLabels.has(l)))await _tab();}return out.join(' ');};
window._send=(k)=>{const c=x=>[x.total,x.stops,x.airline,x.dep,x.arr,x.dur.replace(' h ','h').replace(' min',''),x.via.replace(/Regione di |, Capitale/g,''),(x.change?1:0)|(x.sep?2:0)|(x.night?4:0)|(x.ok?8:0)];const s=JSON.stringify([k,(window._and||[]).map(c),(window._combo||[]).map((r,i)=>r?[i,r.map(c)]:null).filter(Boolean)]).replace(/["'()!]/g,m=>m==='"'?'!':' ').replace(/\[/g,'(').replace(/\]/g,')').replace(/_/g,' ').replace(/ /g,'_');const u='https://davidelodi18.github.io/voli/#q='+encodeURI(s);setTimeout(()=>{location.href=u},100);return 'inviato '+k+' '+u.length;};
let w=0; await _sleep(2000); while(w<20000 && !_labs().length && !/Nessun risultato/.test(document.body.innerText)){await _sleep(1000);w+=1000;}
window._ar=/Andata e ritorno/.test(document.body.innerText.slice(0,800));
const t=await _tab(); const labs=_labs(); window._andLabels=new Set(labs); const all=_list(labs); let and=all.slice(0,10); all.slice(10).filter(x=>x.stops===0).forEach(x=>{if(and.length<12)and.push(x)}); window._and=and; window._combo=[];
const okI=and.map((x,i)=>x.ok?i:-1).filter(i=>i>=0); window._sel=_ar?okI.slice(0,4):[]; if(_ar) and.forEach((x,i)=>{if(x.stops===0&&!_sel.includes(i)&&_sel.length<6)_sel.push(i)});
const cc=_sel.length?await _doCombo(_sel.slice(0,1)):'';
labs.length?('w'+w+' '+labs.length+(_ar?' AR':' SOLO ANDATA')+' sel '+_sel.join(',')+' | '+cc+' | '+and.map(x=>x.total+(x.stops?'':'D')+(x.ok?'':'x')).join(',')):'NESSUN RISULTATO'
```
