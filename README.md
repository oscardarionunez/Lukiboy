# Lukiboy[index.html](https://github.com/user-attachments/files/22047228/index.html)

<!doctype html>
<html lang="es">
<head><meta charset="utf-8"><meta name='viewport' content='width=device-width,initial-scale=1'><title>Mini Espías: Lukas</title>
<style>
html,body{height:100%;margin:0;background:#bfe9ff;display:flex;align-items:center;justify-content:center;font-family:Comic Sans MS, system-ui;}
#wrap{width:100%;max-width:900px;position:relative;}
canvas{width:100%;height:auto;border-radius:12px;background:#87ceeb;box-shadow:0 12px 30px rgba(0,0,0,.25);}
.hud{position:absolute;left:12px;top:12px;display:flex;gap:8px;z-index:20;}
.bubble{background:#fff;padding:6px 10px;border-radius:999px;font-weight:700;}
.pad{position:absolute;left:14px;bottom:14px;width:140px;height:140px;z-index:30;display:grid;place-items:center;opacity:0.9;touch-action:none}
.pad button{width:44px;height:44px;border-radius:8px;border:none;background:#fff;font-size:18px}
.act{position:absolute;right:14px;bottom:24px;display:flex;gap:10px;z-index:30}
.act button{width:64px;height:64px;border-radius:12px;border:none;background:#fff;font-weight:800}
.overlay{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;z-index:50}
.panel{background:#fff;padding:18px;border-radius:12px;text-align:center}
</style></head><body>
<div id="wrap">
  <div class="hud"><div class="bubble">❤️ <span id="ui_life">100</span></div><div class="bubble">⚡ <span id="ui_stam">100</span></div><div class="bubble">😅 <span id="ui_pat">100</span></div><div class="bubble">⭐ <span id="ui_score">0</span></div></div>
  <canvas id="game" width="800" height="600"></canvas>
  <div class="pad" id="pad"><button class="up" data-key="ArrowUp">▲</button><button class="left" data-key="ArrowLeft">◀</button><button class="right" data-key="ArrowRight">▶</button><button class="down" data-key="ArrowDown">▼</button></div>
  <div class="act"><button id="btnAtk">🥋</button><button id="btnDash">🛼</button></div>
  <div class="overlay" id="overlay"><div class="panel"><h2>Mini Espías: Lukas</h2><p>Protege a mamá hasta que vuelva papá.</p><div><button id="startBtn">Jugar</button></div></div></div>
</div>
<script>
const FILES = {
  player_walk: 'player_walk.png',
  player_skate: 'player_skate.png',
  player_swim: 'player_swim.png',
  player_kick: 'player_kick.png',
  mom: 'mom.png',
  pretender: 'pretender.png',
  tileset: 'tileset.png',
  particles: 'particles.png'
};
const canvas = document.getElementById('game'), ctx = canvas.getContext('2d');
const UI = { life: document.getElementById('ui_life'), stam: document.getElementById('ui_stam'), pat: document.getElementById('ui_pat'), score: document.getElementById('ui_score') };
let sprites = {}, loaded=0, tot=Object.keys(FILES).length;
for(const k in FILES){ const img=new Image(); img.src=FILES[k]; img.onload=()=>{ loaded++; }; sprites[k]=img; }

const TILE=40, MAP_W=20, MAP_H=15;
const map = new Array(MAP_H).fill(0).map((_,y)=> new Array(MAP_W).fill(0).map((_,x)=>{
  if (x > 6 && x < 13 && y > 4 && y < 10) return 0;
  if ((x === 6 || x === 13) && y >= 4 && y <= 10) return 1;
  if ((y === 4 || y === 10) && x >= 6 && x <= 13) return 1;
  if (x < 5 && y > 7) return 2;
  return 0;
}));

const mom = { x: (10)*TILE + TILE/2 -12, y: (7)*TILE + TILE/2 -16, w:64, h:64, patience:100 };
const player = { x: mom.x+120, y: mom.y, w:48, h:48, life:100, stam:100, frame:0, frameCD:0, attack:false, atkTimer:0 };
let enemies = [], particles = [], game={ running:false, timeLeft:90, score:0, spawnCD:0, wave:1 };
const keys = {};
window.addEventListener('keydown', e=>{ keys[e.key]=true; if(e.key===' '){ startAttack(); }});
window.addEventListener('keyup', e=>{ keys[e.key]=false; });

document.querySelectorAll('#pad button').forEach(btn=>{ const key = btn.dataset.key; btn.addEventListener('touchstart', e=>{ e.preventDefault(); keys[key]=true; }); btn.addEventListener('touchend', e=>{ e.preventDefault(); keys[key]=false; }); btn.addEventListener('mousedown', e=>{ keys[key]=true; }); btn.addEventListener('mouseup', e=>{ keys[key]=false; }); });
document.getElementById('btnAtk').addEventListener('touchstart', e=>{ e.preventDefault(); startAttack(); });
document.getElementById('btnAtk').addEventListener('mousedown', startAttack);
document.getElementById('btnDash').addEventListener('touchstart', e=>{ e.preventDefault(); keys['Shift']=true; });
document.getElementById('btnDash').addEventListener('touchend', e=>{ e.preventDefault(); keys['Shift']=false; });
document.getElementById('btnDash').addEventListener('mousedown', ()=>keys['Shift']=true);
document.getElementById('btnDash').addEventListener('mouseup', ()=>keys['Shift']=false);

function startAttack(){ if(!player.attack){ player.attack=true; player.atkTimer=18; } }
function spawnEnemy(){ const side=Math.floor(Math.random()*4); let x,y; if(side===0){x=Math.random()*canvas.width; y=-30;} if(side===1){x=canvas.width+30; y=Math.random()*canvas.height;} if(side===2){x=Math.random()*canvas.width; y=canvas.height+30;} if(side===3){x=-30; y=Math.random()*canvas.height;} enemies.push({x,y,w:40,h:40,alive:true,speed:Math.random()*0.6+1.0}); }
function addParticles(x,y,n=6){ for(let i=0;i<n;i++) particles.push({x:x+Math.random()*10-5, y:y+Math.random()*10-5, vx:Math.random()*3-1.5, vy:Math.random()*3-1.5, life:Math.random()*18+8}); }
function tileAt(px,py){ const tx=Math.floor(px/TILE), ty=Math.floor(py/TILE); return (map[ty] && map[ty][tx]!==undefined)? map[ty][tx] : 0; }

let last = performance.now();
function loop(now){ const dtms = Math.min(40, now-last); last=now; if(loaded>=tot && game.running) update(dtms/16.6667); draw(); requestAnimationFrame(loop); }

function update(dt){
  game.timeLeft -= dt/60;
  if(game.timeLeft<=0){ game.timeLeft=0; game.running=false; endGame(true); }
  game.spawnCD -= dt;
  if(game.spawnCD<=0){ spawnEnemy(); game.spawnCD = Math.max(12, 40/game.wave); game.wave += 0.02; }

  let ix=0, iy=0;
  if(keys['ArrowUp']) iy-=1;
  if(keys['ArrowDown']) iy+=1;
  if(keys['ArrowLeft']) ix-=1;
  if(keys['ArrowRight']) ix+=1;
  const tile = tileAt(player.x+player.w/2, player.y+player.h/2);
  player.speed = 2.6;
  if(tile===1){ if(keys['Shift']){ player.speed=4.2; player.stam -= 0.45*dt; } else player.speed=3.4; }
  else if(tile===2) player.speed=2.0;
  else if(!keys['Shift']) player.stam += 0.25*dt;
  player.stam = Math.max(0, Math.min(100, player.stam));
  if(player.stam<=0) player.speed = Math.min(player.speed, 2.0);

  const len = Math.hypot(ix,iy)||1;
  player.x += (ix/len)*player.speed*dt*3.6;
  player.y += (iy/len)*player.speed*dt*3.6;
  player.x = Math.max(2, Math.min(canvas.width-player.w-2, player.x));
  player.y = Math.max(2, Math.min(canvas.height-player.h-2, player.y));

  if(player.attack){ player.atkTimer -= dt; if(player.atkTimer<=0) player.attack=false; }

  enemies.forEach(e=>{
    if(!e.alive) return;
    const d = Math.hypot(mom.x - e.x, mom.y - e.y) || 1;
    e.x += ((mom.x - e.x)/d) * e.speed * dt * 3.2;
    e.y += ((mom.y - e.y)/d) * e.speed * dt * 3.2;
    if(Math.abs(e.x - player.x) < 36 && Math.abs(e.y - player.y) < 36){
      if(player.attack){ e.alive=false; game.score += 10; addParticles(e.x,e.y,8); }
      else { player.life -= 6 * dt; addParticles(player.x, player.y, 4); if(player.life<=0) game.running=false; }
    }
    if(Math.abs(e.x - mom.x) < 28 && Math.abs(e.y - mom.y) < 28){ e.alive=false; mom.patience -= 6; addParticles(mom.x,mom.y,10); if(mom.patience<=0) game.running=false; }
  });

  enemies = enemies.filter(e=>e.alive);
  for(let i=particles.length-1;i>=0;i--){ const p=particles[i]; p.x+=p.vx; p.y+=p.vy; p.vx*=0.96; p.vy*=0.96; p.life-=1; if(p.life<=0) particles.splice(i,1); }

  UI.life.textContent = Math.max(0, Math.floor(player.life));
  UI.stam.textContent = Math.max(0, Math.floor(player.stam));
  UI.pat.textContent = Math.max(0, Math.floor(mom.patience));
  UI.score.textContent = game.score;

  if(player.life<=0 || mom.patience<=0){ game.running=false; endGame(false); }
}

function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  for(let y=0;y<MAP_H;y++){ for(let x=0;x<MAP_W;x++){ const t = map[y][x]; const px = x*TILE, py = y*TILE; if(t===0) ctx.fillStyle='#7ed957'; if(t===1) ctx.fillStyle='#2a3148'; if(t===2) ctx.fillStyle='#0a365a'; ctx.fillRect(px,py,TILE,TILE); ctx.strokeStyle='rgba(0,0,0,0.06)'; ctx.strokeRect(px+0.5,py+0.5,TILE-1,TILE-1); } }
  if(sprites.mom) ctx.drawImage(sprites.mom, mom.x, mom.y, mom.w, mom.h);
  enemies.forEach((e,i)=>{ if(!e.alive) return; const f=Math.floor(performance.now()/200)%4; if(sprites.pretender) ctx.drawImage(sprites.pretender, f*40,0,40,40, e.x, e.y, 40,40); });
  const t = tileAt(player.x+player.w/2, player.y+player.h/2);
  const spr = player.attack ? sprites.player_kick : (t===1 ? sprites.player_skate : (t===2 ? sprites.player_swim : sprites.player_walk));
  if(spr){ player.frameCD++; if(player.frameCD>10){ player.frame=(player.frame+1)%4; player.frameCD=0; } ctx.drawImage(spr, player.frame*48,0,48,48, player.x, player.y, 48,48); }
  particles.forEach(p=>{ ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.fillRect(p.x,p.y,2,2); });
  ctx.fillStyle='rgba(0,0,0,0.25)'; ctx.fillRect(600,12,180,30); ctx.fillStyle='#fff'; ctx.font='bold 14px system-ui'; ctx.fillText('Tiempo: ' + Math.ceil(game.timeLeft), 610, 32);
}

function startGame(){ enemies=[]; particles=[]; game.running=true; game.timeLeft=90; game.score=0; game.spawnCD=0; player.life=100; player.stam=100; mom.patience=100; document.getElementById('overlay').style.display='none'; }
function endGame(win){ document.getElementById('overlay').style.display='flex'; const panel=document.querySelector('.panel'); panel.innerHTML = '<h2>' + (win? '¡Papá volvió!' : 'Fin del turno') + '</h2><p>Puntaje: ' + game.score + '</p><div style="margin-top:12px"><button onclick="startGame()">Reintentar</button></div>'; }

document.getElementById('startBtn').addEventListener('click', ()=>{ startGame(); setInterval(()=>{ if(game.running) spawnEnemy(); }, 2200); });
requestAnimationFrame(loop);
</script></body></html>
