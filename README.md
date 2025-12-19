<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Roguelike Game</title>
<style>
body { margin:0; background:#111; color:white; font-family:Arial,sans-serif; display:flex; flex-direction:column; align-items:center; }
canvas { background: linear-gradient(135deg, #2a2a2a 0%, #1a1a1a 100%); border:3px solid #333; margin-top:20px; }
#gameUI { margin-top:10px; display:flex; gap:15px; }
.stat { background: rgba(50,50,50,0.7); padding:10px; border-radius:8px; }
</style>
</head>
<body>

<h1>Roguelike Dungeon</h1>
<div id="gameUI">
<div class="stat">Комната: <span id="currentRoom">1</span></div>
<div class="stat">Макс: <span id="maxRooms">6</span></div>
<div class="stat">Тип: <span id="roomType">Враги</span></div>
<div class="stat">Урон: <span id="damageStat">1</span></div>
</div>

<canvas id="game" width="800" height="600"></canvas>

<script>
// ================== КОД ИГРЫ ==================
const canvas=document.getElementById('game'); 
const ctx=canvas.getContext('2d');
const currentRoomElement=document.getElementById('currentRoom');
const maxRoomsElement=document.getElementById('maxRooms');
const roomTypeElement=document.getElementById('roomType');
const damageStat=document.getElementById('damageStat');

let mouseX=400,mouseY=300,mouseDown=false;
canvas.addEventListener('mousemove', e=>{ const r=canvas.getBoundingClientRect(); mouseX=e.clientX-r.left; mouseY=e.clientY-r.top; });
canvas.addEventListener('mousedown', e=>{ if(e.button===0) mouseDown=true; });
canvas.addEventListener('mouseup', e=>{ if(e.button===0) mouseDown=false; });
canvas.addEventListener('contextmenu', e=>{ e.preventDefault(); return false; });

const keys={};
window.addEventListener('keydown', e=>keys[e.key.toLowerCase()]=true);
window.addEventListener('keyup', e=>keys[e.key.toLowerCase()]=false);

// Игровые объекты
const player={x:400,y:300,size:20,speed:2,hp:6,damage:1,invuln:0,dirX:0,dirY:-1};
let bullets=[],enemies=[],roomCleared=false,item=null,floatingTexts=[],currentRoomNumber=1,maxRooms=6,mapSize=7;
const map=Array.from({length:mapSize},()=>Array(mapSize).fill(0));
const visitedMap=Array.from({length:mapSize},()=>Array(mapSize).fill(false));
let roomX=3,roomY=3; map[roomY][roomX]=1; visitedMap[roomY][roomX]=true;
let roomType='enemy';

// Изображения игрока и босса
const playerImg=(()=>{
    const c=document.createElement('canvas'); c.width=60;c.height=60;const ct=c.getContext('2d');
    ct.fillStyle='#4CAF50'; ct.beginPath(); ct.arc(30,30,20,0,Math.PI*2); ct.fill();
    ct.fillStyle='#2E7D32'; ct.beginPath(); ct.arc(30,30,15,0,Math.PI*2); ct.fill();
    ct.fillStyle='#FFF'; ct.beginPath(); ct.arc(25,25,4,0,Math.PI*2); ct.arc(35,25,4,0,Math.PI*2); ct.fill();
    ct.fillStyle='#000'; ct.beginPath(); ct.arc(25,25,2,0,Math.PI*2); ct.arc(35,25,2,0,Math.PI*2); ct.fill();
    const img=new Image(); img.src=c.toDataURL(); return img;
})();
const bossImg=(()=>{
    const c=document.createElement('canvas'); c.width=80;c.height=80;const ct=c.getContext('2d');
    ct.fillStyle='#FF4444'; ct.beginPath(); ct.arc(40,40,30,0,Math.PI*2); ct.fill();
    ct.fillStyle='#CC0000'; ct.beginPath(); ct.arc(40,40,25,0,Math.PI*2); ct.fill();
    ct.fillStyle='#000'; ct.beginPath(); ct.arc(30,35,6,0,Math.PI*2); ct.arc(50,35,6,0,Math.PI*2); ct.fill();
    ct.strokeStyle='#000'; ct.lineWidth=4; ct.beginPath(); ct.arc(40,45,12,0,Math.PI); ct.stroke();
    ct.strokeStyle='#FF0000'; ct.lineWidth=3; for(let i=0;i<8;i++){ const a=(i/8)*Math.PI*2; ct.beginPath(); ct.moveTo(40+Math.cos(a)*28,40+Math.sin(a)*28); ct.lineTo(40+Math.cos(a)*35,40+Math.sin(a)*35); ct.stroke(); }
    const img=new Image(); img.src=c.toDataURL(); return img;
})();

// ================== ФУНКЦИИ ==================
function chooseRoomType(){ if(currentRoomNumber===maxRooms) return 'boss'; let r=Math.random()*100; if(r<1) return 'item5'; else if(r<6) return 'item2'; else if(r<26) return 'item1'; else return 'enemy'; }
function spawnEnemies(){ enemies=[]; item=null; roomCleared=false; visitedMap[roomY][roomX]=true; roomType=chooseRoomType();
if(roomType.startsWith('item')){ let count=roomType==='item1'?1:roomType==='item2'?2:5; item={x:400,y:300,size:12,count:count}; roomCleared=true; }
else if(roomType==='boss'){ enemies.push({x:400,y:300,size:30,hp:20,speed:0.6,img:bossImg}); roomCleared=false; }
else{ for(let i=0;i<5;i++){ enemies.push({x:Math.random()*700+50,y:Math.random()*500+50,size:20,hp:3,speed:0.75}); } roomCleared=false; }
updateUI(); }

function updateUI(){ currentRoomElement.textContent=currentRoomNumber; maxRoomsElement.textContent=maxRooms; roomTypeElement.textContent=roomType==='boss'?'Босс':roomType.startsWith('item')?'Предмет':'Враги'; damageStat.textContent=player.damage; }

function shoot(dx,dy){ if(dx===0 && dy===0) dy=-1; bullets.push({x:player.x,y:player.y,dx:dx*8,dy:dy*8,size:6,alive:true,color:'#FFD700'}); }

function addFloatingText(text,x,y,color,size){ floatingTexts.push({text:String(text),x,y,alpha:1,vx:(Math.random()-0.5)*1.5,vy:-1.5-Math.random(),size,color}); }

function canMoveTo(dx,dy){ return roomX+dx>=0 && roomX+dx<mapSize && roomY+dy>=0 && roomY+dy<mapSize; }
function enterDoor(dx,dy){ if(!roomCleared) return; if(!canMoveTo(dx,dy)) return; roomX+=dx; roomY+=dy; currentRoomNumber++; map[roomY][roomX]=1; visitedMap[roomY][roomX]=true; player.x=400; player.y=300; roomType=chooseRoomType(); spawnEnemies(); }

function drawMiniMap(){ const offsetX=canvas.width-miniMapSize*miniTileSize-10; const offsetY=10; for(let y=0;y<miniMapSize;y++){ for(let x=0;x<miniMapSize;x++){ let color='rgba(100,100,100,0.3)'; if(visitedMap[y][x]) color='gray'; if(x===roomX && y===roomY) color='green'; if(map[y][x]!==0 && (roomType==='boss'||roomType.startsWith('item'))) color='gold'; ctx.fillStyle=color; ctx.fillRect(offsetX+x*miniTileSize,offsetY+y*miniTileSize,miniTileSize-2,miniTileSize-2); ctx.strokeStyle='#000'; ctx.strokeRect(offsetX+x*miniTileSize,offsetY+y*miniTileSize,miniTileSize-2,miniTileSize-2); } } }
const miniMapSize=7,miniTileSize=20;

// ================== ОБНОВЛЕНИЕ ==================
function update(){
if(player.hp<=0) return;
if(player.invuln>0) player.invuln--;
if(keys['w']){ player.y-=player.speed; player.dirX=0; player.dirY=-1; }
if(keys['s']){ player.y+=player.speed; player.dirX=0; player.dirY=1; }
if(keys['a']){ player.x-=player.speed; player.dirX=-1; player.dirY=0; }
if(keys['d']){ player.x+=player.speed; player.dirX=1; player.dirY=0; }

player.x=Math.max(20,Math.min(780,player.x)); player.y=Math.max(20,Math.min(580,player.y));

if(mouseDown){ const dx=mouseX-player.x; const dy=mouseY-player.y; const dist=Math.sqrt(dx*dx+dy*dy); if(dist>0) shoot(dx/dist,dy/dist); mouseDown=false; }

bullets.forEach(b=>{ b.x+=b.dx; b.y+=b.dy; });
bullets=bullets.filter(b=>b.x>-50 && b.x<850 && b.y>-50 && b.y<650 && b.alive);

enemies.forEach(e=>{
    const dx=player.x-e.x; const dy=player.y-e.y; const dist=Math.sqrt(dx*dx+dy*dy);
    if(dist>0){ e.x+=dx/dist*e.speed; e.y+=dy/dist*e.speed; }
    const collisionDist=Math.sqrt((player.x-e.x)**2+(player.y-e.y)**2);
    if(collisionDist<player.size+e.size && player.invuln===0){ player.hp--; player.invuln=60; addFloatingText('-1',player.x,player.y-20,'#FF4444',28); }
});
}

// ================== РИСОВАНИЕ ==================
function draw(){
ctx.clearRect(0,0,canvas.width,canvas.height);
ctx.drawImage(playerImg,player.x-30,player.y-30);
bullets.forEach(b=>{ ctx.fillStyle=b.color; ctx.beginPath(); ctx.arc(b.x,b.y,b.size,0,Math.PI*2); ctx.fill(); });
enemies.forEach(e=>{ if(e.img) ctx.drawImage(e.img,e.x-40,e.y-40); else { ctx.fillStyle='red'; ctx.beginPath(); ctx.arc(e.x,e.y,e.size,0,Math.PI*2); ctx.fill(); } });
if(item){ ctx.fillStyle='gold'; ctx.beginPath(); ctx.arc(item.x,item.y,item.size,0,Math.PI*2); ctx.fill(); }
floatingTexts.forEach(t=>{ ctx.globalAlpha=t.alpha; ctx.fillStyle=t.color; ctx.font=t.size+'px Arial'; ctx.fillText(t.text,t.x,t.y); ctx.globalAlpha=1; });
drawMiniMap();
floatingTexts.forEach(t=>{ t.x+=t.vx; t.y+=t.vy; t.alpha-=0.02; }); floatingTexts=floatingTexts.filter(t=>t.alpha>0);
}

// ================== ЦИКЛ ==================
function gameLoop(){ update(); draw(); requestAnimationFrame(gameLoop); }
spawnEnemies(); gameLoop();

// ================== ПЕРЕЗАПУСК ==================
function restartGame(){ location.reload(); }

// ================== КЛАВИАТУРА ==================
window.addEventListener('keydown', e=>{
if(e.key==='r'){ if(canMoveTo(0,-1)) enterDoor(0,-1); else if(canMoveTo(1,0)) enterDoor(1,0); else if(canMoveTo(0,1)) enterDoor(0,1); else if(canMoveTo(-1,0)) enterDoor(-1,0); }
});
</script>

</body>
</html>
