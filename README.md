<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no">
<title>Jetpack Jump – Doodle Jump x Jetpack Joyride</title>
<style>
  html,body{margin:0;height:100%;background:#070a16;overflow:hidden;font-family:"Segoe UI",system-ui,Arial,sans-serif}
  #wrap{position:absolute;inset:0;display:flex;align-items:center;justify-content:center}
  canvas{border-radius:14px;box-shadow:0 0 50px rgba(0,170,255,.25);touch-action:none}
</style>
</head>
<body>
<div id="wrap"><canvas id="c" width="480" height="800"></canvas></div>
<script>
// ===================== Setup =====================
const cv = document.getElementById('c'), ctx = cv.getContext('2d');
const W = 480, H = 800;
function resize(){ const s = Math.min(innerWidth / W, innerHeight / H) * 0.98; cv.style.width = W*s + 'px'; cv.style.height = H*s + 'px'; }
addEventListener('resize', resize); resize();

const rand = (a,b) => a + Math.random()*(b-a);
const clamp = (v,a,b) => Math.max(a, Math.min(b, v));
const posmod = (a,n) => ((a % n) + n) % n;
function rr(x,y,w,h,r){ ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath(); }

// ===================== Sound =====================
let actx = null, muted = false;
function beep(f, d=0.1, type='square', vol=0.08, slide=0){
  if (muted) return;
  try{
    if(!actx) actx = new (window.AudioContext||window.webkitAudioContext)();
    const o = actx.createOscillator(), g = actx.createGain();
    o.type = type; o.frequency.value = f;
    if (slide) o.frequency.exponentialRampToValueAtTime(Math.max(30,f+slide), actx.currentTime+d);
    g.gain.value = vol; g.gain.exponentialRampToValueAtTime(0.0001, actx.currentTime+d);
    o.connect(g); g.connect(actx.destination); o.start(); o.stop(actx.currentTime+d);
  }catch(e){}
}

// ===================== Input =====================
const keys = {};
const touch = {l:false,r:false,j:false,any:false}; let isTouch = false, mouseJet = false;
addEventListener('keydown', e => {
  keys[e.code] = true;
  if (['Space','ArrowUp','ArrowLeft','ArrowRight','ArrowDown'].includes(e.code)) e.preventDefault();
  if (e.code === 'KeyP' && state === 'play') paused = !paused;
  if (e.code === 'KeyM') muted = !muted;
  if (e.code === 'KeyR' && state !== 'play' && canStart()) { startReels(); return; }
  if ((e.code === 'Space' || e.code === 'Enter') && state !== 'play' && canStart()) start();
  if (reels && state === 'play' && (e.code === 'KeyN' || e.code === 'PageDown')) nextReel();
  if (reels && e.code === 'Escape') exitReels();
});
addEventListener('keyup', e => keys[e.code] = false);
cv.addEventListener('mousedown', e => { if (state !== 'play') menuTap(e.clientX, e.clientY); else mouseJet = true; });
addEventListener('mouseup', () => mouseJet = false);
function handleTouches(e){
  e.preventDefault(); isTouch = true;
  touch.l = touch.r = touch.j = false; touch.any = e.touches.length > 0;
  const r = cv.getBoundingClientRect();
  for (const t of e.touches){
    const x = (t.clientX - r.left) / r.width * W;
    if (x < W*0.25) touch.l = true; else if (x < W*0.5) touch.r = true; else touch.j = true;
  }
}
cv.addEventListener('touchstart', e => { isTouch = true; if (state !== 'play'){ e.preventDefault(); menuTap(e.touches[0].clientX, e.touches[0].clientY); return; } handleTouches(e); }, {passive:false});
cv.addEventListener('touchmove', handleTouches, {passive:false});
cv.addEventListener('touchend', handleTouches, {passive:false});
cv.addEventListener('touchcancel', handleTouches, {passive:false});

// ===================== State =====================
let state = 'menu', paused = false, overT = 0;
let warp = null;
let mode = 'jet', gd = null, hc = null, ab = null, portals = [], lastPortal = 0, flash = 0, bonusT = 0;
let P, camY, plats, coins, zaps, missiles, fuels, shields, drones, parts, score, coinsRun, genY, lastZap, missileT, shake, time, newBest;
let best = +localStorage.getItem('jj_best') || 0;
let bank = +localStorage.getItem('jj_coins') || 0;
const FUELMAX = 100, START_Y = H - 160;
const canStart = () => state === 'menu' || performance.now() - overT > 700;

const stars = Array.from({length:140}, () => ({x:rand(0,W), y:rand(0,H), s:rand(0.6,2.2), d:rand(0.05,0.25), tw:rand(0,6)}));
const clouds = Array.from({length:9}, () => ({x:rand(-60,W), y:rand(0,H+200), s:rand(0.7,1.5), d:rand(0.2,0.45)}));

function resetWorld(){
  P = {x:W/2-18, y:START_Y, w:36, h:48, vx:0, vy:-12, face:1, fuel:FUELMAX, shield:0, inv:0, thr:false};
  camY = 0; plats = []; coins = []; zaps = []; missiles = []; fuels = []; shields = []; drones = []; parts = [];
  portals = []; lastPortal = -800; mode = 'jet'; gd = null; hc = null; flash = 0; bonusT = 0;
  score = 0; coinsRun = 0; shake = 0; time = 0; lastZap = 0; missileT = 300; newBest = false;
  plats.push({x:W/2-45, y:H-80, w:90, t:'n', vx:0});
  genY = H - 80;
  while (genY > -H) gen();
}
function start(){ resetWorld(); state = 'play'; paused = false; beep(520,.12,'triangle',.08,300); }

// ===================== Generation =====================
const diff = () => Math.min(1, (-genY) / 30000);
function gen(){
  const d = diff();
  genY -= 55 + Math.random()*(50 + d*75);
  const w = Math.max(52, 88 - d*30);
  let t = 'n'; const r = Math.random();
  if (r < 0.07) t = 's';
  else if (r < 0.07 + 0.12 + d*0.18) t = 'm';
  else if (r < 0.07 + 0.27 + d*0.3 && -genY > 800) t = 'b';
  plats.push({x:rand(0, W-w), y:genY, w, t, vx: t==='m' ? (1 + d*2.2) * (Math.random()<.5?-1:1) : 0, broken:false, fy:0});
  if (t === 'b'){ genY -= rand(25,45); plats.push({x:rand(0,W-w), y:genY, w, t:'n', vx:0}); }

  const r2 = Math.random();
  if (r2 < 0.13) coinRow(genY - 45);
  else if (r2 < 0.165) fuels.push({x:rand(20,W-50), y:genY-70, t:0});
  else if (r2 < 0.177) shields.push({x:rand(20,W-50), y:genY-70, t:0});

  const h = -genY / 10;
  if (h > 40 && genY < lastPortal - 900 && Math.random() < 0.08){
    portals.push({x:rand(50, W-50), y:genY-70, t:0, kind: ['gd','hc','ab'][Math.floor(Math.random()*3)]}); lastPortal = genY - 70;
  }
  if (h > 150 && Math.random() < 0.05 + d*0.12 && genY < lastZap - 280){
    const len = rand(110, 190), rot = Math.random() < 0.25 + d*0.4 ? rand(0.012,0.03)*(Math.random()<.5?-1:1) : 0;
    zaps.push({cx:rand(len/2, W-len/2), cy:genY-35, len, ang: Math.random()<.5 ? 0 : rand(0,Math.PI), rot});
    lastZap = genY - 35;
  }
  if (h > 80 && Math.random() < 0.025 + d*0.05){
    drones.push({x:rand(20,W-60), y:genY-90, vx:rand(1,2.2)*(Math.random()<.5?-1:1), ph:rand(0,6), alive:true, by:genY-90});
  }
}
function coinRow(y){
  const n = 5, sx = rand(40, W-40-(n-1)*32), arc = Math.random() < .5;
  for (let i=0;i<n;i++) coins.push({x:sx+i*32, y:y - (arc ? Math.sin(i/(n-1)*Math.PI)*45 : 0), t:rand(0,6)});
}

// ===================== Particles =====================
function burst(x,y,n,cols,spd=6){
  for (let i=0;i<n;i++){ const a=rand(0,Math.PI*2), s=rand(1,spd);
    parts.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s,life:rand(25,50),max:50,c:cols[i%cols.length],s:rand(2,5),g:0.15}); }
}

// ===================== Damage =====================
function hurt(){
  if (P.inv > 0) return;
  if (P.shield){ P.shield = 0; P.inv = 90; shake = 12; beep(260,.25,'square',.08,-150); if (mode==='hc'){ hcRespawn(); return; }
    if (mode==='gd') burst(GD_PX+20, gd.y+20, 24, ['#6cf','#aef','#fff']); else burst(P.x+P.w/2, P.y+P.h/2, 24, ['#6cf','#aef','#fff']); return; }
  die();
}
function die(){
  if (state !== 'play') return;
  if (reels){ reelEnd(false); return; }
  state = 'over'; overT = performance.now(); shake = 22;
  if (mode === 'gd') burst(GD_PX+20, gd.y+20, 50, ['#0ff','#ff0','#f0f','#fff'], 9); else if (mode === 'hc') burst((hc.w[0].x+hc.w[1].x)/2, (hc.w[0].y+hc.w[1].y)/2, 50, ['#f30','#f90','#ff0','#555'], 9); else burst(P.x+P.w/2, P.y+P.h/2, 50, ['#ff0','#f90','#f30','#fff'], 9);
  beep(110,.6,'sawtooth',.12,-80);
  bank += coinsRun; localStorage.setItem('jj_coins', bank);
  if (score > best){ best = score; newBest = true; localStorage.setItem('jj_best', best); }
}
function segDist(px,py,ax,ay,bx,by){
  const dx=bx-ax, dy=by-ay, t=clamp(((px-ax)*dx+(py-ay)*dy)/(dx*dx+dy*dy),0,1);
  return Math.hypot(px-(ax+t*dx), py-(ay+t*dy));
}

// ===================== Update =====================
function update(){
  time++;
  for (const p of parts){ p.x+=p.vx; p.y+=p.vy; p.vy+=p.g; p.vx*=0.97; p.life--; }
  parts = parts.filter(p => p.life > 0);
  shake *= 0.88;
  if (flash > 0) flash -= 0.04;
  if (reelAnim > 0) reelAnim = Math.max(0, reelAnim - 0.06);
  if (reels && state === 'play'){
    if (mode === 'reelEnd'){ reelEndT++; for (const p of parts){ p.x+=p.vx; p.y+=p.vy; p.vy+=p.g; p.life--; } parts = parts.filter(p=>p.life>0); return; }
    if (reelAnim > 0.1) return;
  }
  if (bonusT > 0) bonusT--;
  if (state !== 'play' || paused) return;
  if (mode === 'warp'){ updateWarp(); return; }
  if (mode === 'ab'){ updateAB(); return; }
  if (MINI[mode]){ MINI[mode].u(); return; }
  if (mode === 'gd'){ updateGD(); return; }
  if (mode === 'hc'){ updateHC(); return; }

  // --- Player movement ---
  const left  = keys.ArrowLeft || keys.KeyA || touch.l;
  const right = keys.ArrowRight || keys.KeyD || touch.r;
  const jetIn = keys.Space || keys.ArrowUp || keys.KeyW || mouseJet || touch.j;
  if (left){ P.vx -= 0.95; P.face = -1; }
  if (right){ P.vx += 0.95; P.face = 1; }
  P.vx = clamp(P.vx * 0.87, -8.5, 8.5);
  P.vy += 0.45;
  P.thr = jetIn && P.fuel > 0;
  if (P.thr){
    P.vy -= 1.08; if (P.vy < -11) P.vy = -11;
    P.fuel = Math.max(0, P.fuel - 0.55);
    const jx = P.x + P.w/2 - 16*P.face, jy = P.y + 44;
    for (let i=0;i<2;i++) parts.push({x:jx+rand(-3,3), y:jy, vx:rand(-0.8,0.8)+P.vx*0.2, vy:rand(4,7), life:rand(12,22), max:22, c:Math.random()<.5?'#ffd23f':'#ff6a1a', s:rand(3,6), g:0});
    if (time % 3 === 0) parts.push({x:jx, y:jy+8, vx:rand(-.5,.5), vy:rand(1.5,3), life:40, max:40, c:'rgba(180,180,190,0.5)', s:rand(5,9), g:-0.02});
  }
  P.vy = Math.min(P.vy, 16);
  const prevBottom = P.y + P.h;
  P.x += P.vx; P.y += P.vy;
  if (P.x + P.w < 0) P.x = W; if (P.x > W) P.x = -P.w;
  if (P.inv > 0) P.inv--;

  // --- Platforms ---
  for (const p of plats){
    if (p.t === 'm'){ p.x += p.vx; if (p.x < 0 || p.x + p.w > W) p.vx *= -1; }
    if (p.broken){ p.fy += 0.5; p.y += p.fy; continue; }
    if (P.vy > 0 && prevBottom <= p.y + 2 && P.y + P.h >= p.y && P.x + P.w - 8 > p.x && P.x + 8 < p.x + p.w){
      if (p.t === 'b'){ p.broken = true; beep(150,.15,'sawtooth',.06,-60); burst(p.x+p.w/2,p.y,10,['#8a5a2b','#c08a4a']); continue; }
      P.y = p.y - P.h;
      if (p.t === 's'){ P.vy = -22; p.sp = 12; beep(700,.25,'sine',.1,700); }
      else { P.vy = -13; beep(380,.08,'triangle',.07,250); }
      P.fuel = Math.min(FUELMAX, P.fuel + 6);
    }
    if (p.sp) p.sp--;
  }

  // --- Camera & score ---
  const target = P.y - H*0.42;
  if (target < camY) camY += (target - camY) * 0.25;
  score = Math.max(score, Math.floor((START_Y - P.y) / 10));
  if (reels && score >= REEL_GOAL){ reelEnd(true); return; }
  while (genY > camY - 250) gen();

  // --- Coins & pickups ---
  const pcx = P.x + P.w/2, pcy = P.y + P.h/2;
  for (const c of coins){ c.t += 0.12;
    if (!c.got && Math.hypot(c.x-pcx, c.y-pcy) < 26){ c.got = true; coinsRun++; beep(1200,.07,'square',.05,400); burst(c.x,c.y,5,['#ffe066','#fff']); } }
  for (const f of fuels){ f.t += 0.05;
    if (!f.got && Math.abs(f.x+14-pcx) < 30 && Math.abs(f.y+18-pcy) < 34){ f.got = true; P.fuel = Math.min(FUELMAX, P.fuel + 55); beep(500,.2,'triangle',.08,500); burst(f.x+14,f.y+18,12,['#ff4d4d','#ffd23f']); } }
  for (const s of shields){ s.t += 0.05;
    if (!s.got && Math.hypot(s.x+15-pcx, s.y+15-pcy) < 32){ s.got = true; P.shield = 1; beep(880,.3,'sine',.08,-300); burst(s.x+15,s.y+15,14,['#6cf','#fff']); } }

  // --- Dimensionslöcher ---
  for (const o of portals){ o.t += 0.08;
    if (!reels && Math.hypot(o.x - pcx, o.y - pcy) < 34){ startWarp(o.kind); return; } }

  // --- Zappers ---
  for (const z of zaps){
    z.ang += z.rot;
    const dx = Math.cos(z.ang)*z.len/2, dy = Math.sin(z.ang)*z.len/2;
    if (segDist(pcx, pcy, z.cx-dx, z.cy-dy, z.cx+dx, z.cy+dy) < 20){ hurt(); if (state!=='play') return; }
  }

  // --- Drones (stompable like Doodle Jump monsters) ---
  for (const d of drones){
    if (!d.alive){ d.y += 8; continue; }
    d.x += d.vx; if (d.x < 0 || d.x + 44 > W) d.vx *= -1;
    d.ph += 0.06; d.y = d.by + Math.sin(d.ph)*10;
    if (P.x + P.w - 6 > d.x && P.x + 6 < d.x + 44 && P.y + P.h > d.y && P.y < d.y + 26){
      if (P.vy > 0 && prevBottom <= d.y + 14){
        d.alive = false; P.vy = -15; coinsRun += 5; beep(300,.2,'square',.09,-200);
        burst(d.x+22,d.y+12,18,['#ff4d6d','#ffd23f','#fff']);
      } else { hurt(); if (state!=='play') return; }
    }
  }

  // --- Missiles (screen-space, like Jetpack Joyride) ---
  if (score > 400){
    missileT--;
    const maxM = score > 3000 ? 2 : 1;
    if (missileT <= 0 && missiles.length < maxM){
      const side = Math.random() < .5 ? 1 : -1;
      missiles.push({dir:side, x: side===1 ? -70 : W+70, sy: pcy - camY, warn: 130});
      missileT = rand(380, 650) - diff()*100;
      beep(900,.08,'square',.05);
    }
  }
  const psy = pcy - camY;
  for (const m of missiles){
    if (m.warn > 0){
      m.warn--; if (m.warn > 60) m.sy += (psy - m.sy) * 0.035;
      if (m.warn % 15 === 0) beep(m.warn < 60 ? 1300 : 950, .05, 'square', .04);
    } else {
      m.x += m.dir * 7;
      parts.push({x:m.x - m.dir*26, y:m.sy+camY, vx:-m.dir*rand(1,3), vy:rand(-.6,.6), life:18, max:18, c:Math.random()<.5?'#ffb03a':'#ff5a1a', s:rand(3,6), g:0});
      if (Math.abs(m.x - pcx) < 24 && Math.abs(m.sy - psy) < 15){ m.hit = true; burst(m.x, m.sy+camY, 20, ['#f90','#ff0','#888']); hurt(); if (state!=='play') return; }
    }
  }
  missiles = missiles.filter(m => !m.hit && m.x > -120 && m.x < W+120);

  // --- Cleanup ---
  const lim = camY + H + 120;
  plats = plats.filter(p => p.y < lim); coins = coins.filter(c => !c.got && c.y < lim);
  fuels = fuels.filter(f => !f.got && f.y < lim); shields = shields.filter(s => !s.got && s.y < lim);
  portals = portals.filter(o => o.y < lim);
  zaps = zaps.filter(z => z.cy - z.len < lim); drones = drones.filter(d => d.y < lim);

  // --- Death by falling ---
  if (P.y - camY > H + 40) die();
}


