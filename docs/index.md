<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Lakota Sky — OLC • Kyle, SD</title>
<style>
  :root{
    --bg:#05070c; --panel:#0b1422; --panel2:#0e1a2a; --ink:#eaf2ff; --muted:#9fb3ca; --edge:#24374f;
    --lakota:#f0abfc; --west:#7dd3fc; --accent:#22c55e; --warn:#f59e0b; --bad:#ef4444;
  }
  *{box-sizing:border-box}
  html,body{height:100%}
  body{margin:0;background:radial-gradient(1400px 900px at 72% 24%, #0c1830 0%, var(--bg) 60%);color:var(--ink);font:14px/1.5 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,Helvetica,Arial}
  .layout{display:grid;grid-template-columns:380px 1fr;grid-template-rows:auto 1fr auto;gap:14px;padding:14px}
  header{grid-column:1/-1;display:flex;align-items:center;justify-content:space-between;padding:10px 14px;background:linear-gradient(180deg,#122139,var(--panel));border:1px solid var(--edge);border-radius:14px;box-shadow:0 10px 40px rgba(0,0,0,.55)}
  h1{margin:0;font-size:18px}
  .panel{background:linear-gradient(180deg,var(--panel),var(--panel2));border:1px solid var(--edge);border-radius:14px;padding:14px;overflow:auto;height:calc(100vh - 160px)}
  .small{font-size:12px;color:var(--muted)}
  .wrap{position:relative;border:1px solid var(--edge);border-radius:14px;overflow:hidden;box-shadow:0 20px 60px rgba(0,0,0,.6)}
  canvas{display:block;width:100%;height:100%;background:radial-gradient(1000px 700px at 60% 35%, #081327, #060b14 60%, #04070e 100%)}
  .hud{position:absolute;top:10px;left:10px;display:flex;gap:8px;flex-wrap:wrap}
  .pill{font-size:12px;padding:4px 8px;border:1px solid #27405c;border-radius:999px;background:#0f1a2a;color:#cde6ff}
  .watermark{position:absolute;bottom:10px;left:12px;font-size:11px;color:#8aa4c4;opacity:.75}
  .footer{grid-column:1/-1;text-align:center;color:#7d93ad;font-size:12px}
  .sliderbar{grid-column:1/-1;display:grid;grid-template-columns:1fr auto;gap:12px;align-items:center;background:linear-gradient(180deg,#0f1b2c,#0d1626);border:1px solid var(--edge);border-radius:14px;padding:10px 12px}
  .sliderbar input[type=range]{width:100%;-webkit-appearance:none;height:10px;border-radius:999px;background:#21324a}
  .sliderbar input[type=range]::-webkit-slider-thumb{-webkit-appearance:none;width:18px;height:18px;border-radius:50%;background:var(--lakota);border:2px solid #0a0f16;box-shadow:0 0 0 6px rgba(240,171,252,.2)}
  .story h3{margin:0 0 8px 0}
  .story p{margin:0 0 10px 0}
  .log{white-space:pre-wrap;font-family:ui-monospace,Menlo,Consolas,monospace;background:#0a1220;border:1px solid #24354f;padding:8px;border-radius:10px}
  .pass{color:#22c55e}.fail{color:#ef4444}.warn{color:#f59e0b}
</style>
</head>
<body>
  <div class="layout">
    <header>
      <div style="display:flex;gap:10px;align-items:center"><div style="width:14px;height:14px;border-radius:50%;background:conic-gradient(from 0deg,#7dd3fc,#f0abfc,#fff,#7dd3fc);"></div><h1>Lakota Sky — OLC • Kyle, SD</h1></div>
      <div class="small">Hóka hé — educational demo; consult local knowledge keepers for authoritative overlays.</div>
    </header>

    <aside class="panel story" id="story">
      <h3>Today’s Sky Story</h3>
      <p id="storyText">Slide the day control to explore the year from OLC’s horizon. A featured constellation is emphasized and a place-based seasonal story appears here (illustrative; please replace with community‑approved text).</p>
      <div class="section">
        <div class="small">Diagnostics</div>
        <div class="inline" style="display:flex;gap:8px;align-items:center;margin:8px 0">
          <button id="runTestsBtn" style="appearance:none;border:1px solid #2a3a52;background:linear-gradient(180deg,#17253b,#101a2a);color:var(--ink);padding:6px 10px;border-radius:10px;cursor:pointer">Run Tests</button>
          <span id="testSummary" class="small">—</span>
        </div>
        <div id="testLog" class="log" style="max-height:160px;overflow:auto"></div>
      </div>
    </aside>

    <main class="wrap">
      <canvas id="sky" width="1200" height="800" aria-label="Sky map"></canvas>
      <div class="hud">
        <div id="info" class="pill">—</div>
      </div>
      <div class="watermark">© Educational demo — shapes & names vary by community</div>
    </main>

    <div class="sliderbar">
      <input id="day" type="range" min="1" max="365" step="1" value="200"/>
      <div class="pill" id="dayLabel">Aug 18</div>
    </div>

    <div class="footer">Vanilla HTML/CSS/JS • No dependencies • MIT License</div>
  </div>

<script>
(()=>{
  const $ = (id)=>document.getElementById(id);
  const sky = $('sky'); const ctx = sky.getContext('2d');
  const daySlider = $('day'); const dayLabel = $('dayLabel');
  const infoEl=$('info'); const storyEl=$('storyText');

  // Kyle, SD (OLC)
  const LAT=43.20, LON=-102.62, UTC_OFFSET=-6; // hours

  // VISUAL STYLE CONSTANTS (brighter & stylized)
  const STYLE={
    starBase: 2.0,          // base radius
    starMax: 4.2,           // cap radius for very bright stars
    starGlow: 0.85,         // glow opacity
    lakotaLine: 2.6,        // focus line width
    lakotaGlow: 12,         // focus glow
    westLine: 1.8,          // western line width
    westAlpha: 0.7,         // western visibility
    labelSize: 13,          // px
  };

  // Animation clock for dynamic horizon/sky accents
  let t=0; let last=performance.now();
  function tick(now){ const dt=Math.min(0.05,(now-last)/1000); last=now; t+=dt; draw(); requestAnimationFrame(tick); }
  requestAnimationFrame(tick);

  // Astronomy helpers (approximate but consistent)
  const DEG=Math.PI/180, HOUR=15*DEG;
  function dayLabelFmt(d){ const date = new Date(new Date().getFullYear(),0,1); date.setDate(d); return date.toLocaleDateString(undefined,{month:'short', day:'numeric'}); }
  function julianDay(y,m,d){ if(m<=2){y-=1; m+=12;} const A=Math.floor(y/100); const B=2-A+Math.floor(A/4); return Math.floor(365.25*(y+4716))+Math.floor(30.6001*(m+1))+d+B-1524.5; }
  function localSiderealTime(dateUTC, lon){ const Y=dateUTC.getUTCFullYear(), M=dateUTC.getUTCMonth()+1, D=dateUTC.getUTCDate() + (dateUTC.getUTCHours()+dateUTC.getUTCMinutes()/60)/24; const JD=julianDay(Y,M,D); const T=(JD-2451545)/36525; let GMST=280.46061837+360.98564736629*(JD-2451545)+0.000387933*T*T - T*T*T/38710000; GMST=((GMST%360)+360)%360; return (GMST+lon)*DEG; }
  function radecToAltAz(raHours, decDeg, latDeg, lstRad){ const ra=raHours*HOUR; const dec=decDeg*DEG; const lat=latDeg*DEG; const H=lstRad-ra; const sinAlt=Math.sin(lat)*Math.sin(dec)+Math.cos(lat)*Math.cos(dec)*Math.cos(H); const alt=Math.asin(sinAlt); const cosAz=(Math.sin(dec)-Math.sin(alt)*Math.sin(lat))/(Math.cos(alt)*Math.cos(lat)); let az=Math.acos(Math.min(1,Math.max(-1,cosAz))); if(Math.sin(H)>0) az=2*Math.PI-az; return {alt,az}; }

  // Dome projection (azimuthal equidistant)
  function project(alt, az){ const fov=160*DEG; const rMax=Math.min(sky.width, sky.height)*0.46; const k=rMax/(fov/2); const r=(Math.PI/2-alt)*k; const cx=sky.width/2, cy=sky.height/2; return {x:cx+r*Math.sin(az), y:cy-r*Math.cos(az), rMax}; }

  // Minimal star catalog
  const Stars=[
    ['Polaris',2.5303, 89.2641, 1.98], ['Vega',18.6156,38.7837,0.03], ['Deneb',20.6905,45.2803,1.25], ['Altair',19.8464,8.8683,0.77],
    ['Sirius',6.7525,-16.7161,-1.46], ['Betelgeuse',5.9195,7.4071,0.58], ['Rigel',5.2423,-8.2016,0.12], ['Capella',5.2782,46,0.08],
    ['Dubhe',11.0621,61.7508,1.79], ['Merak',11.0307,56.3824,2.37], ['Phecda',11.8972,53.6948,2.43], ['Megrez',12.2571,57.0326,3.31], ['Alioth',12.9004,55.9598,1.76], ['Mizar',13.3987,54.9254,2.23], ['Alkaid',13.7923,49.3133,1.85],
    ['Castor',7.5767,31.8883,1.58], ['Pollux',7.7553,28.0262,1.14], ['Aldebaran',4.5987,16.5093,0.85],
  ];

  // Western subset for context (faint but brighter than before)
  const Western={'Big Dipper':{lines:[['Dubhe','Merak'],['Merak','Phecda'],['Phecda','Megrez'],['Megrez','Alioth'],['Alioth','Mizar'],['Mizar','Alkaid']]} };

  // Lakota demo overlay — simplified (replace with community‑approved)
  const Lakota={
    'Wičakíyuhapi (Big Dipper)':{ lines: Western['Big Dipper'].lines },
    'Tayamni (Buffalo — demo)':{ lines:[['Pleiades','Mintaka'],['Mintaka','Alnilam'],['Alnilam','Alnitak'],['Alnitak','Sirius']], virtual:{ 'Pleiades':[3.79,24.1], 'Alnitak':[5.6793,-1.9426], 'Alnilam':[5.6036,-1.2019], 'Mintaka':[5.5334,0.2991] } },
    'Mato Tipila (Bear Lodge — demo)':{ lines:[['Castor','Pollux'],['Pollux','Castor']] }
  };

  function resolveStar(name, virtual){ const s=Stars.find(d=>d[0]===name); if(s) return {ra:s[1], dec:s[2]}; if(virtual && virtual[name]){ const [rah,dd]=virtual[name]; return {ra:rah, dec:dd}; } return null; }

  // Seasonal stories — place-based, culturally mindful (illustrative; replace with community-approved text)
  const Stories=[
    {start:1,end:80,key:'Wičakíyuhapi (Big Dipper)',text:"In deep winter above the snowy grasslands around Kyle, Wičakíyuhapi stands high in the north. Elders say these stars help guide spirits along the Milky Way — the Spirit Road — back to their home. The Dipper’s bright bowl recalls the curve of a buffalo horn used in ceremony, reminding us that winter is a time for stories and for honoring the path between this world and the next."},
    {start:81,end:160,key:'Mato Tipila (Bear Lodge — demo)',text:"As spring returns and buffalo calves find their footing in the breaks and draws, the stars near Gemini recall Mato Tipila, Bear Lodge. Rising beyond the Black Hills, the old story tells of people saved from a mighty bear; the rock bears the marks. This season calls us to protect sacred places and notice how life is waking around them."},
    {start:161,end:250,key:'Tayamni (Buffalo — demo)',text:"Warm summer nights bring the great sky-buffalo, Tayamni, over the southern horizon. Its head rests near Pleiades, the spine follows the Belt of Orion, and the tail reaches toward Sirius. On the open prairie, buffalo gave everything for the people’s survival; in the sky, Tayamni reminds us of reciprocity between land, animals, and people."},
    {start:251,end:365,key:'Wičakíyuhapi (Big Dipper)',text:"In autumn, as prairie grasses turn gold and bison grow thick coats, Wičakíyuhapi tips toward the northwest horizon. Its tilt marks the shortening days and the turning of the year. The stars once again guide travelers home and signal the time to prepare for winter."}
  ];

  function storyForDay(d){ return Stories.find(s=>d>=s.start && d<=s.end) || Stories[0]; }

  // Fix local clock near 21:30 local for sky viewing; slider advances day-of-year
  function dateFromDay(d){ const y=new Date().getFullYear(); const base=new Date(Date.UTC(y,0,1,0,0,0)); const date=new Date(base.getTime()+(d-1)*86400000); const localHour=21.5; const utcHour=localHour-UTC_OFFSET; date.setUTCHours(Math.floor(utcHour), Math.round((utcHour%1)*60),0,0); return date; }

  daySlider.addEventListener('input', ()=>{ dayLabel.textContent=dayLabelFmt(Number(daySlider.value)); updateStory(); });
  dayLabel.textContent=dayLabelFmt(Number(daySlider.value));

  function updateStory(){ const d=Number(daySlider.value); const s=storyForDay(d); storyEl.textContent=s.text + " (Demo text — please replace with community‑approved narrative.)"; }

  function draw(){ const w=sky.width,h=sky.height; ctx.clearRect(0,0,w,h);
    // background dome & stylized Milky Way
    const g=ctx.createRadialGradient(w*0.6,h*0.35,20,w*0.6,h*0.35,Math.max(w,h)); g.addColorStop(0,'rgba(30,58,138,0.35)'); g.addColorStop(1,'rgba(4,7,14,1)'); ctx.fillStyle=g; ctx.fillRect(0,0,w,h);
    drawMilkyWay();

    // horizon edge & prairie
    const cx=w/2, cy=h/2, rMax=Math.min(w,h)*0.46; ctx.save();
    ctx.strokeStyle='rgba(150,200,255,0.25)'; ctx.lineWidth=1.4; ctx.beginPath(); ctx.arc(cx,cy,rMax,0,Math.PI*2); ctx.stroke();
    drawDynamicPrairie(cx,cy,rMax);
    ctx.restore();

    const d=Number(daySlider.value); const dateUTC=dateFromDay(d); const lst=localSiderealTime(dateUTC, LON);

    // stars — brighter, bigger, gentle twinkle that subtly animates with t
    const tw = (d%7)/7 + t*0.6; // pseudo twinkle
    for(const s of Stars){ const {alt,az}=radecToAltAz(s[1],s[2],LAT,lst); if(alt>0){ const {x,y}=project(alt,az);
        const mag = s[3];
        let size = STYLE.starBase + ( -mag + 2 ) * 0.7;
        size = Math.min(STYLE.starMax, Math.max(1.4, size + (Math.sin((x+y)*0.01+tw*3)-0.5)*0.35));
        ctx.save();
        ctx.shadowColor='rgba(255,255,255,'+0.9+')';
        ctx.shadowBlur=size*5.2;
        ctx.fillStyle='white';
        ctx.beginPath(); ctx.arc(x,y,size,0,Math.PI*2); ctx.fill();
        ctx.restore();
      } }

    // context & focus
    drawConstellations(Western,lst,LAT,'rgba(125,211,252,'+STYLE.westAlpha+')', false);
    const focus=storyForDay(d).key; drawConstellations(Lakota,lst,LAT,'rgba(240,171,252,1)', focus);

    infoEl.textContent = `${dayLabelFmt(d)} • 21:30 local`;
  }

  function drawConstellations(set,lst,lat,color,focusName){
    for(const name of Object.keys(set)){
      const focus = (name===focusName);
      const entry=set[name]; const {lines, virtual}=entry;
      ctx.save();
      ctx.globalAlpha = focus ? 1 : 0.75;
      ctx.strokeStyle = color;
      ctx.lineWidth = focus ? STYLE.lakotaLine : STYLE.westLine;
      ctx.shadowColor = color; ctx.shadowBlur = focus ? STYLE.lakotaGlow : 10;
      for(const seg of lines){
        const a=resolveStar(seg[0], virtual); const b=resolveStar(seg[1], virtual); if(!a||!b) continue;
        const A=radecToAltAz(a.ra,a.dec,lat,lst), B=radecToAltAz(b.ra,b.dec,lat,lst);
        if(A.alt>0 && B.alt>0){ const pA=project(A.alt,A.az), pB=project(B.alt,B.az);
          // slight pulsing width on focus lines
          if(focus){ ctx.lineWidth = STYLE.lakotaLine + Math.sin(t*2+ (pA.x+pB.x)*0.001)*0.5; }
          ctx.beginPath(); ctx.moveTo(pA.x,pA.y); ctx.lineTo(pB.x,pB.y); ctx.stroke();
          // nodes
          ctx.beginPath(); ctx.arc(pA.x,pA.y, focus? 3.2:2.2, 0, Math.PI*2); ctx.fillStyle=color; ctx.fill();
          ctx.beginPath(); ctx.arc(pB.x,pB.y, focus? 3.2:2.2, 0, Math.PI*2); ctx.fill();
        }
      }
      if(focus){
        let sx=0,sy=0,n=0; for(const seg of lines){ const a=resolveStar(seg[0], virtual); const b=resolveStar(seg[1], virtual); if(!a||!b) continue; const A=radecToAltAz(a.ra,a.dec,lat,lst), B=radecToAltAz(b.ra,b.dec,lat,lst); if(A.alt>0 && B.alt>0){ const pA=project(A.alt,A.az), pB=project(B.alt,B.az); sx+=(pA.x+pB.x)/2; sy+=(pA.y+pB.y)/2; n++; } }
        if(n>0){
          ctx.shadowBlur=0;
          ctx.font = `800 14px system-ui, sans-serif`;
          ctx.textAlign='center';
          ctx.lineWidth=4; ctx.strokeStyle='rgba(0,0,0,0.85)'; ctx.strokeText(name, sx/n, sy/n-6);
          ctx.fillStyle=color; ctx.fillText(name, sx/n, sy/n-6);
        }
      }
      ctx.restore();
    }
  }

  function drawMilkyWay(){
    const w=sky.width, h=sky.height; const cx=w/2, cy=h/2; const r=Math.min(w,h)*0.42;
    const grad=ctx.createRadialGradient(cx+r*0.2, cy-r*0.1, r*0.1, cx, cy, r*1.2);
    grad.addColorStop(0, 'rgba(180,200,255,0.10)');
    grad.addColorStop(0.5,'rgba(160,190,255,0.07)');
    grad.addColorStop(1, 'rgba(120,150,220,0.00)');
    ctx.save(); ctx.globalCompositeOperation='screen';
    ctx.fillStyle=grad; ctx.beginPath(); ctx.ellipse(cx,cy,r*1.2,r*0.5, -0.6, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  }

  // Dynamic prairie: parallax hills, grass sway, seasonal accents (aurora, birds, fireflies, leaves)
  function drawDynamicPrairie(cx,cy,r){
    const w=sky.width, h=sky.height; const horizonY = cy + r*0.78;
    const season = getSeason(Number(daySlider.value));

    // Night haze gradient (changes with season)
    const gradTop = season==='winter' ? 'rgba(80,140,220,0.28)' : season==='summer' ? 'rgba(80,160,110,0.28)' : 'rgba(100,120,160,0.22)';
    const gradBot = 'rgba(18,30,20,0.94)';
    const gg = ctx.createLinearGradient(0,horizonY-50,0,h);
    gg.addColorStop(0, gradTop); gg.addColorStop(1, gradBot);
    ctx.fillStyle=gg; ctx.fillRect(0,horizonY,w,h-horizonY);

    // Parallax hill layers
    drawHills(horizonY, 0.012, 18, 'rgba(35,58,40,0.9)', 0.4);  // nearest
    drawHills(horizonY-12,0.008, 28, 'rgba(26,44,34,0.85)', 0.25); // mid
    drawHills(horizonY-24,0.006, 38, 'rgba(18,34,28,0.80)', 0.15); // far

    // Seasonal accents
    if(season==='winter') drawAurora(horizonY);
    if(season==='spring') drawBirds(horizonY);
    if(season==='summer') drawFireflies(horizonY);
    if(season==='autumn') drawLeaves(horizonY);

    // Bison herd walking slowly (parallax)
    drawBison(w*0.18 + Math.sin(t*0.25)*8, horizonY-8, 28);
    drawBison(w*0.28 + Math.sin(t*0.3+1.2)*6, horizonY-6, 22);
    drawBison(w*0.78 + Math.sin(t*0.22+2.4)*7, horizonY-7, 26);
  }

  function drawHills(baseY, freq, amp, color, parallax){
    ctx.fillStyle=color; ctx.beginPath(); ctx.moveTo(0,baseY);
    const w=sky.width; for(let x=0;x<=w;x+=32){ const y=baseY-amp*Math.sin(x*freq + t*parallax) - amp*0.6*Math.cos(x*freq*0.6 + t*parallax*0.7); ctx.lineTo(x,y); }
    ctx.lineTo(w,sky.height); ctx.lineTo(0,sky.height); ctx.closePath(); ctx.fill();
  }

  function drawAurora(horizonY){
    const w=sky.width; const base=horizonY-140;
    ctx.save(); ctx.globalCompositeOperation='lighter';
    for(let i=0;i<5;i++){
      const x = w*(0.1 + i*0.18) + Math.sin(t*0.6+i)*20;
      const grad = ctx.createLinearGradient(x,base-80,x,base+120);
      grad.addColorStop(0,'rgba(120,200,255,0.0)');
      grad.addColorStop(0.3,'rgba(120,220,255,0.08)');
      grad.addColorStop(0.7,'rgba(120,255,200,0.10)');
      grad.addColorStop(1,'rgba(0,0,0,0)');
      ctx.fillStyle=grad;
      ctx.beginPath(); ctx.moveTo(x-40,base+120);
      for(let y=0;y<=200;y+=20){ const yy=base+y; const xx=x + Math.sin((y*0.05)+t*1.2+i)*18; ctx.lineTo(xx,yy); }
      ctx.lineTo(x+40,base+120); ctx.closePath(); ctx.fill();
    }
    ctx.restore();
  }

  function drawBirds(horizonY){
    const w=sky.width; const y = horizonY-40;
    ctx.save(); ctx.strokeStyle='rgba(220,230,240,0.7)'; ctx.lineWidth=1.2;
    for(let i=0;i<7;i++){
      const x = ((w + (i*120 + t*60)) % (w+240)) - 120; // drift left→right
      drawBirdGlyph(x,y+Math.sin(i*0.7+t)*6, 10 + (i%3)*2);
    }
    ctx.restore();
  }
  function drawBirdGlyph(x,y,size){ ctx.beginPath(); ctx.moveTo(x-size,y); ctx.quadraticCurveTo(x,y-size*0.8,x+size,y); ctx.quadraticCurveTo(x,y-size*0.8,x-size,y); ctx.stroke(); }

  function drawFireflies(horizonY){
    const w=sky.width; const h=sky.height; ctx.save();
    for(let i=0;i<40;i++){
      const x = (i*37 % w) + (Math.sin(t*1.5 + i)*8);
      const y = horizonY + 6 + (i*13 % (h-horizonY-12)) + Math.sin(t*2 + i)*2;
      const a = 0.2 + 0.8*(0.5+0.5*Math.sin(t*3+i));
      ctx.fillStyle = `rgba(255,255,180,${a})`;
      ctx.shadowColor = 'rgba(255,255,180,0.8)'; ctx.shadowBlur=8;
      ctx.beginPath(); ctx.arc(x,y,1.6,0,Math.PI*2); ctx.fill();
    }
    ctx.restore();
  }

  function drawLeaves(horizonY){
    const w=sky.width; ctx.save();
    for(let i=0;i<20;i++){
      const x = (i*89 % (w+120)) - 60 + Math.sin(t*0.6 + i)*10;
      const y = horizonY - 10 - (i*23 % 80) + (t*20 % 80);
      const r = 4 + (i%3);
      ctx.fillStyle='rgba(200,140,60,0.8)';
      ctx.beginPath(); ctx.ellipse(x,y,r, r*0.6, i, 0, Math.PI*2); ctx.fill();
    }
    ctx.restore();
  }

  function getSeason(d){
    if(d<=80) return 'winter';
    if(d<=160) return 'spring';
    if(d<=250) return 'summer';
    return 'autumn';
  }

  function drawBison(x,y,s){ ctx.save();
    ctx.shadowColor='rgba(255,255,255,0.15)'; ctx.shadowBlur=8;
    ctx.fillStyle='rgba(15,15,15,0.95)';
    ctx.beginPath(); ctx.ellipse(x,y,s*1.2,s*0.6,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x-s*0.9,y-s*0.05,s*0.35,0,Math.PI*2); ctx.fill();
    for(let i=-1;i<=2;i++) { ctx.fillRect(x-6+i*6,y+s*0.4,3,s*0.5); }
    ctx.beginPath(); ctx.moveTo(x-s*1.25,y-s*0.25); ctx.lineTo(x-s*1.0,y-s*0.4); ctx.lineWidth=2; ctx.strokeStyle='rgba(230,230,230,0.55)'; ctx.stroke();
    ctx.restore(); }

  // Tests
  $('runTestsBtn').addEventListener('click', runTests);
  function runTests(){ const logs=[]; const pass=(m)=>logs.push('✅ '+m); const fail=(m)=>logs.push('❌ '+m);
    try{
      const d=Number(daySlider.value); const dateUTC=dateFromDay(d); const lst=localSiderealTime(dateUTC,LON); const pol=Stars.find(s=>s[0]==='Polaris'); const P=radecToAltAz(pol[1],pol[2],LAT,lst); const altDeg=P.alt/DEG; (Math.abs(altDeg-LAT)<2? pass:fail)(`Polaris altitude ≈ latitude (${altDeg.toFixed(1)} vs ${LAT})`);
      const s1=storyForDay(50).key, s2=storyForDay(180).key; (s1!==s2? pass:fail)('Seasonal focus changes with day');
      const pr=project(1,1); (Number.isFinite(pr.x)&&Number.isFinite(pr.y)? pass:fail)('Projection finite');
      try{ draw(); pass('Draw executes'); } catch(e){ fail('Draw threw: '+e.message); }
    }catch(e){ fail('Exception in tests: '+e.message); }
    $('testLog').textContent = logs.join('\n'); $('testSummary').textContent = logs.every(l=>l.startsWith('✅'))? 'All tests passed' : 'Some tests failed'; }

  // First render
  (function init(){ dayLabel.textContent=dayLabelFmt(Number(daySlider.value)); updateStory(); })();
})();
</script>
</body>
</html>