// ===================== Geometry-Dash-Dimension =====================
const GROUND = H - 200, GD_PX = 120, CUBE = 40, GD_SPEED = 7.2;
function buildGD(){
  const obs = []; let x = W + 260; const end = 7200;
  const spike = (x) => obs.push({t:'s', x});
  const block = (x, lvl) => obs.push({t:'b', x, lvl});
  while (x < end - 400){
    const r = Math.random();
    if (r < .2){ spike(x); x += 40; }
    else if (r < .36){ spike(x); spike(x+40); x += 80; }
    else if (r < .46){ spike(x); spike(x+40); spike(x+80); x += 120; }
    else if (r < .6){ block(x,0); spike(x+40); spike(x+80); block(x+120,0); x += 160; }
    else if (r < .75){ block(x,0); block(x+160,0); block(x+160,1); block(x+320,0); block(x+320,1); block(x+320,2);
                       obs.push({t:'c', x:x+340, y:GROUND-CUBE*3-60}); x += 360; }
    else if (r < .88){ obs.push({t:'p', x}); for (let k=0;k<3;k++) block(x+190,k); obs.push({t:'c', x:x+210, y:GROUND-200}); x += 230; }
    else { for (let i=0;i<4;i++) obs.push({t:'c', x:x+i*40, y:GROUND-100}); spike(x+40); spike(x+80); x += 160; }
    x += rand(230, 360);
  }
  return {obs, end};
}
function enterGD(){
  mode = 'gd'; flash = 1; parts = []; missiles = [];
  const lv = buildGD();
  gd = {y: GROUND - CUBE, vy: 0, rot: 0, ground: true, dist: 0, obs: lv.obs, end: lv.end, hue: rand(0,360)};
  beep(200,.5,'sawtooth',.1,1200); setTimeout(()=>beep(1400,.3,'sine',.08,-900),150);
}
function exitGD(){ exitDim(); }
let bonusFail = false;
function exitDim(fail){
  if (reels){ reelEnd(!fail); return; }
  mode = 'jet'; flash = 1; parts = []; bonusFail = !!fail;
  if (!fail) coinsRun += 25; bonusT = 150;
  P.y -= fail ? 0 : 1500; P.vx = 0; P.vy = -14; P.inv = 120; P.fuel = FUELMAX;
  camY = P.y - H*0.42;
  while (genY > camY - 250) gen();
  // sichere Landeplattform & Gefahren in der Nähe entfernen
  plats.push({x:P.x+P.w/2-50, y:P.y+P.h+160, w:100, t:'n', vx:0});
  zaps = zaps.filter(z => Math.abs(z.cy - P.y) > 350);
  drones = drones.filter(d => Math.abs(d.y - P.y) > 300);
  portals = portals.filter(o => Math.abs(o.y - P.y) > 500);
  missileT = 300;
  beep(1400,.4,'sine',.1,-1100); setTimeout(()=>beep(600,.3,'triangle',.08,900),180);
}
function updateGD(){
  const jump = keys.Space || keys.ArrowUp || keys.KeyW || mouseJet || touch.any || touch.j;
  if (P.inv > 0) P.inv--;
  gd.dist += GD_SPEED;
  const prevBottom = gd.y + CUBE;
  gd.vy = Math.min(gd.vy + 1.0, 18);
  gd.y += gd.vy;
  gd.ground = false;
  if (gd.y + CUBE >= GROUND){ gd.y = GROUND - CUBE; gd.vy = 0; gd.ground = true; }
  const px1 = GD_PX, px2 = GD_PX + CUBE;
  for (const o of gd.obs){
    const sx = o.x - gd.dist;
    if (sx > px2 + 10 || sx + 40 < px1 - 10) continue;
    if (o.t === 'b'){
      const top = GROUND - CUBE*(o.lvl+1);
      if (px2 > sx && px1 < sx + 40 && gd.y + CUBE > top && gd.y < top + 40){
        if (gd.vy >= 0 && prevBottom <= top + 10){ gd.y = top - CUBE; gd.vy = 0; gd.ground = true; }
        else if (P.inv <= 0){ hurt(); if (state !== 'play') return; }
      }
    } else if (o.t === 's'){
      if (px2 - 6 > sx + 12 && px1 + 6 < sx + 28 && gd.y + CUBE > GROUND - 28){ hurt(); if (state !== 'play') return; }
    } else if (o.t === 'p'){
      if (px2 > sx && px1 < sx + 40 && gd.y + CUBE >= GROUND - 10 && !o.used){ o.used = true; gd.vy = -22; gd.ground = false; beep(900,.2,'sine',.08,600);
        burst(sx+20, GROUND-6, 12, ['#ffe94a','#fff'], 5); }
    } else if (o.t === 'c' && !o.got){
      if (Math.abs(sx+20 - (px1+20)) < 30 && Math.abs(o.y - (gd.y+20)) < 30){ o.got = true; coinsRun++; beep(1200,.07,'square',.05,400); burst(sx+20,o.y,6,['#ffe066','#fff']); }
    }
  }
  if (gd.ground){
    gd.rot = Math.round(gd.rot / (Math.PI/2)) * (Math.PI/2);
    if (jump){ gd.vy = -15.5; gd.ground = false; beep(520,.06,'square',.04,200); }
    if (time % 2 === 0) parts.push({x:GD_PX+2, y:gd.y+CUBE-2, vx:rand(-4,-2), vy:rand(-1.5,0), life:20, max:20, c:`hsl(${gd.hue},100%,70%)`, s:rand(2,4), g:0.1});
  } else {
    gd.rot += 0.13;
    parts.push({x:GD_PX+20, y:gd.y+20, vx:-GD_SPEED*0.4, vy:0, life:14, max:14, c:`hsla(${gd.hue},100%,70%,.6)`, s:7, g:0});
  }
  for (const p of parts) p.x -= GD_SPEED*0.3;
  if (gd.dist >= gd.end) exitGD();
}
function drawPortal(o){
  if (reels) return;
  ctx.save(); ctx.translate(o.x, o.y);
  const g = ctx.createRadialGradient(0,0,2,0,0,46);
  const PC = {gd:['#1a0033','rgba(170,0,255,.7)','rgba(0,255,255,0)','rgba(0,255,255,.8)','rgba(255,80,255,.8)'],
              hc:['#2a1400','rgba(255,140,0,.75)','rgba(80,255,80,0)','rgba(120,255,90,.85)','rgba(255,170,40,.85)'],
              ab:['#2a0000','rgba(230,20,20,.8)','rgba(255,220,0,0)','rgba(255,220,40,.9)','rgba(255,60,60,.9)']}[o.kind];
  g.addColorStop(0,'#000'); g.addColorStop(.45, PC[0]); g.addColorStop(.75, PC[1]); g.addColorStop(1, PC[2]);
  ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0,0,46,0,7); ctx.fill();
  ctx.lineWidth = 3;
  for (let i=0;i<4;i++){
    ctx.strokeStyle = i%2 ? PC[3] : PC[4];
    ctx.beginPath(); ctx.arc(0,0,14+i*7, o.t*(i%2?-1.5:1.2)+i, o.t*(i%2?-1.5:1.2)+i+3.6); ctx.stroke();
  }
  ctx.fillStyle = '#fff'; ctx.font = 'bold 11px Segoe UI'; ctx.textAlign = 'center';
  ctx.globalAlpha = .7 + .3*Math.sin(o.t*2); ctx.fillText('???', 0, -52);
  ctx.restore();
}
function drawGD(){
  const hue = (gd.hue + gd.dist*0.02) % 360;
  const bg = ctx.createLinearGradient(0,0,0,H);
  bg.addColorStop(0, `hsl(${hue},75%,22%)`); bg.addColorStop(1, `hsl(${(hue+40)%360},80%,38%)`);
  ctx.fillStyle = bg; ctx.fillRect(0,0,W,H);
  // Parallax-Raster
  ctx.strokeStyle = 'rgba(255,255,255,.07)'; ctx.lineWidth = 2;
  const off = posmod(-gd.dist*0.3, 80);
  for (let x=off; x<W; x+=80) for (let y=40; y<GROUND; y+=80) ctx.strokeRect(x+6, y+6, 68, 68);
  // Boden
  const gg = ctx.createLinearGradient(0,GROUND,0,H);
  gg.addColorStop(0, `hsl(${hue},70%,30%)`); gg.addColorStop(1, `hsl(${hue},70%,12%)`);
  ctx.fillStyle = gg; ctx.fillRect(0,GROUND,W,H-GROUND);
  ctx.strokeStyle = 'rgba(0,0,0,.25)';
  for (let x=posmod(-gd.dist,80); x<W; x+=80) ctx.strokeRect(x, GROUND, 80, 80);
  ctx.save(); ctx.shadowColor = '#fff'; ctx.shadowBlur = 12; ctx.fillStyle = '#fff'; ctx.fillRect(0, GROUND-1, W, 3); ctx.restore();
  // Ziel-Portal
  const ex = gd.end - gd.dist + GD_PX;
  if (ex < W + 80){
    ctx.save(); ctx.shadowColor = '#0f8'; ctx.shadowBlur = 25; ctx.strokeStyle = '#3f9'; ctx.lineWidth = 8;
    ctx.beginPath(); ctx.ellipse(ex+20, GROUND-80, 28, 80, 0, 0, 7); ctx.stroke();
    ctx.fillStyle = 'rgba(80,255,150,.25)'; ctx.fill(); ctx.restore();
  }
  // Hindernisse
  for (const o of gd.obs){
    const sx = o.x - gd.dist; if (sx < -60 || sx > W+60) continue;
    if (o.t === 'b'){
      const top = GROUND - CUBE*(o.lvl+1);
      ctx.fillStyle = 'rgba(0,0,0,.75)'; ctx.fillRect(sx, top, 40, 40);
      ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.strokeRect(sx+1, top+1, 38, 38);
      ctx.strokeStyle = `hsla(${hue},100%,70%,.6)`; ctx.strokeRect(sx+8, top+8, 24, 24);
    } else if (o.t === 's'){
      ctx.save(); ctx.shadowColor = `hsl(${hue},100%,70%)`; ctx.shadowBlur = 10;
      ctx.fillStyle = '#000'; ctx.strokeStyle = '#fff'; ctx.lineWidth = 2;
      ctx.beginPath(); ctx.moveTo(sx+2, GROUND); ctx.lineTo(sx+20, GROUND-38); ctx.lineTo(sx+38, GROUND); ctx.closePath(); ctx.fill(); ctx.stroke(); ctx.restore();
    } else if (o.t === 'p'){
      ctx.save(); ctx.shadowColor = '#ff0'; ctx.shadowBlur = 15; ctx.fillStyle = o.used ? '#aa9' : '#ffe94a';
      ctx.beginPath(); ctx.ellipse(sx+20, GROUND, 18, 8, 0, Math.PI, 0); ctx.fill(); ctx.restore();
    } else if (o.t === 'c' && !o.got){
      drawCoin({x:sx+20, y:o.y, t:time*0.12});
    }
  }
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.fillRect(p.x-p.s/2,p.y-p.s/2,p.s,p.s); }
  ctx.globalAlpha = 1;
  // Würfel
  if (state === 'play' && !(P.inv > 0 && Math.floor(P.inv/5)%2)){
    ctx.save(); ctx.translate(GD_PX+20, gd.y+20); ctx.rotate(gd.rot);
    ctx.shadowColor = `hsl(${(hue+180)%360},100%,60%)`; ctx.shadowBlur = 15;
    ctx.fillStyle = '#ffd23f'; ctx.fillRect(-20,-20,40,40); ctx.shadowBlur = 0;
    ctx.strokeStyle = '#000'; ctx.lineWidth = 3; ctx.strokeRect(-20,-20,40,40);
    ctx.fillStyle = '#3cf'; ctx.fillRect(-12,-12,24,24); ctx.strokeRect(-12,-12,24,24);
    ctx.fillStyle = '#000'; ctx.fillRect(-8,-6,5,5); ctx.fillRect(3,-6,5,5); ctx.fillRect(-7,4,14,3);
    ctx.restore();
    if (P.shield){ ctx.globalAlpha = .35; ctx.fillStyle = '#6cf'; ctx.beginPath(); ctx.arc(GD_PX+20, gd.y+20, 34, 0, 7); ctx.fill(); ctx.globalAlpha = 1; }
  }
  // Fortschritt
  const pr = clamp(gd.dist / gd.end, 0, 1);
  ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W/2-130, 16, 260, 16, 8); ctx.fill();
  ctx.fillStyle = '#3f9'; if (pr > 0.02){ rr(W/2-130, 16, 260*pr, 16, 8); ctx.fill(); }
  ctx.fillStyle = '#fff'; ctx.font = 'bold 13px Segoe UI'; ctx.textAlign = 'center'; ctx.fillText(Math.floor(pr*100) + '%', W/2, 29);
  ctx.font = 'bold 16px Segoe UI'; ctx.fillText('DIMENSION: GEOMETRY DASH', W/2, 58);
  ctx.font = '13px Segoe UI'; ctx.fillStyle = 'rgba(255,255,255,.75)'; ctx.fillText('Springen: Leertaste / ↑ / Klick / Tippen', W/2, reels ? 80 : H-60);
  ctx.fillStyle = '#ffd23f'; ctx.font = 'bold 15px Segoe UI'; ctx.textAlign = 'left'; ctx.fillText('● ' + coinsRun, 14, 29);
}


// ===================== Hill-Climb-Dimension =====================
const WR = 16, AXLE = 72, HC_END = 7000;
function ter(x){
  if (x < 350) return 500;
  const a = Math.min(1, (x-350) / 2500), u = x - 350;
  return 500 - (Math.sin(u*0.004)*60 + Math.sin(u*0.011+1)*28*a + Math.sin(u*0.0021)*95*a) + Math.sin(u*0.004)*0 ;
}
const slopeAt = x => (ter(x+1) - ter(x-1)) / 2;
function enterHC(){
  mode = 'hc'; flash = 1; parts = []; missiles = [];
  const coinsL = [], cans = [];
  for (let x = 600; x < HC_END - 200; x += rand(280, 420)){
    if (Math.random() < .6) for (let i=0;i<4;i++) coinsL.push({x:x+i*34, t:rand(0,6)});
  }
  for (let x = 1100; x < HC_END; x += 1100) cans.push({x, t:0});
  hc = {w:[{x:200,y:500-WR,px:200,py:500-WR,c:false},{x:200+AXLE,y:500-WR,px:200+AXLE,py:500-WR,c:false}],
        fuel:100, coins:coinsL, cans, camX:0, camY:0, spin:0, stall:0, flips:0, airRot:0, air:0};
  hc.camY = 500 - H*0.55;
  beep(120,.5,'sawtooth',.1,300); setTimeout(()=>beep(80,.4,'sawtooth',.08,60),200);
}
function hcRespawn(){
  const cx = (hc.w[0].x + hc.w[1].x)/2;
  hc.w[0] = {x:cx-AXLE/2, y:ter(cx-AXLE/2)-WR-30, px:cx-AXLE/2, py:ter(cx-AXLE/2)-WR-30};
  hc.w[1] = {x:cx+AXLE/2, y:ter(cx+AXLE/2)-WR-30, px:cx+AXLE/2, py:ter(cx+AXLE/2)-WR-30};
  P.inv = 90; shake = 10; beep(260,.25,'square',.08,-150);
  burst(cx, ter(cx)-40, 24, ['#6cf','#aef','#fff']);
}
function rotWheels(a){
  const cx = (hc.w[0].x+hc.w[1].x)/2, cy = (hc.w[0].y+hc.w[1].y)/2, c = Math.cos(a), sn = Math.sin(a);
  for (const w of hc.w){ const dx=w.x-cx, dy=w.y-cy; w.x = cx + dx*c - dy*sn; w.y = cy + dx*sn + dy*c; }
}
function updateHC(){
  const gas = keys.ArrowRight || keys.KeyD || keys.Space || keys.ArrowUp || keys.KeyW || mouseJet || touch.r || touch.j;
  const brake = keys.ArrowLeft || keys.KeyA || keys.ArrowDown || keys.KeyS || touch.l;
  if (P.inv > 0) P.inv--;
  const useGas = gas && hc.fuel > 0;
  if (useGas) hc.fuel = Math.max(0, hc.fuel - 0.05);
  let contact = 0;
  for (const w of hc.w){
    let vx = (w.x - w.px)*0.996, vy = (w.y - w.py)*0.996;
    w.px = w.x; w.py = w.y;
    w.x += vx; w.y += vy + 0.45;
    w.c = false;
  }
  for (let it=0; it<3; it++){
    const a = hc.w[0], b = hc.w[1];
    const dx = b.x-a.x, dy = b.y-a.y, d = Math.hypot(dx,dy) || 1, k = (d-AXLE)/d/2;
    a.x += dx*k; a.y += dy*k; b.x -= dx*k; b.y -= dy*k;
  }
  for (const w of hc.w){
    const sl = slopeAt(w.x), len = Math.hypot(1, sl), gy = ter(w.x) - WR*len;
    if (w.y >= gy){
      w.c = true; contact++;
      let vx = w.x - w.px, vy = w.y - w.py;
      w.y = gy;
      const tx = 1/len, ty = sl/len, nx = sl/len, ny = -1/len;
      let vt = vx*tx + vy*ty, vn = vx*nx + vy*ny;
      if (vn < 0) vn = -vn*0.38;
      const trac = clamp(1.15 - Math.abs(sl)*1.1, 0.4, 1);
      if (useGas){ vt = Math.min(vt + 0.24*trac, 10.5); if (trac < 0.6 && time % 4 === 0) parts.push({x:w.x, y:w.y+WR-2, vx:-rand(2,4), vy:-rand(1,3), life:22, max:22, c:'#7a5230', s:rand(2,4), g:0.2}); }
      else if (brake) vt = vt > 0 ? Math.max(0, vt - 0.28*trac) : Math.max(vt - 0.18*trac, -6);
      else vt *= 0.998;
      vx = tx*vt + nx*vn; vy = ty*vt + ny*vn;
      w.px = w.x - vx; w.py = w.y - vy;
    }
  }
  // Neigen (Gas = Nase hoch, Bremse = Nase runter)
  const tilt = contact ? 0.0052 : 0.0095;
  const before = Math.atan2(hc.w[1].y-hc.w[0].y, hc.w[1].x-hc.w[0].x);
  if (useGas) rotWheels(-tilt);
  if (brake) rotWheels(tilt);
  const ang = Math.atan2(hc.w[1].y-hc.w[0].y, hc.w[1].x-hc.w[0].x);
  // Flip-Erkennung (Saltos geben Bonus)
  if (!contact){ hc.air++; let da = ang - before; hc.airRot += Math.atan2(Math.sin(ang - hc.lastAng || 0), Math.cos(ang - hc.lastAng || 0)); }
  else {
    if (hc.air > 20 && Math.abs(hc.airRot) > Math.PI*1.7){ const n = Math.round(Math.abs(hc.airRot)/(Math.PI*2)) || 1; coinsRun += 10*n; hc.flips += n; hc.msg = (n>1? n+'x ':'') + 'SALTO! +' + 10*n; hc.msgT = 90; beep(900,.25,'triangle',.08,600); }
    hc.air = 0; hc.airRot = 0;
  }
  hc.lastAng = ang;
  if (hc.msgT) hc.msgT--;
  const cx = (hc.w[0].x+hc.w[1].x)/2, cy = (hc.w[0].y+hc.w[1].y)/2;
  hc.spin += (hc.w[1].x - hc.w[1].px) / WR;
  // Kopf berührt Boden -> Crash
  const hx = cx + Math.sin(ang)*40, hy = cy - Math.cos(ang)*40;
  if (P.inv <= 0 && hy > ter(hx) - 6){ hurt(); if (state !== 'play') return; }
  // Auspuff / Staub
  if (useGas && time % 2 === 0) parts.push({x:hc.w[0].x - Math.cos(ang)*14, y:hc.w[0].y - 14, vx:-Math.cos(ang)*2+rand(-.5,.5), vy:rand(-1.5,-.5), life:30, max:30, c:'rgba(120,120,120,.6)', s:rand(3,6), g:-0.03});
  if (contact && Math.abs(hc.w[0].x-hc.w[0].px) > 5 && time % 3 === 0) parts.push({x:hc.w[0].x, y:hc.w[0].y+WR-2, vx:-rand(1,3), vy:-rand(1,3), life:25, max:25, c:'#9b6b3a', s:rand(2,4), g:0.2});
  // Einsammeln
  for (const c of hc.coins){ c.t += 0.12; if (!c.got && Math.abs(c.x - cx) < 34 && Math.abs((ter(c.x)-45) - cy) < 50){ c.got = true; coinsRun++; beep(1200,.07,'square',.05,400); burst(c.x, ter(c.x)-45, 5, ['#ffe066','#fff']); } }
  for (const f of hc.cans){ f.t += 0.05; if (!f.got && Math.abs(f.x - cx) < 40){ f.got = true; hc.fuel = 100; beep(500,.2,'triangle',.08,500); burst(f.x, ter(f.x)-40, 14, ['#ff4d4d','#ffd23f']); } }
  // Tank leer & stehen -> vorbei
  const sp = Math.abs(hc.w[1].x - hc.w[1].px);
  if (hc.fuel <= 0 && sp < 0.4){ hc.stall++; if (hc.stall > 120){ die(); return; } } else hc.stall = 0;
  // Kamera
  hc.camX += (cx - 170 - hc.camX) * 0.15;
  hc.camY += (cy - H*0.55 - hc.camY) * 0.1;
  if (cx >= HC_END) exitDim();
}
function drawWheel(x, y, rot){
  ctx.save(); ctx.translate(x, y); ctx.rotate(rot);
  ctx.fillStyle = '#1b1b1b'; ctx.beginPath(); ctx.arc(0,0,WR,0,7); ctx.fill();
  ctx.strokeStyle = '#333'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(0,0,WR-2,0,7); ctx.stroke();
  ctx.fillStyle = '#bfc5cc'; ctx.beginPath(); ctx.arc(0,0,7,0,7); ctx.fill();
  ctx.strokeStyle = '#7a8088'; ctx.lineWidth = 2;
  for (let i=0;i<5;i++){ const a=i*Math.PI*2/5; ctx.beginPath(); ctx.moveTo(0,0); ctx.lineTo(Math.cos(a)*7, Math.sin(a)*7); ctx.stroke(); }
  ctx.restore();
}
function drawHC(){
  // Himmel
  const sk = ctx.createLinearGradient(0,0,0,H);
  sk.addColorStop(0,'#4fb3ff'); sk.addColorStop(1,'#d6f1ff');
  ctx.fillStyle = sk; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = '#fff6b0'; ctx.beginPath(); ctx.arc(380, 110, 40, 0, 7); ctx.fill();
  // Parallax-Berge
  for (const [f, col, amp, base] of [[0.2,'#8fb9d9',90,420],[0.45,'#6aa87a',60,520]]){
    ctx.fillStyle = col; ctx.beginPath(); ctx.moveTo(0,H);
    for (let x=0; x<=W; x+=10){ const wx = x + hc.camX*f; ctx.lineTo(x, base - (Math.sin(wx*0.006)*amp + Math.sin(wx*0.017)*amp*0.3) - hc.camY*f*0.3); }
    ctx.lineTo(W,H); ctx.fill();
  }
  ctx.save(); ctx.translate(-hc.camX, -hc.camY);
  // Boden
  const x0 = hc.camX - 20, x1 = hc.camX + W + 20;
  ctx.beginPath(); ctx.moveTo(x0, hc.camY + H + 50);
  for (let x=x0; x<=x1; x+=8) ctx.lineTo(x, ter(x));
  ctx.lineTo(x1, hc.camY + H + 50); ctx.closePath();
  const dg = ctx.createLinearGradient(0, hc.camY+200, 0, hc.camY+H);
  dg.addColorStop(0,'#8a5a2b'); dg.addColorStop(1,'#4a2f15');
  ctx.fillStyle = dg; ctx.fill();
  ctx.strokeStyle = '#3fae3f'; ctx.lineWidth = 12; ctx.beginPath();
  for (let x=x0; x<=x1; x+=8) x===x0 ? ctx.moveTo(x, ter(x)+4) : ctx.lineTo(x, ter(x)+4);
  ctx.stroke();
  ctx.strokeStyle = '#6ad86a'; ctx.lineWidth = 3; ctx.beginPath();
  for (let x=x0; x<=x1; x+=8) x===x0 ? ctx.moveTo(x, ter(x)) : ctx.lineTo(x, ter(x));
  ctx.stroke();
  // Steine
  for (let x = Math.floor(x0/90)*90; x < x1; x += 90){ const r = (Math.sin(x*12.9898)*43758.5453)%1; if (r > .4){ ctx.fillStyle='rgba(60,35,15,.5)'; ctx.beginPath(); ctx.ellipse(x, ter(x)+30+r*40, 8+r*6, 5, 0, 0, 7); ctx.fill(); } }
  // Distanzschilder
  for (let m = Math.ceil(x0/1000)*1000; m < x1; m += 1000){ if (m <= 0) continue;
    ctx.fillStyle='#6b4423'; ctx.fillRect(m-2, ter(m)-50, 4, 50);
    ctx.fillStyle='#fff'; rr(m-24, ter(m)-70, 48, 22, 4); ctx.fill();
    ctx.fillStyle='#333'; ctx.font='bold 12px Segoe UI'; ctx.textAlign='center'; ctx.fillText((m/10)+' m', m, ter(m)-54); }
  // Ziel
  if (HC_END > x0 && HC_END < x1 + 50){
    const gy = ter(HC_END);
    ctx.fillStyle='#ddd'; ctx.fillRect(HC_END-3, gy-110, 6, 110);
    for (let i=0;i<4;i++) for (let j=0;j<3;j++){ ctx.fillStyle = (i+j)%2 ? '#000' : '#fff'; ctx.fillRect(HC_END+3+i*12, gy-110+j*12, 12, 12); }
    ctx.save(); ctx.shadowColor='#3f9'; ctx.shadowBlur=25; ctx.strokeStyle='#3f9'; ctx.lineWidth=6;
    ctx.beginPath(); ctx.ellipse(HC_END+80, gy-70, 26, 65, 0, 0, 7); ctx.stroke(); ctx.restore();
  }
  // Münzen & Kanister
  for (const c of hc.coins) if (!c.got && c.x > x0 && c.x < x1) drawCoin({x:c.x, y:ter(c.x)-45, t:c.t});
  for (const f of hc.cans) if (!f.got && f.x > x0-40 && f.x < x1) drawFuel({x:f.x-14, y:ter(f.x)-60, t:f.t});
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); }
  ctx.globalAlpha = 1;
  // Auto
  if (state === 'play' && !(P.inv > 0 && Math.floor(P.inv/5)%2)){
    const a = hc.w[0], b = hc.w[1], ang = Math.atan2(b.y-a.y, b.x-a.x), cx=(a.x+b.x)/2, cy=(a.y+b.y)/2;
    ctx.save(); ctx.translate(cx, cy); ctx.rotate(ang);
    // Federung
    ctx.strokeStyle='#555'; ctx.lineWidth=4; ctx.beginPath(); ctx.moveTo(-AXLE/2,0); ctx.lineTo(-AXLE/2+6,-12); ctx.moveTo(AXLE/2,0); ctx.lineTo(AXLE/2-6,-12); ctx.stroke();
    // Fahrer
    ctx.fillStyle='#3a86ff'; rr(-12,-34,14,18,4); ctx.fill();
    ctx.fillStyle='#f2c9a0'; ctx.beginPath(); ctx.arc(-5,-40,9,0,7); ctx.fill();
    ctx.fillStyle='#e63946'; ctx.beginPath(); ctx.arc(-5,-42,9.5,Math.PI,0); ctx.fill(); ctx.fillRect(-5,-44,12,3);
    ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(-1,-40,1.6,0,7); ctx.fill();
    // Überrollbügel
    ctx.strokeStyle='#333'; ctx.lineWidth=3; ctx.beginPath(); ctx.moveTo(-22,-14); ctx.lineTo(-20,-50); ctx.lineTo(4,-50); ctx.lineTo(10,-14); ctx.stroke();
    // Karosserie
    const bg = ctx.createLinearGradient(0,-22,0,0); bg.addColorStop(0,'#ff5a4d'); bg.addColorStop(1,'#b3221a');
    ctx.fillStyle = bg; ctx.beginPath(); ctx.moveTo(-46,-4); ctx.lineTo(-44,-20); ctx.lineTo(16,-20); ctx.lineTo(44,-12); ctx.lineTo(48,-2); ctx.lineTo(-46,-2); ctx.closePath(); ctx.fill();
    ctx.fillStyle='#ffe066'; ctx.fillRect(42,-10,6,4);
    ctx.fillStyle='rgba(255,255,255,.35)'; ctx.fillRect(-40,-18,50,3);
    ctx.fillStyle='#222'; ctx.fillRect(-48,-8,6,5);
    ctx.restore();
    drawWheel(a.x, a.y, hc.spin); drawWheel(b.x, b.y, hc.spin);
    if (P.shield){ ctx.globalAlpha=.3; ctx.fillStyle='#6cf'; ctx.beginPath(); ctx.arc(cx,cy-15,55,0,7); ctx.fill(); ctx.globalAlpha=1; }
  }
  ctx.restore();
  // HUD
  const cx = (hc.w[0].x+hc.w[1].x)/2, pr = clamp(cx / HC_END, 0, 1);
  ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W/2-130, 16, 260, 16, 8); ctx.fill();
  ctx.fillStyle = '#3f9'; if (pr > 0.02){ rr(W/2-130, 16, 260*pr, 16, 8); ctx.fill(); }
  ctx.fillStyle = '#fff'; ctx.font = 'bold 13px Segoe UI'; ctx.textAlign = 'center'; ctx.fillText(Math.floor(cx/10) + ' / ' + (HC_END/10) + ' m', W/2, 29);
  ctx.fillStyle = '#1a3a1a'; ctx.font = 'bold 16px Segoe UI'; ctx.fillText('DIMENSION: HILL CLIMB', W/2, 58);
  // Tankanzeige
  ctx.fillStyle='rgba(0,0,0,.4)'; rr(12, 70, 150, 40, 10); ctx.fill();
  ctx.fillStyle='#fff'; ctx.font='bold 12px Segoe UI'; ctx.textAlign='left'; ctx.fillText('⛽ TANK', 22, 86);
  ctx.fillStyle='rgba(255,255,255,.25)'; rr(22, 92, 130, 10, 5); ctx.fill();
  ctx.fillStyle = hc.fuel > 40 ? '#4cd964' : hc.fuel > 15 ? '#ffd23f' : (time%20<10 ? '#ff3b30' : '#a00');
  if (hc.fuel > 1){ rr(22, 92, 130*hc.fuel/100, 10, 5); ctx.fill(); }
  ctx.fillStyle = '#ffd23f'; ctx.font = 'bold 15px Segoe UI'; ctx.fillText('● ' + coinsRun, 14, 29);
  if (hc.fuel <= 0){ ctx.textAlign='center'; ctx.fillStyle='#ff3b30'; ctx.font='900 28px Segoe UI'; ctx.fillText('TANK LEER!', W/2, 160); }
  if (hc.msgT){ ctx.textAlign='center'; ctx.globalAlpha=Math.min(1,hc.msgT/30); ctx.fillStyle='#ff9f1a'; ctx.font='900 30px Segoe UI'; ctx.fillText(hc.msg, W/2, 200); ctx.globalAlpha=1; }
  ctx.textAlign='center'; ctx.font='13px Segoe UI'; ctx.fillStyle= reels ? 'rgba(0,0,0,0)' : 'rgba(0,0,0,.6)';
  ctx.fillText(isTouch ? 'Links tippen = Bremse · Rechts = Gas' : '→ / D / Leertaste = Gas · ← / A = Bremse · in der Luft neigen', W/2, H-24);
}


// ===================== Zwischensequenz: Flug durch die Welten =====================
const SEG = [ {s:22, e:122, n:'coc'}, {s:144, e:244, n:'cr'}, {s:266, e:366, n:'mc'} ];
const WARP_DUR = 390;
function startWarp(kind){
  mode = 'warp'; flash = 1; parts = []; missiles = [];
  // Clash-of-Clans-Dorf
  const coc = [];
  for (let y = -900; y < H + 100; y += 150){
    for (let x = 30; x < W - 30; x += 150){
      if (Math.random() < .15) continue;
      const r = Math.random();
      coc.push({x: x + rand(0,40), y: y + rand(0,40), t: r < .15 ? 'th' : r < .35 ? 'gold' : r < .55 ? 'elix' : r < .72 ? 'cannon' : r < .85 ? 'hut' : 'tree'});
    }
  }
  const troops = Array.from({length:14}, () => ({x:rand(20,W-20), y:rand(-700,H), c: Math.random()<.5 ? 'barb' : 'arch', ph:rand(0,6)}));
  // Minecraft-Gelände
  const mcH = []; let hh = 7;
  for (let i=0;i<120;i++){ hh = clamp(hh + (Math.random()<.35 ? (Math.random()<.5?-1:1) : 0), 4, 11); mcH.push(hh); }
  const mcTrees = []; for (let i=3;i<118;i+=rand(4,8)|0) mcTrees.push(i);
  const mcOres = Array.from({length:60}, () => ({i:rand(0,120)|0, d:rand(3,9)|0, c:['#5ff','#f33','#fd3','#3c3'][rand(0,4)|0]}));
  warp = {kind, t:0, coc, troops, mcH, mcTrees, mcOres, units: Array.from({length:10}, () => ({lane: Math.random()<.5?0:1, y: rand(150,950), dir: Math.random()<.5?1:-1, c: Math.random()<.5?'#3a7bff':'#ff4040'}))};
  beep(180,.6,'sawtooth',.09,900); setTimeout(()=>beep(900,.5,'sine',.07,-600),250);
}
function updateWarp(){
  warp.t++;
  const sk = keys.Enter || keys.KeyX;
  if (warp.t >= WARP_DUR || (sk && warp.t > 20)){
    const k = warp.kind; warp = null; keys.Enter = keys.KeyX = false;
    if (k === 'hc') enterHC(); else if (k === 'ab') enterAB(); else enterGD();
    return;
  }
  for (const s of SEG) if (warp.t === s.s + 2){ flash = 0.6; beep(s.n==='mc'?330:s.n==='cr'?660:520,.25,'triangle',.07,300); }
  // Jetpack-Flammen
  const bx = W/2 - 16, by = H/2 + 20 + Math.sin(warp.t*0.1)*8;
  parts.push({x:bx+rand(-3,3), y:by, vx:rand(-.6,.6), vy:rand(4,7), life:rand(10,18), max:18, c:Math.random()<.5?'#ffd23f':'#ff6a1a', s:rand(3,6), g:0});
  for (const p of parts){ p.x += p.vx; p.y += p.vy; p.life--; }
  parts = parts.filter(p => p.life > 0);
}
// ---- Tunnel ----
function drawTunnel(a){
  if (a <= 0) return;
  ctx.save(); ctx.globalAlpha = a;
  const hc = warp.kind === 'hc';
  ctx.fillStyle = '#05000f'; ctx.fillRect(0,0,W,H);
  const cx = W/2, cy = H/2, t = warp.t;
  for (let i=0;i<40;i++){
    const ang = i*2.4 + t*0.02, d = ((i*37 + t*14) % 520);
    ctx.strokeStyle = `rgba(255,255,255,${d/520})`; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(cx+Math.cos(ang)*d*0.6, cy+Math.sin(ang)*d*0.6); ctx.lineTo(cx+Math.cos(ang)*d, cy+Math.sin(ang)*d); ctx.stroke();
  }
  for (let i=0;i<9;i++){
    const r = ((i*60 + t*9) % 540);
    ctx.strokeStyle = warp.kind === 'ab' ? (i%2?'rgba(255,220,40,.7)':'rgba(255,60,60,.7)') : hc ? (i%2?'rgba(120,255,90,.7)':'rgba(255,170,40,.7)') : (i%2?'rgba(0,255,255,.7)':'rgba(255,80,255,.7)');
    ctx.lineWidth = 3 + r/60;
    ctx.beginPath(); ctx.arc(cx, cy, r, t*0.05+i, t*0.05+i+4.2); ctx.stroke();
  }
  ctx.restore();
}
// ---- Clash of Clans ----
function cocBuilding(o, y){
  const x = o.x;
  ctx.fillStyle = 'rgba(0,0,0,.18)'; ctx.beginPath(); ctx.ellipse(x+4, y+28, 34, 14, 0, 0, 7); ctx.fill();
  // Mauer
  ctx.fillStyle = '#8d8d8d'; ctx.strokeStyle = '#5c5c5c'; ctx.lineWidth = 1;
  for (let k=-2;k<=2;k++){ for (const [wx,wy] of [[x+k*16-8,y-48],[x+k*16-8,y+38],[x-48,y+k*16-8],[x+38,y+k*16-8]]){ ctx.fillRect(wx,wy,12,12); ctx.strokeRect(wx,wy,12,12); } }
  if (o.t === 'th'){
    ctx.fillStyle = '#c99a5b'; rr(x-28,y-10,56,36,4); ctx.fill();
    ctx.fillStyle = '#8b5a2b'; ctx.fillRect(x-6,y+8,12,18);
    ctx.fillStyle = '#d23a2a'; ctx.beginPath(); ctx.moveTo(x-34,y-8); ctx.lineTo(x,y-38); ctx.lineTo(x+34,y-8); ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#ffd23f'; ctx.fillRect(x-2,y-52,3,16); ctx.fillStyle = '#3a7bff'; ctx.fillRect(x+1,y-52,12,8);
  } else if (o.t === 'gold'){
    ctx.fillStyle = '#7a5230'; rr(x-22,y-6,44,30,6); ctx.fill();
    ctx.fillStyle = '#ffcc33'; ctx.beginPath(); ctx.arc(x,y-6,16,Math.PI,0); ctx.fill();
    ctx.fillStyle = '#fff3a0'; ctx.beginPath(); ctx.arc(x-5,y-12,4,0,7); ctx.fill();
  } else if (o.t === 'elix'){
    ctx.fillStyle = '#6b4a8c'; rr(x-18,y+2,36,22,5); ctx.fill();
    ctx.fillStyle = '#e04ad8'; ctx.beginPath(); ctx.moveTo(x,y-34); ctx.quadraticCurveTo(x+20,y-6,x,y+4); ctx.quadraticCurveTo(x-20,y-6,x,y-34); ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,.5)'; ctx.beginPath(); ctx.arc(x-5,y-10,4,0,7); ctx.fill();
  } else if (o.t === 'cannon'){
    ctx.fillStyle = '#9a7b55'; rr(x-20,y-10,40,34,4); ctx.fill();
    ctx.fillStyle = '#333'; ctx.beginPath(); ctx.arc(x,y+4,12,0,7); ctx.fill();
    ctx.save(); ctx.translate(x,y+4); ctx.rotate(Math.sin(warp.t*0.05+x)*0.8 - Math.PI/2); ctx.fillRect(0,-5,24,10); ctx.restore();
  } else if (o.t === 'hut'){
    ctx.fillStyle = '#b07a43'; rr(x-16,y-4,32,26,3); ctx.fill();
    ctx.fillStyle = '#e6c27a'; ctx.beginPath(); ctx.moveTo(x-22,y-2); ctx.lineTo(x,y-24); ctx.lineTo(x+22,y-2); ctx.closePath(); ctx.fill();
  } else {
    for (const [dx,dy,r] of [[0,0,20],[-12,8,14],[12,8,14]]){ ctx.fillStyle = '#2e8b3a'; ctx.beginPath(); ctx.arc(x+dx,y+dy,r,0,7); ctx.fill(); }
    ctx.fillStyle = '#4cb85a'; ctx.beginPath(); ctx.arc(x-5,y-5,8,0,7); ctx.fill();
  }
}
function drawCoC(lt){
  const off = lt * 7;
  ctx.fillStyle = '#7ec850'; ctx.fillRect(0,0,W,H);
  for (let y = posmod(off, 60) - 60; y < H; y += 60) for (let x = 0; x < W; x += 60){
    if (((x/60) + Math.floor((y - off)/60)) % 2 === 0){ ctx.fillStyle = '#76bf48'; ctx.fillRect(x,y,60,60); } }
  for (const o of warp.coc){ const y = o.y + off; if (y > -80 && y < H + 80) cocBuilding(o, y); }
  for (const u of warp.troops){
    const y = u.y + off + Math.sin(warp.t*0.2+u.ph)*2, x = u.x + Math.sin(warp.t*0.03+u.ph)*20;
    if (y < -20 || y > H+20) continue;
    ctx.fillStyle = u.c === 'barb' ? '#f2c9a0' : '#f2c9a0'; ctx.beginPath(); ctx.arc(x,y,6,0,7); ctx.fill();
    ctx.fillStyle = u.c === 'barb' ? '#ffd23f' : '#e0398b'; ctx.beginPath(); ctx.arc(x,y-3,6,Math.PI,0); ctx.fill();
    ctx.fillStyle = u.c === 'barb' ? '#8b5a2b' : '#3fae3f'; ctx.fillRect(x-4,y+5,8,7);
  }
}
// ---- Clash Royale ----
function crTower(x, y, col, king){
  const s = king ? 1.3 : 1;
  ctx.fillStyle = 'rgba(0,0,0,.2)'; ctx.beginPath(); ctx.ellipse(x+4, y+30*s, 34*s, 12*s, 0, 0, 7); ctx.fill();
  ctx.fillStyle = '#b8b8c0'; rr(x-28*s, y-20*s, 56*s, 50*s, 6); ctx.fill();
  ctx.strokeStyle = '#7d7d88'; ctx.lineWidth = 2; for (let i=0;i<3;i++){ ctx.beginPath(); ctx.moveTo(x-28*s, y-4*s+i*12*s); ctx.lineTo(x+28*s, y-4*s+i*12*s); ctx.stroke(); }
  for (let i=-2;i<=2;i++){ ctx.fillStyle = '#9a9aa4'; ctx.fillRect(x+i*11*s-4*s, y-28*s, 8*s, 9*s); }
  ctx.fillStyle = col; ctx.beginPath(); ctx.arc(x, y-28*s, 14*s, Math.PI, 0); ctx.fill();
  if (king){ ctx.fillStyle = '#ffd23f'; ctx.beginPath(); ctx.moveTo(x-12,y-48); ctx.lineTo(x-8,y-62); ctx.lineTo(x-2,y-52); ctx.lineTo(x+2,y-66); ctx.lineTo(x+6,y-52); ctx.lineTo(x+10,y-62); ctx.lineTo(x+12,y-48); ctx.closePath(); ctx.fill(); }
  else { ctx.fillStyle = '#ffe0b0'; ctx.beginPath(); ctx.arc(x, y-40, 7, 0, 7); ctx.fill(); ctx.fillStyle = col; ctx.beginPath(); ctx.arc(x, y-43, 7, Math.PI, 0); ctx.fill(); }
  ctx.fillStyle = '#222'; rr(x-24, y+34*s, 48, 7, 3); ctx.fill(); ctx.fillStyle = col === '#ff4040' ? '#ff6060' : '#4aa0ff'; rr(x-23, y+35*s, 46*(0.6+0.4*Math.sin(x+y)**2), 5, 2); ctx.fill();
}
function drawCR(lt){
  const AH = 1150, off = -clamp(lt / 100, 0, 1) * (AH - H);
  ctx.fillStyle = '#5a4630'; ctx.fillRect(0,0,W,H);
  ctx.save(); ctx.translate(0, off);
  ctx.fillStyle = '#9a8a70'; ctx.fillRect(0,0,W,AH);
  for (let y = 30; y < AH - 30; y += 40) for (let x = 30; x < W - 30; x += 40){ ctx.fillStyle = ((x+y)/40)%2 ? '#6fbf4a' : '#66b544'; ctx.fillRect(x,y,40,40); }
  // Wege
  ctx.fillStyle = '#c9b48a'; for (const lx of [110, 370]) ctx.fillRect(lx-18, 150, 36, AH-300);
  ctx.fillRect(110, AH/2-140, 260, 30); ctx.fillRect(110, AH/2+110, 260, 30);
  // Fluss & Brücken
  const ry = AH/2 - 30;
  ctx.fillStyle = '#3aa7e8'; ctx.fillRect(0, ry, W, 60);
  ctx.strokeStyle = 'rgba(255,255,255,.4)'; ctx.lineWidth = 2;
  for (let x = posmod(warp.t*2, 40) - 40; x < W; x += 40){ ctx.beginPath(); ctx.moveTo(x, ry+20); ctx.lineTo(x+16, ry+20); ctx.moveTo(x+20, ry+40); ctx.lineTo(x+36, ry+40); ctx.stroke(); }
  for (const bx of [110, 370]){ ctx.fillStyle = '#8b5a2b'; ctx.fillRect(bx-26, ry-8, 52, 76); ctx.strokeStyle = '#5c3a1a'; for (let i=0;i<7;i++){ ctx.beginPath(); ctx.moveTo(bx-26, ry-8+i*11); ctx.lineTo(bx+26, ry-8+i*11); ctx.stroke(); } }
  // Einheiten
  for (const u of warp.units){
    const x = u.lane ? 370 : 110, y = posmod(u.y + warp.t*1.5*u.dir, AH-300) + 150;
    ctx.fillStyle = u.c; ctx.beginPath(); ctx.arc(x + Math.sin(warp.t*.2+u.y)*4, y, 9, 0, 7); ctx.fill();
    ctx.fillStyle = '#f2c9a0'; ctx.beginPath(); ctx.arc(x + Math.sin(warp.t*.2+u.y)*4, y-4, 5, 0, 7); ctx.fill();
  }
  crTower(240, 110, '#ff4040', true); crTower(110, 250, '#ff4040'); crTower(370, 250, '#ff4040');
  crTower(110, AH-260, '#3a7bff'); crTower(370, AH-260, '#3a7bff'); crTower(240, AH-120, '#3a7bff', true);
  // Pfeile zwischen Türmen
  if (warp.t % 20 < 10){ ctx.strokeStyle = '#ffe94a'; ctx.lineWidth = 3; const k = (warp.t % 20)/10; ctx.beginPath(); ctx.moveTo(110, 250 + k*120); ctx.lineTo(110, 262 + k*120); ctx.stroke(); }
  ctx.restore();
}
// ---- Minecraft ----
function mcBlock(x, y, top, side, spk){
  ctx.fillStyle = side; ctx.fillRect(x, y, 32, 32);
  ctx.fillStyle = spk; for (let i=0;i<5;i++){ const sx = ((x*7 + y*13 + i*29) % 28 + 28) % 28, sy = ((x*3 + y*5 + i*17) % 28 + 28) % 28; ctx.fillRect(x+sx, y+sy, 4, 4); }
  if (top){ ctx.fillStyle = top; ctx.fillRect(x, y, 32, 8); ctx.fillStyle = '#4e8a2c'; for (let i=0;i<4;i++) ctx.fillRect(x + i*8 + 2, y+8, 4, 4); }
  ctx.strokeStyle = 'rgba(0,0,0,.15)'; ctx.lineWidth = 1; ctx.strokeRect(x+.5, y+.5, 31, 31);
}
function drawMC(lt){
  const off = lt * 9;
  const sky = ctx.createLinearGradient(0,0,0,H); sky.addColorStop(0,'#78a7ff'); sky.addColorStop(1,'#c3dbff');
  ctx.fillStyle = sky; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = '#fffbd0'; ctx.fillRect(360, 70, 56, 56); ctx.fillStyle = '#fff2a0'; ctx.fillRect(368, 78, 40, 40);
  ctx.fillStyle = 'rgba(255,255,255,.9)';
  for (let i=0;i<6;i++){ const x = posmod(i*170 - off*0.3, W+200) - 100; ctx.fillRect(x, 120 + (i%3)*50, 96, 24); ctx.fillRect(x+24, 108 + (i%3)*50, 48, 16); }
  const first = Math.floor(off / 32);
  for (let c = first; c < first + 17; c++){
    const i = posmod(c, 120), h = warp.mcH[i], x = c*32 - off;
    const topY = H - h*32 - 40;
    for (let r = 0; r < h + 2; r++){
      const y = topY + r*32;
      if (r === 0) mcBlock(x, y, '#5fa83a', '#866043', '#6b4a30');
      else if (r < 3) mcBlock(x, y, null, '#866043', '#6b4a30');
      else {
        const ore = warp.mcOres.find(o => o.i === i && o.d === r);
        mcBlock(x, y, null, '#7d7d7d', ore ? ore.c : '#666');
      }
    }
    if (warp.mcTrees.includes(i)){
      for (let k=1;k<=4;k++) mcBlock(x, topY - k*32, null, '#6b4f2a', '#4f3a1e');
      for (let dx=-1; dx<=1; dx++) for (let dy=5; dy<=6; dy++) mcBlock(x+dx*32, topY - dy*32, null, '#3f8f2a', '#2f6e1f');
      mcBlock(x, topY - 7*32, null, '#3f8f2a', '#2f6e1f');
    }
    // Creeper
    if (i === 40 || i === 95){
      const cy = topY;
      ctx.fillStyle = '#4fae3c'; ctx.fillRect(x+8, cy-60, 16, 28); ctx.fillRect(x+6, cy-84, 20, 22);
      ctx.fillRect(x+6, cy-34, 8, 34); ctx.fillRect(x+18, cy-34, 8, 34);
      ctx.fillStyle = '#111'; ctx.fillRect(x+9, cy-78, 5, 5); ctx.fillRect(x+18, cy-78, 5, 5); ctx.fillRect(x+13, cy-72, 6, 8);
    }
  }
}
function drawWarp(){
  const t = warp.t;
  let seg = SEG.find(s => t >= s.s - 22 && t < s.e + 22) || SEG[0];
  const lt = t - seg.s;
  if (seg.n === 'coc') drawCoC(lt); else if (seg.n === 'cr') drawCR(lt); else drawMC(lt);
  // Spieler
  const sx = P.x, sy = P.y, sf = P.face, sth = P.thr, svx = P.vx;
  P.x = W/2 - 18; P.y = H/2 - 30 + Math.sin(t*0.1)*8; P.face = 1; P.thr = false; P.vx = seg.n === 'mc' ? 6 : 0;
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); }
  ctx.globalAlpha = 1;
  drawPlayer();
  P.x = sx; P.y = sy; P.face = sf; P.thr = sth; P.vx = svx;
  // Geschwindigkeitslinien
  ctx.strokeStyle = 'rgba(255,255,255,.35)'; ctx.lineWidth = 2;
  for (let i=0;i<10;i++){ const x = (i*53 + t*3) % W, y = (i*97 + t*25) % H; ctx.beginPath(); if (seg.n === 'mc'){ ctx.moveTo(y % W, x); ctx.lineTo(y % W - 40, x); } else { ctx.moveTo(x, y); ctx.lineTo(x, y + 40); } ctx.stroke(); }
  // Tunnel-Übergänge
  let a = 0;
  if (t < seg.s) a = 1 - (t - (seg.s - 22)) / 22;
  if (t >= seg.e) a = (t - seg.e) / 22;
  if (t < 22) a = 1 - t/22;
  drawTunnel(clamp(a, 0, 1));
  // Titel
  const names = {coc:'🏰 Welt 1: Clash of Clans', cr:'👑 Welt 2: Clash Royale', mc:'⛏️ Welt 3: Minecraft'};
  if (lt > 0 && lt < 100){
    ctx.globalAlpha = clamp(Math.min(lt, 100-lt)/15, 0, 1);
    ctx.fillStyle = 'rgba(0,0,0,.45)'; rr(W/2-170, 26, 340, 46, 12); ctx.fill();
    ctx.fillStyle = '#fff'; ctx.font = '900 24px Segoe UI'; ctx.textAlign = 'center'; ctx.fillText(names[seg.n], W/2, 58);
    ctx.globalAlpha = 1;
  }
  // Fortschritt + Ziel
  ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W/2-100, H-46, 200, 8, 4); ctx.fill();
  ctx.fillStyle = {hc:'#ffa32b', ab:'#ff3b3b', gd:'#d07bff'}[warp.kind]; rr(W/2-100, H-46, 200*t/WARP_DUR, 8, 4); ctx.fill();
  ctx.fillStyle = 'rgba(255,255,255,.85)'; ctx.font = 'bold 13px Segoe UI'; ctx.textAlign = 'center';
  ctx.fillText('Ziel: ' + ({hc:'Hill Climb', ab:'Angry Birds', gd:'Geometry Dash'}[warp.kind]) + '   ·   ENTER = überspringen', W/2, H-58);
}


// ===================== Angry-Birds-Dimension =====================
const GR = 620, ANC = {x:95, y:GR-100}, PULL = 85, BR = 13;
const ptr = {x:0, y:0, down:false};
function cpos(cx, cy){ const r = cv.getBoundingClientRect(); ptr.x = (cx - r.left)/r.width*W; ptr.y = (cy - r.top)/r.height*H; }
cv.addEventListener('mousedown', e => { cpos(e.clientX, e.clientY); ptr.down = true; });
addEventListener('mousemove', e => cpos(e.clientX, e.clientY));
addEventListener('mouseup', () => ptr.down = false);
cv.addEventListener('touchstart', e => { if (e.touches[0]){ cpos(e.touches[0].clientX, e.touches[0].clientY); ptr.down = true; } }, {passive:false});
cv.addEventListener('touchmove', e => { if (e.touches[0]) cpos(e.touches[0].clientX, e.touches[0].clientY); }, {passive:false});
cv.addEventListener('touchend', e => { if (!e.touches.length) ptr.down = false; }, {passive:false});

const MAT = {wood:{d:1, hp:6, k:1}, glass:{d:.7, hp:2.6, k:1.8}, stone:{d:2.4, hp:14, k:.45}, pig:{d:.8, hp:2.2, k:1.4}};
function box(x, y, w, h, mat){ return {x, y, w, h, vx:0, vy:0, mat, hp:MAT[mat].hp, max:MAT[mat].hp, m:MAT[mat].d*w*h/900, dead:false}; }
function buildAB(){
  const B = [], pick = () => ['wood','wood','glass','stone'][rand(0,4)|0];
  const L = rand(0,3)|0;
  if (L === 0){
    const o = 250, m1 = pick(), m2 = pick();
    B.push(box(o, GR-80, 14, 80, m1), box(o+86, GR-80, 14, 80, m1), box(o-5, GR-94, 110, 14, 'wood'), box(o+35, GR-30, 30, 30, 'pig'));
    B.push(box(o+8, GR-154, 12, 60, m2), box(o+80, GR-154, 12, 60, m2), box(o, GR-168, 100, 14, 'wood'), box(o+35, GR-124, 30, 30, 'pig'));
    B.push(box(o+35, GR-198, 30, 30, 'pig'));
    B.push(box(o+140, GR-50, 50, 50, 'stone'), box(o+150, GR-80, 30, 30, 'pig'));
  } else if (L === 1){
    const o = 240;
    for (let i=0;i<4;i++) B.push(box(o+i*52, GR-46, 46, 46, i%3===0 ? 'stone' : 'wood'));
    for (let i=0;i<3;i++) B.push(box(o+26+i*52, GR-92, 46, 46, i===1 ? 'glass' : 'wood'));
    for (let i=0;i<2;i++) B.push(box(o+52+i*52, GR-138, 46, 46, 'glass'));
    B.push(box(o+90, GR-168, 30, 30, 'pig'), box(o+5, GR-76, 30, 30, 'pig'), box(o+175, GR-76, 30, 30, 'pig'));
  } else {
    const o = 230;
    B.push(box(o, GR-110, 16, 110, 'wood'), box(o+60, GR-110, 16, 110, 'wood'), box(o-6, GR-124, 88, 14, 'wood'), box(o+23, GR-30, 30, 30, 'pig'), box(o+23, GR-154, 30, 30, 'pig'));
    B.push(box(o+120, GR-70, 14, 70, 'glass'), box(o+190, GR-70, 14, 70, 'glass'), box(o+114, GR-84, 96, 14, 'stone'), box(o+147, GR-30, 30, 30, 'pig'));
    B.push(box(o+130, GR-130, 46, 46, 'wood'), box(o+138, GR-160, 30, 30, 'pig'));
  }
  return B;
}
function enterAB(){
  mode = 'ab'; flash = 1; parts = []; missiles = [];
  ab = {blocks: buildAB(), birds: 3, bird: null, phase: 'aim', t: 0, ang: -0.75, pow: 0.75, drag: false, spPrev: true, wait: 0, msg: '', msgT: 0, ptrPrev: true};
  // Gebäude einmal setzen lassen
  for (let i=0;i<60;i++) abPhysics(true);
  beep(500,.15,'triangle',.08,-200); setTimeout(()=>beep(700,.15,'triangle',.08,-200),120);
}
function abDamage(b, amt){
  if (b.dead) return;
  b.hp -= amt * MAT[b.mat].k;
  if (b.hp <= 0){
    b.dead = true;
    const cols = {wood:['#a8742f','#d9a35b'], glass:['#bfefff','#fff'], stone:['#888','#bbb'], pig:['#7ed957','#4a9e2f','#fff']}[b.mat];
    burst(b.x+b.w/2, b.y+b.h/2, b.mat === 'pig' ? 22 : 12, cols, 5);
    if (b.mat === 'pig'){ coinsRun += 5; beep(220,.3,'square',.09,-150); ab.msg = '+5'; ab.msgT = 40; ab.mx = b.x+b.w/2; ab.my = b.y; }
    else beep(b.mat==='glass'?1500:b.mat==='stone'?160:300,.12,'square',.05,-100);
  } else if (amt > 1.2 && b.mat === 'pig') beep(500,.08,'sine',.05,200);
}
function abPhysics(silent){
  const B = ab.blocks.filter(b => !b.dead);
  for (const b of B){
    b.vy += 0.35; b.x += b.vx; b.y += b.vy;
    if (b.y + b.h > GR){ if (!silent && b.vy > 3.5) abDamage(b, (b.vy-3.5)*0.6); b.y = GR - b.h; b.vy = -b.vy*0.08; b.vx *= 0.8; }
    if (b.x < 0){ b.x = 0; b.vx *= -0.3; } if (b.x + b.w > W){ b.x = W - b.w; b.vx *= -0.3; }
  }
  for (let it=0; it<5; it++){
    for (let i=0;i<B.length;i++) for (let j=i+1;j<B.length;j++){
      const a = B[i], b = B[j];
      const ox = Math.min(a.x+a.w, b.x+b.w) - Math.max(a.x, b.x), oy = Math.min(a.y+a.h, b.y+b.h) - Math.max(a.y, b.y);
      if (ox <= 0 || oy <= 0) continue;
      const tm = a.m + b.m;
      if (ox < oy){
        const s = (a.x + a.w/2 < b.x + b.w/2) ? 1 : -1;
        a.x -= s*ox*b.m/tm; b.x += s*ox*a.m/tm;
        const imp = (a.vx - b.vx)*s;
        if (imp > 0){ const J = 1.1*imp/(1/a.m+1/b.m); a.vx -= s*J/a.m; b.vx += s*J/b.m; if (!silent && it===0 && imp > 2.2){ abDamage(a, imp-2.2); abDamage(b, imp-2.2); } }
      } else {
        const s = (a.y + a.h/2 < b.y + b.h/2) ? 1 : -1;   // s=1: a liegt oben
        const top = s===1 ? a : b, bot = s===1 ? b : a;
        const frac = bot.y + bot.h >= GR - 0.5 ? 1 : top.m/tm;
        top.y -= oy*(bot.y + bot.h >= GR - 0.5 ? 1 : bot.m/tm); if (frac !== 1) bot.y += oy*top.m/tm;
        const imp = top.vy - bot.vy;
        if (imp > 0 && (frac === 1 || imp < 1.2)){ top.vy = bot.vy; if (!silent && it===0 && imp > 2.5){ abDamage(top, imp-2.5); abDamage(bot, (imp-2.5)*0.7); } }
        else if (imp > 0){ const J = 1.08*imp/(1/a.m+1/b.m); top.vy -= J/top.m; bot.vy += J/bot.m; if (!silent && it===0 && imp > 2.5){ abDamage(a, imp-2.5); abDamage(b, imp-2.5); } }
        top.vx += (bot.vx - top.vx)*0.15;
      }
    }
  }
  for (const b of B){ if (Math.abs(b.vx) < 0.04) b.vx = 0; b.vx *= 0.995; }
}
function abBirdStep(){
  const bd = ab.bird;
  for (let k=0;k<3;k++){
    bd.vy += 0.3/3; bd.x += bd.vx/3; bd.y += bd.vy/3;
    if (bd.y + BR > GR){ bd.y = GR - BR; bd.vy = -bd.vy*0.35; bd.vx *= 0.85; }
    for (const b of ab.blocks){
      if (b.dead) continue;
      const cx = clamp(bd.x, b.x, b.x+b.w), cy = clamp(bd.y, b.y, b.y+b.h);
      let dx = bd.x - cx, dy = bd.y - cy, d = Math.hypot(dx, dy);
      if (d >= BR) continue;
      if (d < 0.01){ dx = -1; dy = 0; d = 1; }
      const nx = dx/d, ny = dy/d;
      bd.x += nx*(BR-d); bd.y += ny*(BR-d);
      const imp = -((bd.vx - b.vx)*nx + (bd.vy - b.vy)*ny);
      if (imp > 0){
        const mB = 1.6, J = 1.25*imp/(1/mB + 1/b.m);
        bd.vx += nx*J/mB; bd.vy += ny*J/mB; b.vx -= nx*J/b.m; b.vy -= ny*J/b.m;
        if (imp > 1.5){ abDamage(b, imp*0.9); bd.hits++; }
        if (b.dead){ bd.vx -= nx*J/mB*0.6; bd.vy -= ny*J/mB*0.6; }
      }
    }
  }
  if (time % 2 === 0 && !bd.rest) parts.push({x:bd.x, y:bd.y, vx:0, vy:0, life:60, max:60, c:'rgba(255,255,255,.7)', s:2.5, g:0});
}
function abAimPos(){
  if (ab.drag){
    let dx = ptr.x - ANC.x, dy = ptr.y - ANC.y, d = Math.hypot(dx, dy);
    if (d > PULL){ dx *= PULL/d; dy *= PULL/d; }
    return {x: ANC.x + dx, y: ANC.y + dy};
  }
  return {x: ANC.x - Math.cos(ab.ang)*ab.pow*PULL, y: ANC.y - Math.sin(ab.ang)*ab.pow*PULL};
}
const abLaunchV = p => ({vx: (ANC.x - p.x)*0.155, vy: (ANC.y - p.y)*0.155});
function abLaunch(p){
  const v = abLaunchV(p);
  ab.bird = {x:p.x, y:p.y, vx:v.vx, vy:v.vy, t:0, rest:0, hits:0};
  ab.birds--; ab.phase = 'fly'; parts = parts.filter(q => q.c.indexOf('255,255,255') < 0);
  beep(300,.35,'sawtooth',.07,500);
}
function updateAB(){
  ab.t++;
  if (ab.msgT) ab.msgT--;
  for (const p of parts){ p.x += p.vx; p.y += p.vy; p.vy += p.g; p.vx *= 0.97; p.life--; }
  parts = parts.filter(p => p.life > 0);
  abPhysics(false);
  const pigs = ab.blocks.filter(b => b.mat === 'pig' && !b.dead).length;
  if (ab.phase === 'aim'){
    const sp = keys.Space || keys.Enter;
    if (keys.ArrowUp || keys.KeyW) ab.ang = Math.max(-1.45, ab.ang - 0.025);
    if (keys.ArrowDown || keys.KeyS) ab.ang = Math.min(0.5, ab.ang + 0.025);
    if (keys.ArrowRight || keys.KeyD) ab.pow = Math.min(1, ab.pow + 0.012);
    if (keys.ArrowLeft || keys.KeyA) ab.pow = Math.max(0.25, ab.pow - 0.012);
    if (ptr.down && !ab.ptrPrev && ab.t > 20) ab.drag = true;
    if (ab.drag && !ptr.down){
      ab.drag = false; const p = abAimPos();
      if (Math.hypot(p.x-ANC.x, p.y-ANC.y) > 18) abLaunch(p);
    } else if (sp && !ab.spPrev && !ab.drag && ab.t > 20) abLaunch(abAimPos());
    ab.spPrev = sp;
  } else if (ab.phase === 'fly'){
    ab.bird.t++; abBirdStep();
    const bd = ab.bird, slow = Math.hypot(bd.vx, bd.vy) < 0.6;
    bd.rest = slow ? bd.rest + 1 : 0;
    if (bd.rest > 40 || bd.x > W + 40 || bd.x < -40 || bd.t > 480){ ab.phase = 'wait'; ab.wait = 80; }
  } else if (ab.phase === 'wait'){
    if (ab.bird && ab.bird.x > -40 && ab.bird.x < W+40) abBirdStep();
    if (--ab.wait <= 0){
      ab.bird = null;
      if (pigs === 0) { ab.phase = 'win'; ab.wait = 100; coinsRun += ab.birds*10; ab.bonusBirds = ab.birds; beep(660,.2,'triangle',.08,400); setTimeout(()=>beep(990,.3,'triangle',.08,400),150); }
      else if (ab.birds > 0){ ab.phase = 'aim'; ab.drag = false; ab.spPrev = true; ab.ptrPrev = true; ab.t = 0; }
      else { ab.phase = 'fail'; ab.wait = 110; beep(200,.5,'sawtooth',.08,-120); }
    }
  } else if (ab.phase === 'win' || ab.phase === 'fail'){
    if (--ab.wait <= 0){ const f = ab.phase === 'fail'; ab = null; exitDim(f); return; }
  }
  if (pigs === 0 && ab.phase === 'fly' && ab.bird.t > 30 && ab.bird.rest > 5){ ab.phase = 'wait'; ab.wait = 40; }
  ab.ptrPrev = ptr.down;
}
function drawBird(x, y, r, rot){
  ctx.save(); ctx.translate(x, y); ctx.rotate(rot || 0); const s = r/13; ctx.scale(s, s);
  ctx.fillStyle = '#d62828'; ctx.beginPath(); ctx.arc(0,0,13,0,7); ctx.fill();
  ctx.fillStyle = '#b01e1e'; ctx.beginPath(); ctx.moveTo(-6,-11); ctx.lineTo(-2,-19); ctx.lineTo(1,-12); ctx.lineTo(5,-18); ctx.lineTo(5,-10); ctx.fill();
  ctx.fillStyle = '#f2e3c6'; ctx.beginPath(); ctx.ellipse(1,7,8,5,0,0,7); ctx.fill();
  ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(3,-2,4,0,7); ctx.arc(9,-2,3.5,0,7); ctx.fill();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(4.5,-1.5,1.7,0,7); ctx.arc(9.5,-1.5,1.6,0,7); ctx.fill();
  ctx.fillStyle = '#222'; ctx.save(); ctx.translate(6,-7); ctx.rotate(0.15); ctx.fillRect(-7,-1.5,14,3.5); ctx.restore();
  ctx.fillStyle = '#ffb703'; ctx.beginPath(); ctx.moveTo(10,2); ctx.lineTo(19,4); ctx.lineTo(10,7); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#222'; ctx.beginPath(); ctx.moveTo(-13,0); ctx.lineTo(-19,-3); ctx.lineTo(-18,3); ctx.closePath(); ctx.fill();
  ctx.restore();
}
function drawBlock(b){
  const x = b.x, y = b.y, w = b.w, h = b.h, dmg = 1 - b.hp/b.max;
  if (b.mat === 'pig'){
    const cx = x+w/2, cy = y+h/2, r = w/2;
    ctx.fillStyle = '#7ed957'; ctx.beginPath(); ctx.arc(cx,cy,r,0,7); ctx.fill();
    ctx.strokeStyle = '#3f8a24'; ctx.lineWidth = 2; ctx.stroke();
    ctx.fillStyle = '#7ed957'; ctx.beginPath(); ctx.arc(cx-9,cy-r+2,4,0,7); ctx.arc(cx+9,cy-r+2,4,0,7); ctx.fill();
    ctx.fillStyle = '#a6eb7c'; ctx.beginPath(); ctx.ellipse(cx,cy+3,7,5.5,0,0,7); ctx.fill();
    ctx.fillStyle = '#2f6b1a'; ctx.beginPath(); ctx.arc(cx-2.5,cy+3,1.5,0,7); ctx.arc(cx+2.5,cy+3,1.5,0,7); ctx.fill();
    ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(cx-7,cy-5,3.5,0,7); ctx.arc(cx+7,cy-5,3.5,0,7); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(cx-6.5,cy-5,1.5,0,7); ctx.arc(cx+7.5,cy-5,1.5,0,7); ctx.fill();
    if (dmg > 0.3){ ctx.strokeStyle = '#3a2a6a'; ctx.lineWidth = 2; ctx.beginPath(); ctx.arc(cx+7,cy-5,5,0,7); ctx.stroke(); }
    return;
  }
  if (b.mat === 'wood'){
    ctx.fillStyle = '#c8913f'; ctx.fillRect(x,y,w,h);
    ctx.strokeStyle = '#9c6a26'; ctx.lineWidth = 1;
    if (w > h) for (let i=1;i<3;i++){ ctx.beginPath(); ctx.moveTo(x+2, y+h*i/3); ctx.lineTo(x+w-2, y+h*i/3); ctx.stroke(); }
    else for (let i=1;i<3;i++){ ctx.beginPath(); ctx.moveTo(x+w*i/3, y+2); ctx.lineTo(x+w*i/3, y+h-2); ctx.stroke(); }
    ctx.strokeStyle = '#7a4f1a'; ctx.lineWidth = 2; ctx.strokeRect(x+1,y+1,w-2,h-2);
  } else if (b.mat === 'glass'){
    ctx.fillStyle = 'rgba(170,225,255,.55)'; ctx.fillRect(x,y,w,h);
    ctx.strokeStyle = 'rgba(255,255,255,.9)'; ctx.lineWidth = 2; ctx.strokeRect(x+1,y+1,w-2,h-2);
    ctx.beginPath(); ctx.moveTo(x+4,y+h*0.4); ctx.lineTo(x+w*0.4,y+4); ctx.stroke();
  } else {
    ctx.fillStyle = '#9a9aa2'; ctx.fillRect(x,y,w,h);
    ctx.fillStyle = '#85858e'; ctx.fillRect(x+3,y+3,w*0.4,h*0.35); ctx.fillRect(x+w*0.5,y+h*0.55,w*0.4,h*0.35);
    ctx.strokeStyle = '#5d5d66'; ctx.lineWidth = 2; ctx.strokeRect(x+1,y+1,w-2,h-2);
  }
  if (dmg > 0.35){ ctx.strokeStyle = 'rgba(30,20,10,.7)'; ctx.lineWidth = 1.5; ctx.beginPath();
    ctx.moveTo(x+w*0.2,y+h*0.1); ctx.lineTo(x+w*0.5,y+h*0.45); ctx.lineTo(x+w*0.35,y+h*0.7);
    if (dmg > 0.65){ ctx.moveTo(x+w*0.5,y+h*0.45); ctx.lineTo(x+w*0.85,y+h*0.6); } ctx.stroke(); }
}
function drawAB(){
  const sk = ctx.createLinearGradient(0,0,0,GR); sk.addColorStop(0,'#7cc8f0'); sk.addColorStop(1,'#d8f1ff');
  ctx.fillStyle = sk; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = 'rgba(255,255,255,.85)';
  for (const [x,y,s] of [[80,140,1],[300,90,1.3],[420,200,.8]]){ ctx.beginPath(); ctx.arc(x,y,20*s,0,7); ctx.arc(x+24*s,y-8*s,26*s,0,7); ctx.arc(x+50*s,y,20*s,0,7); ctx.fill(); }
  ctx.fillStyle = '#9fd36b'; ctx.beginPath(); ctx.moveTo(0,GR);
  for (let x=0;x<=W;x+=10) ctx.lineTo(x, GR-60-Math.sin(x*0.012)*35-Math.sin(x*0.031)*12); ctx.lineTo(W,GR); ctx.fill();
  ctx.fillStyle = '#5a9e2f'; ctx.fillRect(0,GR,W,16);
  const gg = ctx.createLinearGradient(0,GR+16,0,H); gg.addColorStop(0,'#8b5a2b'); gg.addColorStop(1,'#4a2f15');
  ctx.fillStyle = gg; ctx.fillRect(0,GR+16,W,H-GR-16);
  // Schleuder hinten
  ctx.fillStyle = '#6b3f1a'; ctx.fillRect(ANC.x-5, ANC.y+10, 10, GR-ANC.y-10);
  ctx.fillRect(ANC.x-16, ANC.y-14, 8, 30); ctx.fillRect(ANC.x+8, ANC.y-14, 8, 30);
  const aiming = ab.phase === 'aim', ap = aiming ? abAimPos() : ANC;
  if (aiming){ ctx.strokeStyle = '#3b1f0a'; ctx.lineWidth = 5; ctx.beginPath(); ctx.moveTo(ANC.x+12, ANC.y-10); ctx.lineTo(ap.x, ap.y); ctx.stroke(); }
  // Flugbahn-Vorschau
  if (aiming){
    const v = abLaunchV(ap); let x = ap.x, y = ap.y, vx = v.vx, vy = v.vy;
    ctx.fillStyle = 'rgba(255,255,255,.85)';
    for (let i=0;i<70;i++){ vy += 0.3; x += vx; y += vy; if (y > GR) break; if (i % 3 === 0){ ctx.beginPath(); ctx.arc(x, y, 3 - i*0.03, 0, 7); ctx.fill(); } }
  }
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); }
  ctx.globalAlpha = 1;
  for (const b of ab.blocks) if (!b.dead) drawBlock(b);
  if (aiming) drawBird(ap.x, ap.y, BR, 0);
  if (ab.bird) drawBird(ab.bird.x, ab.bird.y, BR, Math.atan2(ab.bird.vy, ab.bird.vx)*0.6 + (ab.bird.rest ? 0 : 0));
  if (aiming){ ctx.strokeStyle = '#3b1f0a'; ctx.lineWidth = 5; ctx.beginPath(); ctx.moveTo(ANC.x-12, ANC.y-10); ctx.lineTo(ap.x, ap.y); ctx.stroke(); }
  // Vögel in der Warteschlange
  for (let i=0;i<ab.birds - (aiming ? 1 : 0);i++) drawBird(50 - i*26, GR-11, 11, 0);
  // HUD
  const pigs = ab.blocks.filter(b => b.mat === 'pig' && !b.dead).length;
  ctx.fillStyle = 'rgba(0,0,0,.35)'; rr(W/2-150, 14, 300, 52, 12); ctx.fill();
  ctx.fillStyle = '#fff'; ctx.font = 'bold 16px Segoe UI'; ctx.textAlign = 'center'; ctx.fillText('DIMENSION: ANGRY BIRDS', W/2, 36);
  ctx.font = 'bold 14px Segoe UI'; ctx.fillText('🐷 ' + pigs + ' übrig   ·   🐦 ' + ab.birds + ' Schuss   ·   ● ' + coinsRun, W/2, 57);
  if (ab.msgT){ ctx.globalAlpha = ab.msgT/40; ctx.fillStyle = '#ffd23f'; ctx.font = '900 22px Segoe UI'; ctx.fillText(ab.msg, ab.mx, ab.my - (40-ab.msgT)); ctx.globalAlpha = 1; }
  ctx.fillStyle = 'rgba(255,255,255,.85)'; ctx.font = '14px Segoe UI';
  if (!reels){
  ctx.fillText('Ziehen & loslassen zum Schießen', W/2, GR+60);
  ctx.fillText('Tastatur: ↑↓ Winkel · ←→ Stärke · Leertaste schießen', W/2, GR+84);
  ctx.fillText('Alle Schweine besiegen = Bonus!', W/2, GR+108);
  } else ctx.fillText('Ziehen & loslassen · ↑↓ ←→ + Leertaste', W/2, GR+40);
  if (ab.phase === 'win' || ab.phase === 'fail'){
    ctx.fillStyle = 'rgba(0,0,0,.4)'; ctx.fillRect(0, 230, W, 110);
    ctx.font = '900 38px Segoe UI'; ctx.fillStyle = ab.phase === 'win' ? '#ffd23f' : '#ff6b6b';
    ctx.fillText(ab.phase === 'win' ? 'LEVEL GESCHAFFT!' : 'DANEBEN!', W/2, 285);
    ctx.font = 'bold 17px Segoe UI'; ctx.fillStyle = '#fff';
    ctx.fillText(ab.phase === 'win' ? (ab.bonusBirds ? '+' + ab.bonusBirds*10 + ' Münzen für übrige Vögel' : 'Alle Schweine erledigt') : 'Zurück zum Jetpack – ohne Bonus', W/2, 318);
  }
}



// ===================== Mini-Spiele für Reels =====================
let mg = null;
const MINI = {};
const tapIn = () => keys.Space || keys.ArrowUp || keys.KeyW || mouseJet || touch.any || ptr.down;
function mgHud(title, line){
  ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W/2-150, 14, 300, 50, 12); ctx.fill();
  ctx.fillStyle = '#fff'; ctx.textAlign = 'center'; ctx.font = 'bold 16px Segoe UI'; ctx.fillText(title, W/2, 35);
  ctx.font = 'bold 14px Segoe UI'; ctx.fillText(line + '   ·   ● ' + coinsRun, W/2, 55);
}
function mgParts(){ for (const p of parts){ p.x+=p.vx; p.y+=p.vy; p.vy+=p.g; p.vx*=0.97; p.life--; } parts = parts.filter(p=>p.life>0); }
function mgDrawParts(){ for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); } ctx.globalAlpha = 1; }

// ---------- Flappy Bird ----------
MINI.fb = {
  e(){ mode = 'fb'; parts = []; mg = {y:H/2-60, vy:0, pipes:[], passed:0, goal:12, t:0, prev:true, dist:0, started:false}; },
  u(){
    const g = mg; g.t++; mgParts();
    const tap = tapIn();
    if (tap && !g.prev && g.t > 15){ g.vy = -8.2; g.started = true; beep(600,.06,'square',.04,200); }
    g.prev = tap;
    if (!g.started){ g.y = H/2-60 + Math.sin(g.t*0.1)*8; return; }
    g.vy = Math.min(g.vy + 0.45, 11); g.y += g.vy; g.dist += 3.2;
    if (g.pipes.length === 0 || g.pipes[g.pipes.length-1].x < W - 230){
      if (g.passed + g.pipes.length < g.goal) g.pipes.push({x: W + 40, gy: rand(190, H-300), gap: Math.max(150, 185 - (g.passed + g.pipes.length)*3), done:false});
    }
    if (P.inv > 0) P.inv--;
    for (const p of g.pipes){
      p.x -= 3.2;
      if (!p.done && p.x + 60 < 110){ p.done = true; g.passed++; coinsRun++; beep(900,.07,'square',.05,300); }
      if (P.inv <= 0 && 110+15 > p.x && 110-15 < p.x+60 && (g.y-12 < p.gy || g.y+12 > p.gy + p.gap)){ hurt(); if (state!=='play'||mode!=='fb') return; }
    }
    g.pipes = g.pipes.filter(p => p.x > -80);
    if (g.y > H-122 || g.y < -40){ P.shield = 0; P.inv = 0; hurt(); return; }
    if (g.passed >= g.goal) exitDim(false);
  },
  d(){
    const g = mg;
    const sk = ctx.createLinearGradient(0,0,0,H); sk.addColorStop(0,'#4ec0ca'); sk.addColorStop(1,'#bdeef2'); ctx.fillStyle = sk; ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#d7f5d1'; for (let i=0;i<8;i++){ const x = posmod(i*90 - g.dist*0.3, W+100)-50; ctx.beginPath(); ctx.arc(x, H-140, 40, Math.PI, 0); ctx.fill(); }
    ctx.fillStyle = '#9ad48a'; for (let i=0;i<6;i++){ const x = posmod(i*110 - g.dist*0.5, W+120)-60; ctx.fillRect(x, H-180, 40, 70); ctx.fillRect(x+48, H-160, 30, 50); }
    for (const p of g.pipes){
      for (const [y0, h, cap] of [[0, p.gy, p.gy-26], [p.gy+p.gap, H-110-(p.gy+p.gap), p.gy+p.gap]]){
        const pg = ctx.createLinearGradient(p.x,0,p.x+60,0); pg.addColorStop(0,'#5ea83a'); pg.addColorStop(.4,'#9be15d'); pg.addColorStop(1,'#3f7a22');
        ctx.fillStyle = pg; ctx.fillRect(p.x, y0, 60, h); ctx.fillRect(p.x-5, cap, 70, 26);
        ctx.strokeStyle = '#2b4f16'; ctx.lineWidth = 2; ctx.strokeRect(p.x, y0, 60, h); ctx.strokeRect(p.x-5, cap, 70, 26);
      }
    }
    ctx.fillStyle = '#ded895'; ctx.fillRect(0, H-110, W, 110);
    ctx.fillStyle = '#73bf2e'; ctx.fillRect(0, H-110, W, 16);
    ctx.fillStyle = '#5ea323'; for (let x = posmod(-g.dist, 24)-24; x < W; x += 24){ ctx.beginPath(); ctx.moveTo(x, H-94); ctx.lineTo(x+12, H-110); ctx.lineTo(x+24, H-94); ctx.fill(); }
    mgDrawParts();
    if (!(P.inv > 0 && Math.floor(P.inv/5)%2)){
      ctx.save(); ctx.translate(110, g.y); ctx.rotate(clamp(g.vy*0.07, -0.5, 1.2));
      ctx.fillStyle = '#f8d030'; ctx.beginPath(); ctx.ellipse(0,0,17,13,0,0,7); ctx.fill(); ctx.strokeStyle = '#000'; ctx.lineWidth = 2; ctx.stroke();
      ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(7,-5,6,0,7); ctx.fill(); ctx.stroke();
      ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(9,-5,2.2,0,7); ctx.fill();
      ctx.fillStyle = '#f5f0e0'; ctx.beginPath(); ctx.ellipse(-7, 2 + (Math.floor(g.t/5)%2 ? -4 : 2), 9, 5, 0, 0, 7); ctx.fill(); ctx.stroke();
      ctx.fillStyle = '#f05a28'; ctx.beginPath(); ctx.ellipse(13,4,9,4,0,0,7); ctx.fill(); ctx.stroke();
      ctx.restore();
      if (P.shield){ ctx.globalAlpha = .3; ctx.fillStyle = '#6cf'; ctx.beginPath(); ctx.arc(110, g.y, 30, 0, 7); ctx.fill(); ctx.globalAlpha = 1; }
    }
    ctx.fillStyle = '#fff'; ctx.strokeStyle = '#000'; ctx.lineWidth = 5; ctx.font = '900 56px Segoe UI'; ctx.textAlign = 'center';
    ctx.strokeText(g.passed, W/2, 140); ctx.fillText(g.passed, W/2, 140);
    mgHud('FLAPPY BIRD', g.passed + ' / ' + g.goal + ' Röhren');
    if (!g.started){ ctx.fillStyle = '#fff'; ctx.strokeStyle='#000'; ctx.lineWidth=3; ctx.font = 'bold 22px Segoe UI'; ctx.strokeText('Tippen / Leertaste zum Flattern', W/2, H/2+40); ctx.fillText('Tippen / Leertaste zum Flattern', W/2, H/2+40); }
  }
};

// ---------- Fruit Ninja ----------
const FRUITS = [ {c:'#e63946', in:'#ffd6d6', r:24, n:'Apfel'}, {c:'#2a9d3f', in:'#ff4d6d', r:34, n:'Melone'}, {c:'#ff9f1a', in:'#ffd08a', r:24, n:'Orange'},
                 {c:'#ffe14d', in:'#fff7c2', r:20, n:'Zitrone'}, {c:'#7b2cbf', in:'#d9b3ff', r:18, n:'Pflaume'}, {c:'#a3d940', in:'#e8ffc2', r:22, n:'Kiwi'} ];
MINI.fn = {
  e(){ mode = 'fn'; parts = []; mg = {items:[], halves:[], trail:[], cut:0, goal:20, miss:0, t:0, next:40, combo:0, comboT:0, lp:null, msgT:0}; },
  u(){
    const g = mg; g.t++; mgParts();
    if (--g.next <= 0){
      const n = rand(1, 3.6)|0;
      for (let i=0;i<n;i++){
        const bomb = g.t > 200 && Math.random() < 0.16, x = rand(70, W-70);
        g.items.push({x, y:H+40, vx:(W/2-x)*rand(0.006,0.014), vy:-rand(15,19), rot:0, vr:rand(-.1,.1), bomb, f: FRUITS[rand(0,FRUITS.length)|0], dead:false});
      }
      g.next = rand(45, 80); beep(250,.08,'sine',.03,150);
    }
    if (ptr.down) g.trail.push({x:ptr.x, y:ptr.y, l:12});
    for (const t of g.trail) t.l--; g.trail = g.trail.filter(t => t.l > 0);
    const cur = ptr.down ? {x:ptr.x, y:ptr.y} : null;
    if (cur && g.lp){
      const sp = Math.hypot(cur.x-g.lp.x, cur.y-g.lp.y);
      if (sp > 4) for (const it of g.items){
        if (it.dead) continue;
        const r = it.bomb ? 22 : it.f.r;
        if (segDist(it.x, it.y, g.lp.x, g.lp.y, cur.x, cur.y) < r){
          it.dead = true;
          if (it.bomb){ burst(it.x, it.y, 40, ['#f90','#ff0','#333','#fff'], 10); shake = 16; P.shield = 0; die(); return; }
          g.cut++; coinsRun++; g.combo++; g.comboT = 14;
          const ang = Math.atan2(cur.y-g.lp.y, cur.x-g.lp.x);
          for (const s of [-1,1]) g.halves.push({x:it.x, y:it.y, vx:it.vx + Math.cos(ang+Math.PI/2)*s*2.5, vy:it.vy*0.3 + Math.sin(ang+Math.PI/2)*s*2.5 - 1, rot:ang, vr:s*0.12, f:it.f, s});
          burst(it.x, it.y, 14, [it.f.in, it.f.c], 5);
          beep(1000 + g.combo*120, .08, 'sawtooth', .04, -500);
        }
      }
    }
    g.lp = cur;
    if (g.comboT > 0 && --g.comboT === 0){ if (g.combo >= 3){ coinsRun += g.combo; g.msg = g.combo + 'er COMBO! +' + g.combo; g.msgT = 60; beep(1300,.2,'triangle',.07,300); } g.combo = 0; }
    if (g.msgT) g.msgT--;
    for (const it of g.items){ it.vy += 0.3; it.x += it.vx; it.y += it.vy; it.rot += it.vr;
      if (!it.dead && it.vy > 0 && it.y > H + 50){ it.dead = true; if (!it.bomb){ g.miss++; beep(150,.2,'square',.06,-60); if (g.miss >= 3){ P.shield = 0; die(); return; } } } }
    for (const h of g.halves){ h.vy += 0.35; h.x += h.vx; h.y += h.vy; h.rot += h.vr; }
    g.items = g.items.filter(it => !it.dead); g.halves = g.halves.filter(h => h.y < H + 80);
    if (g.cut >= g.goal) exitDim(false);
  },
  d(){
    const g = mg;
    ctx.fillStyle = '#5b3a1e'; ctx.fillRect(0,0,W,H);
    for (let i=0;i<7;i++){ const x = i*72; ctx.fillStyle = i%2 ? '#6e4726' : '#654122'; ctx.fillRect(x, 0, 72, H); ctx.strokeStyle = '#4a2e15'; ctx.lineWidth = 2; ctx.strokeRect(x, 0, 72, H);
      ctx.strokeStyle = 'rgba(0,0,0,.15)'; ctx.beginPath(); ctx.ellipse(x+36, (i*137)%H, 10, 30, 0, 0, 7); ctx.stroke(); }
    const vg = ctx.createRadialGradient(W/2,H/2,100,W/2,H/2,520); vg.addColorStop(0,'rgba(0,0,0,0)'); vg.addColorStop(1,'rgba(0,0,0,.55)'); ctx.fillStyle = vg; ctx.fillRect(0,0,W,H);
    for (const h of g.halves){
      ctx.save(); ctx.translate(h.x, h.y); ctx.rotate(h.rot);
      const a0 = h.s>0 ? 0 : Math.PI, a1 = h.s>0 ? Math.PI : Math.PI*2;
      ctx.fillStyle = h.f.c; ctx.beginPath(); ctx.arc(0,0,h.f.r, a0, a1); ctx.fill();
      ctx.fillStyle = h.f.in; ctx.beginPath(); ctx.arc(0,0,h.f.r-4, a0, a1); ctx.fill();
      ctx.restore();
    }
    for (const it of g.items){
      ctx.save(); ctx.translate(it.x, it.y); ctx.rotate(it.rot);
      if (it.bomb){
        ctx.fillStyle = '#222'; ctx.beginPath(); ctx.arc(0,0,22,0,7); ctx.fill();
        ctx.strokeStyle = '#ff2d2d'; ctx.lineWidth = 3; ctx.stroke();
        ctx.fillStyle = '#555'; ctx.fillRect(-5,-28,10,8);
        ctx.fillStyle = Math.floor(g.t/4)%2 ? '#ffd23f' : '#ff6a1a'; ctx.beginPath(); ctx.arc(0,-32,5,0,7); ctx.fill();
        ctx.fillStyle = '#fff'; ctx.font = 'bold 18px Arial'; ctx.textAlign = 'center'; ctx.fillText('✕', 0, 6);
      } else {
        const f = it.f;
        const fg = ctx.createRadialGradient(-f.r*0.3,-f.r*0.3,2,0,0,f.r); fg.addColorStop(0,'#fff'); fg.addColorStop(.25,f.c); fg.addColorStop(1,f.c);
        ctx.fillStyle = fg; ctx.beginPath(); ctx.arc(0,0,f.r,0,7); ctx.fill();
        if (f.n === 'Melone'){ ctx.strokeStyle = '#1b6b2a'; ctx.lineWidth = 3; for (let k=-2;k<=2;k++){ ctx.beginPath(); ctx.ellipse(k*8,0,3,f.r-2,0,0,7); ctx.stroke(); } }
        ctx.fillStyle = '#3b7a1e'; ctx.beginPath(); ctx.ellipse(4,-f.r,6,3,0.5,0,7); ctx.fill();
      }
      ctx.restore();
    }
    mgDrawParts();
    if (g.trail.length > 1){
      ctx.lineCap = 'round';
      for (let i=1;i<g.trail.length;i++){ const a = g.trail[i-1], b = g.trail[i];
        ctx.strokeStyle = `rgba(255,255,255,${b.l/12})`; ctx.lineWidth = b.l*0.9; ctx.beginPath(); ctx.moveTo(a.x,a.y); ctx.lineTo(b.x,b.y); ctx.stroke(); }
      ctx.lineCap = 'butt';
    }
    ctx.textAlign = 'right'; ctx.font = 'bold 30px Segoe UI';
    for (let i=0;i<3;i++){ ctx.fillStyle = i < g.miss ? '#ff3b30' : 'rgba(255,255,255,.35)'; ctx.fillText('✕', W-16-i*32, 105); }
    if (g.msgT){ ctx.textAlign='center'; ctx.globalAlpha = Math.min(1, g.msgT/20); ctx.fillStyle = '#ffd23f'; ctx.font = '900 30px Segoe UI'; ctx.fillText(g.msg, W/2, 200); ctx.globalAlpha = 1; }
    mgHud('FRUIT NINJA', '🍉 ' + g.cut + ' / ' + g.goal);
    if (g.t < 150){ ctx.textAlign='center'; ctx.fillStyle = 'rgba(255,255,255,.9)'; ctx.font = 'bold 18px Segoe UI'; ctx.fillText('Gedrückt halten & durch Früchte wischen', W/2, H/2); ctx.font = '15px Segoe UI'; ctx.fillText('💣 Bomben meiden · max. 3 Früchte verpassen', W/2, H/2+26); }
  }
};

// ---------- Space Invaders ----------
MINI.si = {
  e(){
    mode = 'si'; parts = [];
    const al = [];
    for (let r=0;r<4;r++) for (let c=0;c<7;c++) al.push({c, r, alive:true});
    mg = {px: W/2, al, ox: 40, oy: 130, dir: 1, shots:[], bombs:[], lives:3, t:0, cool:0, stars: Array.from({length:70},()=>({x:rand(0,W),y:rand(0,H),s:rand(.5,2)})), ufo:null, bunkers:[]};
    for (let b=0;b<4;b++) for (let i=0;i<6;i++) for (let j=0;j<3;j++) mg.bunkers.push({x: 40 + b*115 + i*9, y: H-210 + j*9, hp:2});
  },
  u(){
    const g = mg; g.t++; mgParts();
    const L = keys.ArrowLeft || keys.KeyA || touch.l, Rr = keys.ArrowRight || keys.KeyD || touch.r;
    if (ptr.down && !L && !Rr) g.px += clamp(ptr.x - g.px, -6, 6);
    if (L) g.px -= 6; if (Rr) g.px += 6;
    g.px = clamp(g.px, 24, W-24);
    if (--g.cool <= 0){ g.shots.push({x:g.px, y:H-150}); g.cool = 13; beep(880,.04,'square',.025,-300); }
    for (const s of g.shots) s.y -= 11;
    const alive = g.al.filter(a => a.alive);
    const spd = 0.5 + (28 - alive.length)*0.07;
    g.ox += g.dir * spd;
    let minX = 99, maxX = -1; for (const a of alive){ minX = Math.min(minX, a.c); maxX = Math.max(maxX, a.c); }
    if (g.ox + maxX*48 + 34 > W - 10 || g.ox + minX*48 < 10){ g.dir *= -1; g.oy += 18; g.ox += g.dir*spd; }
    if (Math.random() < 0.008 + (28-alive.length)*0.0006 && alive.length){ const a = alive[rand(0,alive.length)|0]; g.bombs.push({x:g.ox + a.c*48 + 17, y:g.oy + a.r*40 + 26}); }
    if (!g.ufo && Math.random() < 0.002) g.ufo = {x:-40, v:2.2};
    if (g.ufo){ g.ufo.x += g.ufo.v; if (g.ufo.x > W+40) g.ufo = null; }
    for (const b of g.bombs) b.y += 3.6;
    for (const s of g.shots){
      for (const a of alive){ const ax = g.ox + a.c*48, ay = g.oy + a.r*40;
        if (a.alive && s.x > ax && s.x < ax+34 && s.y > ay && s.y < ay+26){ a.alive = false; s.y = -99; coinsRun++; burst(ax+17, ay+13, 12, ['#39ff14','#fff','#0ff'], 4); beep(200,.1,'square',.05,-100); break; } }
      if (g.ufo && Math.abs(s.x - g.ufo.x) < 26 && Math.abs(s.y - 95) < 14){ burst(g.ufo.x, 95, 24, ['#ff2d55','#fff','#ffd23f'], 6); coinsRun += 5; g.ufo = null; s.y = -99; beep(500,.3,'sawtooth',.07,600); }
      for (const k of g.bunkers) if (k.hp > 0 && Math.abs(s.x - (k.x+4.5)) < 5 && Math.abs(s.y - (k.y+4.5)) < 6){ k.hp--; s.y = -99; break; }
    }
    for (const b of g.bombs){
      for (const k of g.bunkers) if (k.hp > 0 && Math.abs(b.x - (k.x+4.5)) < 6 && Math.abs(b.y - (k.y+4.5)) < 6){ k.hp--; b.y = H+99; break; }
      if (Math.abs(b.x - g.px) < 22 && b.y > H-150 && b.y < H-120){
        b.y = H + 99;
        if (P.shield){ P.shield = 0; burst(g.px, H-135, 20, ['#6cf','#fff']); continue; }
        g.lives--; shake = 10; burst(g.px, H-135, 26, ['#39ff14','#ff0','#fff'], 6); beep(120,.35,'sawtooth',.09,-60);
        if (g.lives <= 0){ die(); return; }
      }
    }
    for (const a of alive){ const ay = g.oy + a.r*40; if (ay + 26 > H - 210){ g.lives = 0; die(); return; }
      for (const k of g.bunkers){ const ax = g.ox + a.c*48; if (k.hp > 0 && k.x > ax && k.x < ax+34 && k.y > ay && k.y < ay+26) k.hp = 0; } }
    g.shots = g.shots.filter(s => s.y > 40); g.bombs = g.bombs.filter(b => b.y < H);
    if (!g.al.some(a => a.alive)) exitDim(false);
  },
  d(){
    const g = mg;
    ctx.fillStyle = '#05050f'; ctx.fillRect(0,0,W,H);
    for (const st of g.stars){ ctx.fillStyle = `rgba(255,255,255,${.4+.4*Math.sin(time*.05+st.x)})`; ctx.fillRect(st.x, (st.y + time*st.s*0.3) % H, st.s, st.s); }
    const cols = ['#ff2d55','#ff9f1a','#ffd23f','#0ff'];
    const fr = Math.floor(g.t/25)%2;
    for (const a of g.al){ if (!a.alive) continue;
      const x = g.ox + a.c*48, y = g.oy + a.r*40;
      ctx.fillStyle = cols[a.r];
      if (a.r === 0){ ctx.fillRect(x+12,y,10,4); ctx.fillRect(x+6,y+4,22,6); ctx.fillRect(x+2,y+10,30,6); ctx.fillRect(x+2,y+16,6,4); ctx.fillRect(x+26,y+16,6,4); ctx.fillRect(fr?x:x+6,y+20,6,4); ctx.fillRect(fr?x+28:x+22,y+20,6,4); }
      else if (a.r < 2){ ctx.fillRect(x+6,y,4,4); ctx.fillRect(x+24,y,4,4); ctx.fillRect(x+4,y+4,26,6); ctx.fillRect(x,y+10,34,8); ctx.fillRect(fr?x:x+4,y+18,6,6); ctx.fillRect(fr?x+28:x+24,y+18,6,6); }
      else { ctx.fillRect(x+4,y+2,26,8); ctx.fillRect(x,y+10,34,8); ctx.fillRect(x+6,y+18,8,4); ctx.fillRect(x+20,y+18,8,4); ctx.fillRect(fr?x+2:x+10,y+22,6,3); ctx.fillRect(fr?x+26:x+18,y+22,6,3); }
      ctx.fillStyle = '#000'; ctx.fillRect(x+10,y+10,5,5); ctx.fillRect(x+20,y+10,5,5);
    }
    if (g.ufo){ ctx.fillStyle = '#ff2d55'; ctx.beginPath(); ctx.ellipse(g.ufo.x, 95, 26, 9, 0, 0, 7); ctx.fill(); ctx.fillStyle = '#ffb3c6'; ctx.beginPath(); ctx.arc(g.ufo.x, 89, 10, Math.PI, 0); ctx.fill();
      for (let i=-1;i<=1;i++){ ctx.fillStyle = (Math.floor(g.t/6)+i)%2 ? '#ffd23f' : '#fff'; ctx.fillRect(g.ufo.x+i*12-2, 95, 4, 4); } }
    for (const k of g.bunkers) if (k.hp > 0){ ctx.fillStyle = k.hp === 2 ? '#39ff14' : '#1a8a0a'; ctx.fillRect(k.x, k.y, 9, 9); }
    ctx.fillStyle = '#fff'; for (const s of g.shots) ctx.fillRect(s.x-1.5, s.y, 3, 12);
    ctx.fillStyle = '#ff5050'; for (const b of g.bombs){ ctx.fillRect(b.x-2, b.y, 4, 12); ctx.fillRect(b.x + (Math.floor(b.y/6)%2?-4:2), b.y+4, 2, 4); }
    mgDrawParts();
    const x = g.px, y = H-140;
    ctx.save(); ctx.shadowColor = '#39ff14'; ctx.shadowBlur = 12; ctx.fillStyle = '#39ff14';
    ctx.fillRect(x-22, y+8, 44, 12); ctx.fillRect(x-16, y+2, 32, 8); ctx.fillRect(x-4, y-8, 8, 12); ctx.restore();
    if (P.shield){ ctx.globalAlpha=.3; ctx.fillStyle='#6cf'; ctx.beginPath(); ctx.arc(x, y+6, 32, 0, 7); ctx.fill(); ctx.globalAlpha=1; }
    ctx.fillStyle = '#39ff14'; ctx.fillRect(0, H-110, W, 2);
    ctx.textAlign = 'left'; ctx.font = 'bold 16px Consolas, monospace'; ctx.fillText('LEBEN ' + '♥'.repeat(Math.max(0,g.lives)), 14, H-88);
    mgHud('SPACE INVADERS', '👾 ' + g.al.filter(a=>a.alive).length + ' übrig');
    if (g.t < 140){ ctx.textAlign='center'; ctx.fillStyle='#fff'; ctx.font='bold 15px Segoe UI'; ctx.fillText('← → bewegen · Maus/Finger ziehen · schießt automatisch', W/2, H/2+60); }
  }
};

// ---------- Snake ----------
const SN = {cols: 20, rows: 26, cell: 22, ox: 20, oy: 110};
MINI.sn = {
  e(){
    mode = 'sn'; parts = [];
    mg = {body:[{x:6,y:13},{x:5,y:13},{x:4,y:13}], dir:{x:1,y:0}, q:[], apple:null, eaten:0, goal:12, step:0, t:0, prevP:true, kp:{}};
    MINI.sn.place();
  },
  place(){ let a; do { a = {x:rand(0,SN.cols)|0, y:rand(0,SN.rows)|0}; } while (mg.body.some(b => b.x===a.x && b.y===a.y)); mg.apple = a; },
  turn(d){
    const last = mg.q.length ? mg.q[mg.q.length-1] : mg.dir;
    if (d.x === -last.x && d.y === -last.y) return; if (d.x === last.x && d.y === last.y) return;
    if (mg.q.length < 3) mg.q.push(d);
  },
  u(){
    const g = mg; g.t++; mgParts();
    const map = {ArrowUp:{x:0,y:-1}, KeyW:{x:0,y:-1}, ArrowDown:{x:0,y:1}, KeyS:{x:0,y:1}, ArrowLeft:{x:-1,y:0}, KeyA:{x:-1,y:0}, ArrowRight:{x:1,y:0}, KeyD:{x:1,y:0}};
    for (const k in map){ if (keys[k] && !g.kp[k]) MINI.sn.turn(map[k]); g.kp[k] = keys[k]; }
    if (ptr.down && !g.prevP && g.t > 10){
      const last = g.q.length ? g.q[g.q.length-1] : g.dir;
      MINI.sn.turn(ptr.x < W/2 ? {x:last.y, y:-last.x} : {x:-last.y, y:last.x});
    }
    g.prevP = ptr.down;
    if (g.t < 40) return;
    const every = Math.max(4, 8 - Math.floor(g.eaten/3));
    if (++g.step < every) return; g.step = 0;
    if (g.q.length) g.dir = g.q.shift();
    const h = {x: g.body[0].x + g.dir.x, y: g.body[0].y + g.dir.y};
    if (h.x < 0 || h.y < 0 || h.x >= SN.cols || h.y >= SN.rows || g.body.some((b,i) => i < g.body.length-1 && b.x===h.x && b.y===h.y)){
      P.shield = 0; burst(SN.ox + g.body[0].x*SN.cell + 11, SN.oy + g.body[0].y*SN.cell + 11, 30, ['#2ecc71','#fff','#e63946'], 6); die(); return; }
    g.body.unshift(h);
    if (h.x === g.apple.x && h.y === g.apple.y){
      g.eaten++; coinsRun++; beep(700 + g.eaten*30,.08,'square',.05,300);
      burst(SN.ox + h.x*SN.cell + 11, SN.oy + h.y*SN.cell + 11, 12, ['#e63946','#fff'], 4);
      if (g.eaten >= g.goal){ exitDim(false); return; }
      MINI.sn.place();
    } else g.body.pop();
  },
  d(){
    const g = mg, C = SN.cell;
    ctx.fillStyle = '#1e3b1e'; ctx.fillRect(0,0,W,H);
    for (let y=0;y<SN.rows;y++) for (let x=0;x<SN.cols;x++){ ctx.fillStyle = (x+y)%2 ? '#aad751' : '#a2d149'; ctx.fillRect(SN.ox + x*C, SN.oy + y*C, C, C); }
    ctx.strokeStyle = '#578a34'; ctx.lineWidth = 4; ctx.strokeRect(SN.ox-2, SN.oy-2, SN.cols*C+4, SN.rows*C+4);
    const a = g.apple, ax = SN.ox + a.x*C + C/2, ay = SN.oy + a.y*C + C/2, pul = 1 + Math.sin(g.t*0.15)*0.08;
    ctx.fillStyle = '#e63946'; ctx.beginPath(); ctx.arc(ax, ay+1, 9*pul, 0, 7); ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,.5)'; ctx.beginPath(); ctx.arc(ax-3, ay-2, 2.5, 0, 7); ctx.fill();
    ctx.fillStyle = '#3b7a1e'; ctx.beginPath(); ctx.ellipse(ax+3, ay-9, 4, 2, -0.5, 0, 7); ctx.fill();
    g.body.forEach((b, i) => {
      const x = SN.ox + b.x*C, y = SN.oy + b.y*C, k = i / g.body.length;
      ctx.fillStyle = `rgb(${Math.round(70 - k*30)},${Math.round(116 + k*40)},${Math.round(232 - k*80)})`;
      rr(x+1, y+1, C-2, C-2, i === 0 ? 8 : 6); ctx.fill();
    });
    const hd = g.body[0], hx = SN.ox + hd.x*C + C/2, hy = SN.oy + hd.y*C + C/2;
    for (const s of [-1, 1]){ const ex = hx + g.dir.x*4 + g.dir.y*s*5, ey = hy + g.dir.y*4 - g.dir.x*s*5;
      ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(ex, ey, 4, 0, 7); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(ex + g.dir.x*1.5, ey + g.dir.y*1.5, 2, 0, 7); ctx.fill(); }
    mgDrawParts();
    mgHud('SNAKE', '🍎 ' + g.eaten + ' / ' + g.goal);
    ctx.textAlign = 'center'; ctx.fillStyle = 'rgba(255,255,255,.85)'; ctx.font = '14px Segoe UI';
    ctx.fillText(isTouch ? 'Links tippen = links abbiegen · Rechts tippen = rechts' : 'Pfeiltasten / WASD steuern', W/2, SN.oy + SN.rows*C + 26);
    if (g.t < 40){ ctx.fillStyle = '#fff'; ctx.font = '900 40px Segoe UI'; ctx.fillText(Math.ceil((40-g.t)/13.3), W/2, H/2); }
  }
};

// ===================== Reels-Modus =====================
let reels = false, reelN = 0, reelKind = null, reelAnim = 0, reelSnap = null, reelEndT = 0, reelWin = false,
    reelStreak = 0, reelBestStreak = +localStorage.getItem('jj_streak') || 0, reelEndSnap = null, reelLikes = 0, reelLiked = false, reelWins = 0, lastSwipe = 0;
const REEL_GOAL = 200, REEL_BTN = {x: W/2-150, y: 665, w: 300, h: 56};
const REEL_INFO = {
  jet: {name:'Jetpack Jump', tag:'#jetpack #doodlejump', task:'Erreiche ' + REEL_GOAL + ' m!', col:'#3a86ff'},
  gd:  {name:'Geometry Dash', tag:'#geometrydash #rhythm', task:'Schaff es bis zum Portal!', col:'#d07bff'},
  hc:  {name:'Hill Climb', tag:'#hillclimb #offroad', task:'Fahr bis zur Zielflagge!', col:'#ffa32b'},
  ab: {name:'Angry Birds', tag:'#angrybirds #schweine', task:'Besiege alle Schweine!', col:'#ff3b3b'},
  fb: {name:'Flappy Bird', tag:'#flappybird #tap', task:'Fliege durch 12 Röhren!', col:'#4cd964'},
  fn: {name:'Fruit Ninja', tag:'#fruitninja #slice', task:'Schneide 20 Früchte!', col:'#ff9f1a'},
  si: {name:'Space Invaders', tag:'#spaceinvaders #retro', task:'Zerstöre alle Aliens!', col:'#39ff14'},
  sn: {name:'Snake', tag:'#snake #nokia', task:'Friss 12 Äpfel!', col:'#2ecc71'}
};
function copyCanvas(){ const c = document.createElement('canvas'); c.width = W; c.height = H; c.getContext('2d').drawImage(cv, 0, 0); return c; }
function menuTap(cx, cy){
  const r = cv.getBoundingClientRect(), x = (cx - r.left)/r.width*W, y = (cy - r.top)/r.height*H;
  if (state === 'menu' && x > REEL_BTN.x && x < REEL_BTN.x + REEL_BTN.w && y > REEL_BTN.y && y < REEL_BTN.y + REEL_BTN.h){ startReels(); return; }
  if (canStart()) start();
}
function startReels(){ reels = true; reelN = 0; reelStreak = 0; reelWins = 0; reelKind = null; state = 'play'; paused = false; nextReel(true); }
function exitReels(){
  bank += coinsRun; localStorage.setItem('jj_coins', bank);
  reels = false; state = 'menu'; mode = 'jet'; resetWorld(); reelAnim = 0;
}
function nextReel(first){
  const now = performance.now(); if (!first && now - lastSwipe < 350) return; lastSwipe = now;
  if (!first){ reelSnap = copyCanvas(); reelAnim = 1; if (mode !== 'reelEnd') reelStreak = 0; }
  bank += coinsRun; localStorage.setItem('jj_coins', bank);
  const kinds = ['jet','gd','hc','ab','fb','fn','si','sn'].filter(k => k !== reelKind);
  reelKind = kinds[Math.floor(Math.random()*kinds.length)];
  resetWorld(); state = 'play'; paused = false; mode = 'jet';
  if (reelKind === 'gd') enterGD(); else if (reelKind === 'hc') enterHC(); else if (reelKind === 'ab') enterAB(); else if (MINI[reelKind]) MINI[reelKind].e();
  flash = 0; reelN++; reelEndT = 0; reelLiked = false;
  reelLikes = Math.floor(rand(800, 250000));
  ptr.down = false; mouseJet = false;
  beep(700,.08,'sine',.05,300);
}
function reelEnd(win){
  reelEndSnap = copyCanvas();
  mode = 'reelEnd'; reelWin = win; reelEndT = 0; parts = [];
  if (win){ reelStreak++; reelWins++; coinsRun += 10; reelLiked = true; reelLikes++;
    if (reelStreak > reelBestStreak){ reelBestStreak = reelStreak; localStorage.setItem('jj_streak', reelBestStreak); }
    burst(W/2, H/2-40, 40, ['#ffd23f','#ff2d87','#3f9','#fff'], 8);
    beep(660,.15,'triangle',.08,300); setTimeout(()=>beep(990,.25,'triangle',.08,300),120);
  } else { reelStreak = 0; shake = 12; beep(160,.4,'sawtooth',.09,-80); }
}
function fmtK(n){ return n >= 1e6 ? (n/1e6).toFixed(1).replace('.',',') + ' Mio.' : n >= 1000 ? (n/1000).toFixed(1).replace('.',',') + 'K' : '' + n; }
function drawReelEnd(){
  if (reelEndSnap) ctx.drawImage(reelEndSnap, 0, 0);
  ctx.fillStyle = 'rgba(0,0,0,.6)'; ctx.fillRect(0,0,W,H);
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); }
  ctx.globalAlpha = 1;
  const k = Math.min(1, reelEndT/12), sc = 0.6 + 0.4*k;
  ctx.save(); ctx.translate(W/2, H/2 - 40); ctx.scale(sc, sc); ctx.globalAlpha = k;
  ctx.fillStyle = 'rgba(20,20,35,.92)'; rr(-170, -120, 340, 240, 22); ctx.fill();
  ctx.strokeStyle = reelWin ? '#3f9' : '#ff4d4d'; ctx.lineWidth = 3; ctx.stroke();
  ctx.textAlign = 'center';
  ctx.font = '50px Segoe UI'; ctx.fillText(reelWin ? '🏆' : '💀', 0, -55);
  ctx.font = '900 32px Segoe UI'; ctx.fillStyle = reelWin ? '#3f9' : '#ff4d4d'; ctx.fillText(reelWin ? 'GESCHAFFT!' : 'VERLOREN!', 0, 0);
  ctx.font = 'bold 16px Segoe UI'; ctx.fillStyle = '#fff';
  ctx.fillText(reelWin ? '+10 Bonus-Münzen  ·  🔥 Serie ' + reelStreak : 'Serie verloren', 0, 32);
  ctx.fillStyle = '#aab'; ctx.font = '14px Segoe UI'; ctx.fillText('Beste Serie: ' + reelBestStreak + '  ·  Gewonnen: ' + reelWins + '/' + reelN, 0, 56);
  ctx.fillStyle = '#fff'; ctx.font = 'bold 16px Segoe UI'; ctx.globalAlpha = k * (0.6 + 0.4*Math.sin(time*0.12));
  ctx.fillText('⬆ Wischen für das nächste Level', 0, 96);
  ctx.restore();
}
function drawReelUI(){
  const info = REEL_INFO[reelKind]; if (!info) return;
  // rechte Icon-Leiste
  const x = W - 32; let y = 360;
  ctx.textAlign = 'center';
  ctx.fillStyle = 'rgba(0,0,0,.28)'; rr(x-26, y-34, 52, 200, 26); ctx.fill();
  ctx.font = '26px Segoe UI'; ctx.fillStyle = reelLiked ? '#ff2d55' : '#fff';
  const pulse = reelLiked && mode === 'reelEnd' ? 1 + Math.max(0, 0.4 - reelEndT*0.02) : 1;
  ctx.save(); ctx.translate(x, y); ctx.scale(pulse, pulse); ctx.fillText(reelLiked ? '❤' : '♡', 0, 0); ctx.restore();
  ctx.font = 'bold 11px Segoe UI'; ctx.fillStyle = '#fff'; ctx.fillText(fmtK(reelLikes), x, y + 16);
  ctx.font = '22px Segoe UI'; ctx.fillText('💬', x, y + 58); ctx.font = 'bold 11px Segoe UI'; ctx.fillText(fmtK(Math.floor(reelLikes/37)), x, y + 74);
  ctx.font = '22px Segoe UI'; ctx.fillText('↗', x, y + 112); ctx.font = 'bold 11px Segoe UI'; ctx.fillText('Teilen', x, y + 128);
  // Streak oben rechts
  if (reelStreak > 0){ ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W-82, 74, 70, 28, 14); ctx.fill(); ctx.fillStyle = '#ffb03a'; ctx.font = 'bold 15px Segoe UI'; ctx.fillText('🔥 ' + reelStreak, W-47, 94); }
  // Caption unten links
  const g = ctx.createLinearGradient(0, H-130, 0, H); g.addColorStop(0,'rgba(0,0,0,0)'); g.addColorStop(1,'rgba(0,0,0,.55)');
  ctx.fillStyle = g; ctx.fillRect(0, H-130, W, 130);
  ctx.textAlign = 'left';
  ctx.fillStyle = info.col; ctx.beginPath(); ctx.arc(24, H-62, 12, 0, 7); ctx.fill();
  ctx.fillStyle = '#fff'; ctx.font = 'bold 13px Segoe UI'; ctx.fillText('🎮', 16, H-57);
  ctx.font = 'bold 14px Segoe UI'; ctx.fillText('@jetpackjump · Reel #' + reelN, 42, H-57);
  ctx.font = '13px Segoe UI'; ctx.fillStyle = 'rgba(255,255,255,.92)';
  ctx.fillText(info.name + ': ' + info.task + '  ' + info.tag, 14, H-34);
  // Ziel-Fortschritt im Jetpack-Modus
  if (reelKind === 'jet' && mode === 'jet'){
    const pr = clamp(score/REEL_GOAL, 0, 1);
    ctx.fillStyle = 'rgba(0,0,0,.4)'; rr(W/2-90, 84, 180, 14, 7); ctx.fill();
    ctx.fillStyle = '#3f9'; if (pr > 0.05){ rr(W/2-90, 84, 180*pr, 14, 7); ctx.fill(); }
    ctx.fillStyle = '#fff'; ctx.font = 'bold 11px Segoe UI'; ctx.textAlign = 'center'; ctx.fillText('ZIEL ' + score + ' / ' + REEL_GOAL + ' m', W/2, 95);
  }
  // Fortschrittsbalken wie bei Reels
  ctx.fillStyle = 'rgba(255,255,255,.3)'; ctx.fillRect(0, H-3, W, 3);
  ctx.fillStyle = '#fff'; ctx.fillRect(0, H-3, mode === 'reelEnd' ? W : W * ((time % 600)/600), 3);
  // Hinweis
  ctx.textAlign = 'center'; ctx.font = '11px Segoe UI'; ctx.fillStyle = 'rgba(255,255,255,.7)';
  ctx.fillText((isTouch ? '⬆ Nach oben wischen' : '⬆ Mausrad / N') + ' = nächstes Level · ESC = Menü', W/2, H-11);
}
// ---- Wisch-Erkennung ----
let swY = null, swX = 0, swT = 0;
let swY0 = 0;
function swStart(x, y){ swX = x; swY = y; swY0 = y; swT = performance.now(); }
function swEnd(x, y){
  if (swY === null) return; const dy = swY - y, dx = Math.abs(x - swX), dt = performance.now() - swT; swY = null;
  if (!reels || state !== 'play') return;
  const r = cv.getBoundingClientRect(), dyC = dy / r.height * H;
  const yC = (swY0 - r.top) / r.height * H;
  if ((mode === 'ab' || mode === 'fn' || mode === 'si') && yC < H*0.85) return;
  if (dyC > 90 && dt < 450 && dx < dy){ if (ab) ab.drag = false; nextReel(); }
}
cv.addEventListener('touchstart', e => { if (e.touches.length === 1) swStart(e.touches[0].clientX, e.touches[0].clientY); }, {passive:true});
cv.addEventListener('touchend', e => { const t = e.changedTouches[0]; if (t) swEnd(t.clientX, t.clientY); }, {passive:true});
cv.addEventListener('mousedown', e => swStart(e.clientX, e.clientY));
addEventListener('mouseup', e => swEnd(e.clientX, e.clientY));
cv.addEventListener('wheel', e => { e.preventDefault(); if (reels && state === 'play' && e.deltaY > 25) nextReel(); }, {passive:false});

// ===================== Drawing =====================
const SKY = [
  [0,    [110,190,255],[205,235,255]],
  [1200, [70,100,200], [250,160,120]],
  [3000, [20,20,70],   [70,40,110]],
  [5500, [2,2,12],     [12,6,35]]
];
function skyAt(m){
  for (let i=0;i<SKY.length-1;i++){
    const [a,t1,b1] = SKY[i], [b,t2,b2] = SKY[i+1];
    if (m < b){ const t = clamp((m-a)/(b-a),0,1); const L=(x,y)=>x.map((v,k)=>Math.round(v+(y[k]-v)*t)); return [L(t1,t2), L(b1,b2)]; }
  }
  return [SKY[3][1], SKY[3][2]];
}
function drawBackground(){
  const m = Math.max(0, (START_Y - camY - H*0.42) / 10);
  const [top, bot] = skyAt(m);
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0, `rgb(${top})`); g.addColorStop(1, `rgb(${bot})`);
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
  const sa = clamp((m-900)/1800, 0, 1);
  if (sa > 0) for (const s of stars){
    ctx.globalAlpha = sa * (0.6 + 0.4*Math.sin(time*0.05 + s.tw));
    ctx.fillStyle = '#fff'; ctx.fillRect(s.x, posmod(s.y - camY*s.d, H), s.s, s.s);
  }
  ctx.globalAlpha = 1;
  // planet in space
  if (m > 2500){
    ctx.globalAlpha = clamp((m-2500)/1500,0,1);
    const py = posmod(200 - camY*0.03, H+400) - 200;
    const pg = ctx.createRadialGradient(370,py-20,10,390,py,90);
    pg.addColorStop(0,'#ffb27a'); pg.addColorStop(1,'#7a2f6b');
    ctx.fillStyle = pg; ctx.beginPath(); ctx.arc(390,py,80,0,7); ctx.fill();
    ctx.strokeStyle='rgba(255,220,180,.6)'; ctx.lineWidth=4; ctx.beginPath(); ctx.ellipse(390,py,125,22,-0.3,0,7); ctx.stroke();
    ctx.globalAlpha = 1;
  }
  const ca = clamp(1 - (m-800)/1200, 0, 1);
  if (ca > 0) for (const c of clouds){
    const y = posmod(c.y - camY*c.d, H+200) - 100;
    ctx.globalAlpha = ca*0.75; ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.arc(c.x, y, 22*c.s, 0, 7); ctx.arc(c.x+25*c.s, y-10*c.s, 28*c.s, 0, 7); ctx.arc(c.x+55*c.s, y, 22*c.s, 0, 7);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}
function drawMarkers(){
  ctx.font = 'bold 12px Segoe UI, Arial'; ctx.textAlign = 'left';
  const topM = Math.floor((START_Y - camY) / 1000) + 1;
  for (let k = topM; k >= topM - 2 && k > 0; k--){
    const y = START_Y - k*1000;
    ctx.strokeStyle='rgba(255,255,255,.25)'; ctx.setLineDash([8,8]); ctx.lineWidth=1;
    ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke(); ctx.setLineDash([]);
    ctx.fillStyle='rgba(255,255,255,.55)'; ctx.fillText(k*100 + ' m', 8, y-5);
  }
  if (best > 0){
    const y = START_Y - best*10;
    ctx.strokeStyle='rgba(255,210,63,.8)'; ctx.lineWidth=2; ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke();
    ctx.fillStyle='#ffd23f'; ctx.textAlign='right'; ctx.fillText('BEST ' + best + ' m', W-8, y-6);
  }
}
function drawPlatform(p){
  const cols = {n:['#4cd964','#2e9e45'], s:['#4cd964','#2e9e45'], m:['#4aa8ff','#2766c4'], b:['#b9874f','#7a5530']}[p.t];
  ctx.fillStyle = 'rgba(0,0,0,.18)'; rr(p.x+3, p.y+5, p.w, 14, 7); ctx.fill();
  const g = ctx.createLinearGradient(0,p.y,0,p.y+14); g.addColorStop(0,cols[0]); g.addColorStop(1,cols[1]);
  ctx.fillStyle = g; rr(p.x, p.y, p.w, 14, 7); ctx.fill();
  ctx.fillStyle = 'rgba(255,255,255,.35)'; rr(p.x+5, p.y+2, p.w-10, 4, 2); ctx.fill();
  if (p.t === 'b'){ ctx.strokeStyle='#4a3018'; ctx.lineWidth=2; ctx.beginPath();
    ctx.moveTo(p.x+p.w*0.35,p.y); ctx.lineTo(p.x+p.w*0.45,p.y+7); ctx.lineTo(p.x+p.w*0.38,p.y+14);
    ctx.moveTo(p.x+p.w*0.7,p.y); ctx.lineTo(p.x+p.w*0.62,p.y+8); ctx.stroke(); }
  if (p.t === 's'){
    const sx = p.x + p.w/2 - 10, comp = p.sp ? 4 : 0;
    ctx.strokeStyle='#ccc'; ctx.lineWidth=3; ctx.beginPath();
    for (let i=0;i<4;i++){ const yy = p.y - 3 - i*(4-comp/4); ctx.moveTo(sx, yy); ctx.lineTo(sx+20, yy-2); }
    ctx.stroke(); ctx.fillStyle='#ff5a5a'; rr(sx-3, p.y-20+comp, 26, 5, 2); ctx.fill();
  }
}
function drawCoin(c){
  const w = Math.abs(Math.cos(c.t)) * 10 + 1.5;
  ctx.fillStyle = '#c98a00'; ctx.beginPath(); ctx.ellipse(c.x, c.y, w+1.5, 11.5, 0, 0, 7); ctx.fill();
  ctx.fillStyle = '#ffd23f'; ctx.beginPath(); ctx.ellipse(c.x, c.y, w, 10, 0, 0, 7); ctx.fill();
  ctx.fillStyle = 'rgba(255,255,255,.6)'; ctx.beginPath(); ctx.ellipse(c.x - w*0.3, c.y-3, w*0.25, 3, 0, 0, 7); ctx.fill();
}
function drawFuel(f){
  const y = f.y + Math.sin(f.t*3)*4;
  ctx.save(); ctx.shadowColor='#ff4d4d'; ctx.shadowBlur=12;
  ctx.fillStyle='#e63946'; rr(f.x, y+6, 28, 32, 5); ctx.fill(); ctx.restore();
  ctx.fillStyle='#333'; rr(f.x+9, y, 10, 8, 2); ctx.fill();
  ctx.fillStyle='#fff'; ctx.font='bold 18px Arial'; ctx.textAlign='center'; ctx.fillText('F', f.x+14, y+29);
}
function drawShieldItem(s){
  const y = s.y + Math.sin(s.t*3)*4;
  ctx.save(); ctx.shadowColor='#6cf'; ctx.shadowBlur=16;
  ctx.fillStyle='rgba(100,200,255,.35)'; ctx.beginPath(); ctx.arc(s.x+15, y+15, 17, 0, 7); ctx.fill();
  ctx.strokeStyle='#bfefff'; ctx.lineWidth=2; ctx.stroke(); ctx.restore();
  ctx.fillStyle='#e8fbff'; ctx.beginPath(); ctx.moveTo(s.x+15,y+5); ctx.lineTo(s.x+24,y+9); ctx.lineTo(s.x+22,y+19); ctx.lineTo(s.x+15,y+26); ctx.lineTo(s.x+8,y+19); ctx.lineTo(s.x+6,y+9); ctx.closePath(); ctx.fill();
}
function drawZap(z){
  const dx = Math.cos(z.ang)*z.len/2, dy = Math.sin(z.ang)*z.len/2;
  const ax = z.cx-dx, ay = z.cy-dy, bx = z.cx+dx, by = z.cy+dy;
  ctx.save(); ctx.shadowColor = '#fff36b'; ctx.shadowBlur = 18;
  for (let pass=0; pass<2; pass++){
    ctx.strokeStyle = pass ? '#ffffff' : '#ffe94a'; ctx.lineWidth = pass ? 2 : 6;
    ctx.beginPath(); ctx.moveTo(ax, ay);
    const n = 9;
    for (let i=1;i<n;i++){ const t=i/n; ctx.lineTo(ax+(bx-ax)*t + rand(-6,6), ay+(by-ay)*t + rand(-6,6)); }
    ctx.lineTo(bx, by); ctx.stroke();
  }
  ctx.restore();
  for (const [x,y] of [[ax,ay],[bx,by]]){
    ctx.fillStyle='#555'; ctx.beginPath(); ctx.arc(x,y,13,0,7); ctx.fill();
    ctx.fillStyle='#ffe94a'; ctx.beginPath(); ctx.arc(x,y,7,0,7); ctx.fill();
    ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(x-2,y-2,2.5,0,7); ctx.fill();
  }
}
function drawDrone(d){
  ctx.save(); ctx.translate(d.x+22, d.y+13); if (!d.alive) ctx.rotate(time*0.3);
  ctx.fillStyle='rgba(120,240,255,.5)'; ctx.beginPath(); ctx.arc(0,-4,11,Math.PI,0); ctx.fill();
  const g = ctx.createLinearGradient(0,-6,0,10); g.addColorStop(0,'#ff6b81'); g.addColorStop(1,'#a3172d');
  ctx.fillStyle=g; ctx.beginPath(); ctx.ellipse(0,2,22,9,0,0,7); ctx.fill();
  ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(0,-4,5,0,7); ctx.fill();
  ctx.fillStyle='#111'; ctx.beginPath(); ctx.arc(Math.sign(P.x-d.x)*2,-4,2.5,0,7); ctx.fill();
  for (let i=-1;i<=1;i++){ ctx.fillStyle = (Math.floor(time/8)+i)%2 ? '#ffd23f' : '#fff'; ctx.beginPath(); ctx.arc(i*11,5,2.2,0,7); ctx.fill(); }
  ctx.restore();
}
function drawPlayer(){
  const blink = P.inv > 0 && Math.floor(P.inv/5) % 2;
  const cx = P.x + P.w/2, y = P.y;
  if (!blink){
    ctx.save(); ctx.translate(cx, y); ctx.scale(P.face, 1); ctx.rotate(P.vx*0.025);
    if (P.thr){
      const len = rand(16, 28);
      const fg = ctx.createLinearGradient(0,40,0,40+len); fg.addColorStop(0,'#fff6a0'); fg.addColorStop(.5,'#ffa31a'); fg.addColorStop(1,'rgba(255,60,0,0)');
      ctx.fillStyle = fg; ctx.beginPath(); ctx.moveTo(-23,40); ctx.lineTo(-9,40); ctx.lineTo(-16,40+len); ctx.closePath(); ctx.fill();
    }
    // jetpack
    ctx.fillStyle='#8d99ae'; rr(-25,14,16,27,5); ctx.fill();
    ctx.fillStyle='#ef8a17'; ctx.fillRect(-25,22,16,4); ctx.fillRect(-25,30,16,3);
    ctx.fillStyle='#555'; rr(-23,38,12,5,2); ctx.fill();
    // legs
    ctx.fillStyle='#283b6e'; rr(-10,36,8,10,3); ctx.fill(); rr(2,36,8,10,3); ctx.fill();
    ctx.fillStyle='#222'; rr(-11,44,10,5,2); ctx.fill(); rr(1,44,11,5,2); ctx.fill();
    // body
    ctx.fillStyle='#eef2f7'; rr(-13,16,26,24,8); ctx.fill();
    ctx.fillStyle='#3a86ff'; ctx.fillRect(-2,18,4,20);
    ctx.fillStyle='#8d99ae'; ctx.fillRect(-13,22,6,3);
    // arm
    ctx.fillStyle='#eef2f7'; rr(4,20,14,7,3); ctx.fill();
    ctx.fillStyle='#f2c9a0'; ctx.beginPath(); ctx.arc(18,23.5,4,0,7); ctx.fill();
    // head
    ctx.fillStyle='#f2c9a0'; ctx.beginPath(); ctx.arc(0,6,12,0,7); ctx.fill();
    ctx.fillStyle='#3b2416'; ctx.beginPath(); ctx.arc(-1,2,12.5,Math.PI*1.05,Math.PI*1.95); ctx.fill(); ctx.fillRect(-13,0,8,6);
    // goggles
    ctx.fillStyle='#222'; ctx.fillRect(-12,3,24,3);
    ctx.fillStyle='#7fdcff'; ctx.beginPath(); ctx.arc(6,5,5,0,7); ctx.fill();
    ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(7.5,3.5,1.6,0,7); ctx.fill();
    ctx.strokeStyle='#7a4a30'; ctx.lineWidth=1.5; ctx.beginPath(); ctx.arc(5,12,3,0.2,Math.PI-0.4); ctx.stroke();
    ctx.restore();
  }
  if (P.shield){
    ctx.save(); ctx.globalAlpha = 0.35 + 0.15*Math.sin(time*0.2);
    ctx.fillStyle='#6cf'; ctx.beginPath(); ctx.arc(cx, y+24, 38, 0, 7); ctx.fill();
    ctx.globalAlpha = 0.9; ctx.strokeStyle='#bfefff'; ctx.lineWidth=2; ctx.stroke(); ctx.restore();
  }
}
function drawMissiles(){
  for (const m of missiles){
    if (m.warn > 0){
      const x = m.dir === 1 ? 30 : W-30, on = Math.floor(m.warn / (m.warn < 60 ? 4 : 8)) % 2;
      ctx.save(); ctx.globalAlpha = on ? 1 : 0.4;
      ctx.fillStyle = m.warn < 60 ? '#ff2d2d' : '#ff8a1a';
      ctx.beginPath(); ctx.moveTo(x, m.sy-18); ctx.lineTo(x+18, m.sy+14); ctx.lineTo(x-18, m.sy+14); ctx.closePath(); ctx.fill();
      ctx.fillStyle='#fff'; ctx.font='bold 20px Arial'; ctx.textAlign='center'; ctx.fillText('!', x, m.sy+10);
      ctx.restore();
    } else {
      ctx.save(); ctx.translate(m.x, m.sy); ctx.scale(m.dir, 1);
      ctx.fillStyle='#d0d6de'; rr(-26,-7,40,14,6); ctx.fill();
      ctx.fillStyle='#e63946'; ctx.beginPath(); ctx.moveTo(14,-7); ctx.lineTo(28,0); ctx.lineTo(14,7); ctx.closePath(); ctx.fill();
      ctx.fillStyle='#e63946'; ctx.beginPath(); ctx.moveTo(-26,-7); ctx.lineTo(-34,-14); ctx.lineTo(-18,-7); ctx.moveTo(-26,7); ctx.lineTo(-34,14); ctx.lineTo(-18,7); ctx.fill();
      ctx.restore();
    }
  }
}
function drawHUD(){
  ctx.textAlign='left'; ctx.fillStyle='rgba(0,0,0,.35)'; rr(10,10,170,62,12); ctx.fill();
  ctx.fillStyle='#fff'; ctx.font='bold 30px Segoe UI, Arial'; ctx.fillText(score + ' m', 22, 44);
  ctx.font='bold 15px Segoe UI, Arial'; ctx.fillStyle='#ffd23f'; ctx.fillText('● ' + coinsRun, 22, 63);
  ctx.fillStyle='rgba(255,255,255,.7)'; ctx.fillText('Best ' + best, 90, 63);
  // fuel bar
  const fx = W-150, fy = 18;
  ctx.fillStyle='rgba(0,0,0,.35)'; rr(fx-10, 10, 150, 46, 12); ctx.fill();
  ctx.fillStyle='#fff'; ctx.font='bold 12px Segoe UI, Arial'; ctx.fillText('JETPACK', fx, fy+10);
  ctx.fillStyle='rgba(255,255,255,.2)'; rr(fx, fy+17, 128, 14, 7); ctx.fill();
  const fr = P.fuel / FUELMAX;
  ctx.fillStyle = fr > .5 ? '#4cd964' : fr > .2 ? '#ffd23f' : (time%20<10 ? '#ff3b30' : '#a00');
  if (fr > 0){ rr(fx, fy+17, Math.max(10,128*fr), 14, 7); ctx.fill(); }
  if (P.shield){ ctx.fillStyle='#6cf'; ctx.font='bold 12px Segoe UI'; ctx.fillText('🛡 SCHILD', fx+62, fy+10); }
  if (isTouch){
    ctx.globalAlpha = 0.18; ctx.fillStyle='#fff';
    rr(8,H-90,W*0.25-16,80,14); ctx.fill(); rr(W*0.25+8,H-90,W*0.25-16,80,14); ctx.fill(); rr(W*0.5+8,H-90,W*0.5-16,80,14); ctx.fill();
    ctx.globalAlpha = 0.6; ctx.fillStyle='#000'; ctx.font='bold 26px Arial'; ctx.textAlign='center';
    ctx.fillText('◀', W*0.125, H-40); ctx.fillText('▶', W*0.375, H-40); ctx.fillText('JET 🔥', W*0.75, H-40);
    ctx.globalAlpha = 1;
  }
}
function panel(lines, yStart){
  ctx.fillStyle='rgba(5,10,30,.72)'; ctx.fillRect(0,0,W,H);
  ctx.textAlign='center';
  for (const [txt, font, col, y] of lines){ ctx.font = font; ctx.fillStyle = col; ctx.fillText(txt, W/2, y); }
}
function draw(){
  ctx.save();
  if (shake > 0.5) ctx.translate(rand(-shake,shake), rand(-shake,shake));
  if (mode === 'warp'){ drawWarp(); ctx.restore(); drawOverlays(); return; }
  if (mode === 'reelEnd'){ drawReelEnd(); ctx.restore(); drawOverlays(); return; }
  if (mode === 'ab'){ drawAB(); ctx.restore(); drawOverlays(); return; }
  if (MINI[mode]){ MINI[mode].d(); ctx.restore(); drawOverlays(); return; }
  if (mode === 'gd'){ drawGD(); ctx.restore(); drawOverlays(); return; }
  if (mode === 'hc'){ drawHC(); ctx.restore(); drawOverlays(); return; }
  drawBackground();
  ctx.save(); ctx.translate(0, -camY);
  drawMarkers();
  plats.forEach(drawPlatform);
  coins.forEach(drawCoin);
  fuels.forEach(drawFuel);
  shields.forEach(drawShieldItem);
  portals.forEach(drawPortal);
  zaps.forEach(drawZap);
  drones.forEach(drawDrone);
  for (const p of parts){ ctx.globalAlpha = clamp(p.life/p.max,0,1); ctx.fillStyle=p.c; ctx.beginPath(); ctx.arc(p.x,p.y,p.s,0,7); ctx.fill(); }
  ctx.globalAlpha = 1;
  if (state !== 'over') drawPlayer();
  ctx.restore();
  drawMissiles();
  ctx.restore();

  if (state === 'play') drawHUD();
  if (bonusT > 0 && state === 'play'){
    ctx.globalAlpha = Math.min(1, bonusT/40); ctx.textAlign = 'center';
    ctx.fillStyle = bonusFail ? '#ff6b6b' : '#3f9'; ctx.font = '900 30px Segoe UI'; ctx.fillText(bonusFail ? 'SCHWEINE ÜBERLEBT!' : 'DIMENSION GESCHAFFT!', W/2, 200);
    ctx.fillStyle = '#ffd23f'; ctx.font = 'bold 20px Segoe UI'; ctx.fillText(bonusFail ? 'Kein Bonus · Tank voll' : '+150 m  ·  +25 Münzen  ·  Tank voll', W/2, 232);
    ctx.globalAlpha = 1;
  }
  drawOverlays();
}
function drawOverlays(){
  if (flash > 0){ ctx.fillStyle = `rgba(255,255,255,${clamp(flash,0,1)})`; ctx.fillRect(0,0,W,H); }
  if (reels && state === 'play') drawReelUI();
  if (state === 'play' && paused) panel([['PAUSE','bold 48px Segoe UI','#fff',H/2],['P zum Fortsetzen','18px Segoe UI','#ccc',H/2+40]]);
  if (state === 'menu'){
    const bob = Math.sin(time*0.05)*8;
    panel([
      ['JETPACK','900 58px Segoe UI, Arial','#ffd23f',200+bob],
      ['JUMP','900 58px Segoe UI, Arial','#4cd964',258+bob],
      ['Doodle Jump  ×  Jetpack Joyride','16px Segoe UI','#cfe3ff',295],
      ['← → / A D   bewegen','bold 18px Segoe UI','#fff',380],
      ['LEERTASTE / ↑ / Maus halten   Jetpack','bold 18px Segoe UI','#fff',410],
      ['Handy: links ◀ ▶ · rechts JET','16px Segoe UI','#aab',440],
      ['Auf Plattformen springen · Treibstoff (F) sammeln','15px Segoe UI','#ffb3b3',490],
      ['Zappern & Raketen ausweichen · Drohnen von oben plattmachen','15px Segoe UI','#ffb3b3',514],
      ['🌀 Lila → Geometry Dash · Orange → Hill Climb · Rot → Angry Birds','bold 14px Segoe UI','#d9a6ff',540],
      ['Rekord: ' + best + ' m    Münzen: ' + bank,'bold 18px Segoe UI','#ffd23f',575],
      [(time%60<40) ? 'LEERTASTE / KLICK ZUM STARTEN' : '','bold 22px Segoe UI','#fff',625],
      ['P = Pause · M = Ton an/aus','13px Segoe UI','#889',760]
    ]);
    const bg = ctx.createLinearGradient(REEL_BTN.x, 0, REEL_BTN.x + REEL_BTN.w, 0);
    bg.addColorStop(0,'#ff2d87'); bg.addColorStop(.5,'#a64dff'); bg.addColorStop(1,'#ff9f1a');
    ctx.save(); ctx.shadowColor = '#ff2d87'; ctx.shadowBlur = 18 + Math.sin(time*0.08)*8;
    ctx.fillStyle = bg; rr(REEL_BTN.x, REEL_BTN.y, REEL_BTN.w, REEL_BTN.h, 16); ctx.fill(); ctx.restore();
    ctx.fillStyle = '#fff'; ctx.font = '900 21px Segoe UI'; ctx.textAlign = 'center';
    ctx.fillText('▶ REELS-MODUS  (R)', W/2, REEL_BTN.y + 31);
    ctx.font = '12px Segoe UI'; ctx.fillStyle = 'rgba(255,255,255,.85)'; ctx.fillText('Wischen = neues Level aus allen Spielen', W/2, REEL_BTN.y + 46);
  }
  if (state === 'over'){
    panel([
      ['GAME OVER','900 50px Segoe UI, Arial','#ff4d4d',250],
      [score + ' m','bold 64px Segoe UI','#fff',340],
      [newBest ? '★ NEUER REKORD! ★' : 'Rekord: ' + best + ' m','bold 22px Segoe UI', newBest ? '#ffd23f' : '#cfe3ff',385],
      ['+' + coinsRun + ' Münzen   (gesamt ' + bank + ')','bold 18px Segoe UI','#ffd23f',430],
      [(performance.now()-overT > 700 && time%60<40) ? 'LEERTASTE / KLICK FÜR NEUSTART' : '','bold 22px Segoe UI','#fff',540]
    ]);
  }
}

// ===================== Loop =====================
resetWorld();
let last = performance.now(), acc = 0;
function loop(t){
  acc += Math.min(100, t - last); last = t;
  while (acc >= 1000/60){ update(); acc -= 1000/60; }
  if (reelAnim > 0 && reelSnap){
    const e = reelAnim * reelAnim * (3 - 2*reelAnim), off = H * e;
    ctx.save(); ctx.translate(0, off); draw(); ctx.restore();
    ctx.drawImage(reelSnap, 0, off - H);
    ctx.fillStyle = 'rgba(0,0,0,.6)'; ctx.fillRect(0, off - 3, W, 6);
  } else draw();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
</script>
</body>
</html>
