@@ -0,0 +1,235 @@
<!DOCTYPE html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Echo Flap — ultrasonic Flappy Bird</title>
<style>
*{box-sizing:border-box}
html,body{margin:0;height:100%;background:#0e1a2b;color:#eaf4ff;font:14px/1.4 "Trebuchet MS",system-ui,sans-serif}
body{display:flex;flex-direction:column}
#bar{display:flex;flex-wrap:wrap;gap:8px;align-items:center;padding:8px 12px;background:#14253d;border-bottom:2px solid #1f3a5f}
button,select{font:inherit;color:#eaf4ff;background:#21406b;border:0;border-radius:8px;padding:6px 12px;cursor:pointer}
button:hover{background:#2c5489}button:focus-visible,select:focus-visible,input:focus-visible{outline:2px solid #ffd23f}
#dot{width:10px;height:10px;border-radius:50%;background:#e5484d}
#dot.on{background:#3ddc84}
#gauge{width:120px;height:10px;background:#0a1626;border-radius:6px;overflow:hidden;position:relative}
#gauge i{position:absolute;top:0;bottom:0;background:#3a6ea5;opacity:.5}
#gauge b{position:absolute;top:-2px;bottom:-2px;width:4px;background:#ffd23f;border-radius:2px;left:0}
#panel{display:none;gap:14px;flex-wrap:wrap;width:100%;padding:6px 0 2px}
#panel.show{display:flex}
#panel label{display:flex;flex-direction:column;font-size:12px;gap:2px}
#panel input[type=range]{width:130px}
#stage{flex:1;min-height:0;display:flex;align-items:center;justify-content:center;padding:8px}
canvas{max-width:100%;max-height:100%;border-radius:14px;box-shadow:0 10px 40px #0008;touch-action:none;cursor:pointer}
.sp{flex:1}
</style></head><body>
<div id="bar">
  <button id="bConn">Connect sensor</button><span id="dot"></span><span id="dTxt">no sensor</span>
  <div id="gauge" title="live distance"><i id="gBand"></i><b id="gPos"></b></div>
  <span class="sp"></span>
  <button id="bOri">Landscape</button>
  <button id="bSnd">Sound on</button>
  <button id="bSet">Settings</button>
  <div id="panel">
    <label>Control mode<select id="sMode"><option value="pos">Position (hand height = bird height)</option><option value="flap">Flap (dip your hand)</option></select></label>
    <label>Input<select id="sIn"><option value="sensor">Ultrasonic sensor</option><option value="mouse">Mouse / touch (test)</option></select></label>
    <label>Difficulty<select id="sDiff"><option value="0">Easy</option><option value="1" selected>Normal</option><option value="2">Hard</option></select></label>
    <label>Theme<select id="sTheme"><option value="0">Day</option><option value="1">Sunset</option><option value="2">Night</option></select></label>
    <label>Near (cm) <span id="vNear"></span><input type="range" id="sNear" min="3" max="60" step="1"></label>
    <label>Far (cm) <span id="vFar"></span><input type="range" id="sFar" min="10" max="150" step="1"></label>
    <label title="Readings farther than Far + this value are ignored; the bird keeps its last height">Ignore beyond Far + (cm) <span id="vCut"></span><input type="range" id="sCut" min="0" max="200" step="1"></label>
    <label>Game speed <span id="vSpd"></span><input type="range" id="sSpd" min="0.5" max="3" step="0.1"></label>
    <label>Smoothing <span id="vSm"></span><input type="range" id="sSm" min="4" max="30" step="1"></label>
    <label>Volume<input type="range" id="sVol" min="0" max="100"></label>
    <label>&nbsp;<span><button id="bNear">Set near here</button> <button id="bFar">Set far here</button></span></label>
    <label><span><input type="checkbox" id="sInv"> Invert (closer = higher)</span></label>
  </div>
</div>
<div id="stage"><canvas id="c"></canvas></div>
<script>
const $=id=>document.getElementById(id), cv=$('c'), g=cv.getContext('2d');
const clamp=(v,a,b)=>Math.max(a,Math.min(b,v)), lerp=(a,b,t)=>a+(b-a)*t;
let S={portrait:false,mode:'pos',input:'sensor',diff:1,theme:0,near:8,far:35,cut:30,speed:1,sm:14,vol:70,inv:false,mute:false,best:0};
try{Object.assign(S,JSON.parse(localStorage.getItem('echoflap')||'{}'))}catch(e){}
const save=()=>{try{localStorage.setItem('echoflap',JSON.stringify(S))}catch(e){}};

/* ---------- audio (all synthesized) ---------- */
let AC,master;
function ac(){if(!AC){AC=new(window.AudioContext||window.webkitAudioContext)();master=AC.createGain();master.connect(AC.destination)}master.gain.value=S.mute?0:S.vol/100;if(AC.state==='suspended')AC.resume();return AC}
function tone(f,d,type='sine',v=.2,f2,delay=0){if(S.mute)return;const a=ac(),t=a.currentTime+delay,o=a.createOscillator(),n=a.createGain();o.type=type;o.frequency.setValueAtTime(f,t);if(f2)o.frequency.exponentialRampToValueAtTime(f2,t+d);n.gain.setValueAtTime(v,t);n.gain.exponentialRampToValueAtTime(.001,t+d);o.connect(n);n.connect(master);o.start(t);o.stop(t+d+.02)}
function noise(d,v){if(S.mute)return;const a=ac(),b=a.createBuffer(1,a.sampleRate*d,a.sampleRate),x=b.getChannelData(0);for(let i=0;i<x.length;i++)x[i]=(Math.random()*2-1)*(1-i/x.length);const s=a.createBufferSource(),n=a.createGain(),f=a.createBiquadFilter();f.type='lowpass';f.frequency.value=900;s.buffer=b;n.gain.value=v;s.connect(f);f.connect(n);n.connect(master);s.start()}
const sfx={
 flap:()=>{tone(320,.12,'triangle',.16,640);noise(.05,.05)},
 score:()=>{tone(880,.09,'square',.08);tone(1318,.2,'square',.08,null,.08)},
 hit:()=>{noise(.35,.5);tone(220,.35,'sawtooth',.2,40)},
 die:()=>{[392,330,262,196].forEach((f,i)=>tone(f,.2,'triangle',.18,null,.25+i*.15))},
 best:()=>{[523,659,784,1047].forEach((f,i)=>tone(f,.18,'square',.07,null,i*.1))},
 link:()=>{tone(660,.1,'sine',.15);tone(990,.18,'sine',.15,null,.1)},
 start:()=>tone(440,.15,'sine',.15,880)
};

/* ---------- sensor input (Web Serial) + 2nd-stage smoothing ---------- */
let port,reader,dist=null,rawDist=null,lastMsg=0,hist=[],ignored=false;
function feed(cm){rawDist=cm;lastMsg=performance.now();
  if(cm>S.far+S.cut){ignored=true;return} // out of range: drop reading, bird holds previous height (link stays 'live')
  ignored=false;
  if(dist===null)dist=cm;else{const a=clamp(Math.abs(cm-dist)/8,.25,.9);dist+=(cm-dist)*a} // adaptive: calm=smooth, fast=responsive
  hist.push([lastMsg,dist]);while(hist.length&&lastMsg-hist[0][0]>300)hist.shift()}
async function connect(){
  if(!('serial' in navigator)){alert('Web Serial needs Chrome or Edge on desktop. Use the Mouse input to test.');return}
  try{
    port=await navigator.serial.requestPort();await port.open({baudRate:115200});
    sfx.link();setLink(true);
    const dec=new TextDecoderStream();port.readable.pipeTo(dec.writable);reader=dec.readable.getReader();
    let buf='';
    for(;;){const{value,done}=await reader.read();if(done)break;buf+=value;let i;
      while((i=buf.indexOf('\n'))>=0){const ln=buf.slice(0,i).trim();buf=buf.slice(i+1);
        if(ln.startsWith('D:')){const v=parseFloat(ln.slice(2));if(isFinite(v))feed(v)}}}
  }catch(e){console.warn(e)}
  setLink(false)}
function setLink(on){$('dot').classList.toggle('on',on);if(!on)$('dTxt').textContent='no sensor'}
$('bConn').onclick=connect;

/* ---------- game state ---------- */
let W,H,GY,bird,pipes,parts,clouds,score,state='ready',scroll=0,shake=0,t0=0,newBest=false,flash=0,tMs=0,flapCool=0,playT=0;
const DIFF=[{gap:210,spd:170,gapT:1.7},{gap:178,spd:215,gapT:1.5},{gap:150,spd:260,gapT:1.35}];
function orient(p){S.portrait=p;W=p?540:960;H=p?960:540;cv.width=W;cv.height=H;GY=H-84;$('bOri').textContent=p?'Portrait':'Landscape';
  clouds=Array.from({length:7},(_,i)=>({x:Math.random()*W,y:40+Math.random()*(H*.4),s:.6+Math.random()*1.1,l:Math.random()}));reset();save()}
function reset(){bird={x:W*(S.portrait?.3:.27),y:GY/2,vy:0,tilt:0,r:17,w:0};pipes=[];parts=[];score=0;newBest=false;scroll=0;playT=0;state='ready';spawnT=0}
let spawnT=0;
function spawn(){const d=DIFF[S.diff],m=70,cy=m+d.gap/2+Math.random()*(GY-2*m-d.gap);pipes.push({x:W+60,cy,gap:d.gap,pass:false})}
function puff(x,y,n,cols,sp=160){for(let i=0;i<n;i++){const a=Math.random()*6.28,s=Math.random()*sp;parts.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s,l:1,c:cols[i%cols.length],r:2+Math.random()*3})}}

/* control: target height from sensor/mouse */
let mouseY=.5;
function norm(){ // 0..1 where 1 = top of screen
  let t;
  if(S.input==='mouse')t=1-mouseY;
  else{if(dist===null)return null;t=clamp((dist-S.near)/(S.far-S.near),0,1);if(S.inv)t=1-t}
  return t}
const topY=()=>46, botY=()=>GY-34;
function flap(){if(state==='ready'){state='play';sfx.start()}if(state!=='play')return;bird.vy=-430;sfx.flap();puff(bird.x-10,bird.y+6,6,['#fff','#ffe38a'],90)}

const rampMul=()=>1+Math.min(playT*0.012,1); // +1.2%/s, caps at 2x after ~83s
const spdMul=()=>S.speed*rampMul();
function update(dt){
  const d=DIFF[S.diff],mul=spdMul();tMs+=dt;if(state==='play')playT+=dt;
  const live=S.input==='mouse'||(performance.now()-lastMsg<500);
  const t=norm();
  if(S.mode==='pos'&&t!==null&&live&&state!=='dead'){
    const ty=lerp(botY(),topY(),t),ny=lerp(bird.y,ty,1-Math.exp(-dt*S.sm));
    bird.vy=(ny-bird.y)/dt;bird.y=ny;
    if(state==='ready'&&Math.abs(ty-GY/2)>40&&false)state='play'
  }else if(state==='play'||state==='dead'){bird.vy+=1500*dt;bird.y+=bird.vy*dt}
  else if(state==='ready'&&S.mode==='flap')bird.y=GY/2+Math.sin(tMs*4)*8;
  if(S.mode==='flap'&&S.input==='sensor'&&hist.length>3&&performance.now()-hist[hist.length-1][0]<150&&state!=='dead'){ // dip gesture: hand moves toward sensor fast
    flapCool-=dt;const o=hist[0][1],n=hist[hist.length-1][1];
    if(flapCool<=0&&o-n>4.5){flap();flapCool=.28}}
  bird.tilt=lerp(bird.tilt,clamp(bird.vy/600,-.5,1.1),1-Math.exp(-dt*12));bird.w+=dt*(state==='dead'?0:(14+clamp(-bird.vy,0,500)/40));
  if(state==='ready'&&S.mode==='pos'){bird.y=clamp(bird.y,topY(),botY())}
  if(state!=='dead')scroll+=d.spd*mul*dt*(state==='play'?1:.5);
  if(state==='play'){
    bird.y=clamp(bird.y,-30,GY+40);
    spawnT-=dt;if(spawnT<=0){spawn();spawnT=d.gapT/mul}
    for(const p of pipes){p.x-=d.spd*mul*dt;
      if(!p.pass&&p.x+34<bird.x){p.pass=true;score++;sfx.score();puff(bird.x,bird.y,10,['#ffd23f','#fff'],130);if(score>S.best){S.best=score;if(!newBest&&score>1){newBest=true;sfx.best()}}}
      if(Math.abs(bird.x-(p.x))<34+bird.r*.75){const top=p.cy-p.gap/2,bot=p.cy+p.gap/2;
        if(bird.y-bird.r*.8<top||bird.y+bird.r*.8>bot)die()}}
    pipes=pipes.filter(p=>p.x>-90);
    if(bird.y+bird.r>=GY||bird.y<-20)die()
  }
  if(state==='dead'&&bird.y>GY-bird.r){bird.y=GY-bird.r;bird.vy=0}
  for(const p of parts){p.x+=p.vx*dt;p.y+=p.vy*dt;p.vy+=300*dt;p.l-=dt*1.6}parts=parts.filter(p=>p.l>0);
  for(const c of clouds){c.x-=(6+c.l*18)*dt;if(c.x<-160)c.x=W+160}
  shake=Math.max(0,shake-dt*30);flash=Math.max(0,flash-dt*3)
}
function die(){if(state==='dead')return;state='dead';bird.vy=-200;shake=14;flash=1;sfx.hit();setTimeout(sfx.die,150);puff(bird.x,bird.y,26,['#ffd23f','#ff9f1c','#fff'],260);save()}

/* ---------- rendering ---------- */
const TH=[
 {sky:['#4fc3f7','#b3e5fc','#e1f5fe'],sun:'#fff3b0',hill:['#7ec8a4','#5fb08a'],cloud:'rgba(255,255,255,.9)',pipe:['#2e9e3e','#6fd45a','#1f7a2e'],gr:['#d9c58a','#b9a46a'],grass:'#5dbb3b',txt:'#fff',star:0},
 {sky:['#4a2c6e','#e0576b','#ffb367'],sun:'#ffe0a3',hill:['#7a4a73','#5a3560'],cloud:'rgba(255,200,170,.8)',pipe:['#2a7d6b','#5fc2a6','#1d5a4d'],gr:['#c09a72','#9c7a58'],grass:'#4aa57e',txt:'#fff',star:.3},
 {sky:['#070b24','#14234d','#26407a'],sun:'#e8eefc',hill:['#1c2f55','#142544'],cloud:'rgba(120,150,210,.35)',pipe:['#237a55','#4ec48f','#165539'],gr:['#4a4a6a','#383852'],grass:'#2f8f66',txt:'#dfe9ff',star:1}][0] && null;
const THEMES=[
 {sky:['#3fb4f2','#9bdcfa','#e4f6ff'],sun:'#fff3b0',hill:['#8fd1b0','#69b890'],cloud:'rgba(255,255,255,.92)',pipe:['#2e9e3e','#7be063','#1f7a2e'],gr:['#dcc98f','#bba66c'],grass:'#5dbb3b',star:0},
 {sky:['#4a2c6e','#e0576b','#ffb367'],sun:'#ffe0a3',hill:['#8a5580','#633b68'],cloud:'rgba(255,205,175,.8)',pipe:['#2a8a74','#6fd4b6','#1d5a4d'],gr:['#c09a72','#9c7a58'],grass:'#4aa57e',star:.25},
 {sky:['#060a22','#12214a','#2a4580'],sun:'#e8eefc',hill:['#1f3560','#162a4d'],cloud:'rgba(130,160,220,.3)',pipe:['#237a55','#52c993','#165539'],gr:['#4a4a6a','#383852'],grass:'#2f8f66',star:1}];
const stars=Array.from({length:70},()=>[Math.random(),Math.random()*.6,Math.random()*1.6+.4,Math.random()*6]);
function hills(y,amp,f,col,off){g.fillStyle=col;g.beginPath();g.moveTo(0,GY);for(let x=0;x<=W;x+=12)g.lineTo(x,y-amp*(Math.sin((x+off)*f)+.5*Math.sin((x+off)*f*2.3+1)));g.lineTo(W,GY);g.fill()}
function cloud(c,col){g.fillStyle=col;const x=c.x,y=c.y,s=c.s*34;g.beginPath();g.ellipse(x,y,s*1.7,s*.7,0,0,6.3);g.ellipse(x-s*.8,y+s*.1,s,s*.55,0,0,6.3);g.ellipse(x+s*.9,y+s*.15,s*1.1,s*.55,0,0,6.3);g.ellipse(x+s*.1,y-s*.4,s*.9,s*.6,0,0,6.3);g.fill()}
function drawBg(th){
  const sg=g.createLinearGradient(0,0,0,GY);th.sky.forEach((c,i)=>sg.addColorStop(i/2,c));g.fillStyle=sg;g.fillRect(0,0,W,H);
  if(th.star){for(const s of stars){g.globalAlpha=th.star*(.5+.5*Math.sin(tMs*2+s[3]));g.fillStyle='#fff';g.fillRect(s[0]*W,s[1]*GY,s[2],s[2])}g.globalAlpha=1}
  const sx=W*.78,sy=GY*.22,sgd=g.createRadialGradient(sx,sy,10,sx,sy,110);sgd.addColorStop(0,th.sun);sgd.addColorStop(1,'rgba(255,255,255,0)');g.fillStyle=sgd;g.fillRect(sx-120,sy-120,240,240);g.fillStyle=th.sun;g.beginPath();g.arc(sx,sy,32,0,6.3);g.fill();
  clouds.filter(c=>c.l<.5).forEach(c=>cloud(c,th.cloud));
  hills(GY-40,46,.006,th.hill[0],scroll*.08);hills(GY-6,34,.011,th.hill[1],scroll*.2+300);
  clouds.filter(c=>c.l>=.5).forEach(c=>cloud(c,th.cloud))}
function drawPipe(p,th){
  const x=p.x-34,top=p.cy-p.gap/2,bot=p.cy+p.gap/2;
  const gr=g.createLinearGradient(x,0,x+68,0);gr.addColorStop(0,th.pipe[2]);gr.addColorStop(.3,th.pipe[1]);gr.addColorStop(.6,th.pipe[0]);gr.addColorStop(1,th.pipe[2]);
  g.fillStyle=gr;g.strokeStyle='rgba(0,0,0,.55)';g.lineWidth=3;
  const seg=(y0,y1,cy,ch)=>{g.fillRect(x+5,y0,58,y1-y0);g.strokeRect(x+5,y0,58,y1-y0);g.fillRect(x-4,cy,76,ch);g.strokeRect(x-4,cy,76,ch);g.fillStyle='rgba(255,255,255,.22)';g.fillRect(x+12,y0,6,y1-y0);g.fillRect(x+2,cy+3,6,ch-6);g.fillStyle=gr};
  seg(0,top-30,top-30,30);seg(bot+30,GY,bot,30)}
function drawGround(th){
  const gg=g.createLinearGradient(0,GY,0,H);gg.addColorStop(0,th.gr[0]);gg.addColorStop(1,th.gr[1]);g.fillStyle=gg;g.fillRect(0,GY,W,H-GY);
  g.fillStyle=th.grass;g.fillRect(0,GY,W,14);g.fillStyle='rgba(0,0,0,.25)';g.fillRect(0,GY+14,W,3);
  g.fillStyle='rgba(0,0,0,.1)';const o=-(scroll%40);for(let x=o-40;x<W+40;x+=40){g.beginPath();g.moveTo(x,GY+17);g.lineTo(x+20,GY+17);g.lineTo(x+4,H);g.lineTo(x-16,H);g.fill()}}
function drawBird(){
  g.save();g.translate(bird.x,bird.y);g.rotate(bird.tilt);const r=bird.r,sc=1+clamp(Math.abs(bird.vy)/3000,0,.12);g.scale(sc,1/sc);
  g.fillStyle='rgba(0,0,0,.18)';
  const bg=g.createRadialGradient(-5,-6,2,0,0,r*1.5);bg.addColorStop(0,'#fff0a0');bg.addColorStop(.6,'#ffc928');bg.addColorStop(1,'#f08c00');
  g.fillStyle='#f4a300';g.beginPath();g.moveTo(-r*.9,-2);g.lineTo(-r*1.7,-6+Math.sin(bird.w)*3);g.lineTo(-r*1.5,6);g.fill(); // tail
  g.fillStyle=bg;g.strokeStyle='#8a4b00';g.lineWidth=2.5;g.beginPath();g.ellipse(0,0,r*1.25,r,0,0,6.3);g.fill();g.stroke();
  g.fillStyle='#ffe9a0';g.beginPath();g.ellipse(2,r*.35,r*.8,r*.5,0,0,3.15);g.fill(); // belly
  g.save();g.translate(-4,3);g.rotate(Math.sin(bird.w)*.7-.2);g.fillStyle='#ffb300';g.strokeStyle='#8a4b00';g.beginPath();g.ellipse(-4,0,r*.75,r*.42,0,0,6.3);g.fill();g.stroke();g.restore(); // wing
  g.fillStyle='#fff';g.beginPath();g.arc(r*.55,-r*.35,r*.42,0,6.3);g.fill();g.stroke();g.fillStyle='#111';g.beginPath();g.arc(r*.7,-r*.33,r*.2,0,6.3);g.fill();g.fillStyle='#fff';g.fillRect(r*.72,-r*.42,3,3);
  const bk=g.createLinearGradient(0,0,0,r);bk.addColorStop(0,'#ff6b3d');bk.addColorStop(1,'#d93f1d');g.fillStyle=bk;g.beginPath();g.ellipse(r*1.2,r*.15,r*.55,r*.3,0,0,6.3);g.fill();g.stroke();
  g.restore()}
function text(s,x,y,size,col='#fff',al='center'){g.font=`900 ${size}px "Trebuchet MS",system-ui,sans-serif`;g.textAlign=al;g.lineJoin='round';g.lineWidth=size/6;g.strokeStyle='rgba(20,30,60,.85)';g.strokeText(s,x,y);g.fillStyle=col;g.fillText(s,x,y)}
function render(){
  const th=THEMES[S.theme];g.save();
  if(shake>0)g.translate((Math.random()-.5)*shake,(Math.random()-.5)*shake);
  drawBg(th);pipes.forEach(p=>drawPipe(p,th));drawGround(th);
  // target ring (precision aid)
  const t=norm(),live=S.input==='mouse'||performance.now()-lastMsg<500;
  if(S.mode==='pos'&&t!==null&&live&&state!=='dead'){const ty=lerp(botY(),topY(),t);g.strokeStyle='rgba(255,255,255,.7)';g.setLineDash([6,6]);g.lineWidth=2;g.beginPath();g.moveTo(bird.x-40,ty);g.lineTo(bird.x+40,ty);g.stroke();g.setLineDash([])}
  parts.forEach(p=>{g.globalAlpha=clamp(p.l,0,1);g.fillStyle=p.c;g.beginPath();g.arc(p.x,p.y,p.r,0,6.3);g.fill()});g.globalAlpha=1;
  drawBird();
  const cx=W/2;
  if(state!=='ready'){text(score,cx,H*.14,S.portrait?96:84,'#fff');text('Speed '+spdMul().toFixed(1)+'x',cx,H*.14+34,20,'#ffe38a')}
  if(state==='ready'){
    text('ECHO FLAP',cx,H*.24,S.portrait?84:78,'#ffd23f');
	text('By Hak Se Engineer',cx,H*.74,S.portrait?36:28,'#ff623f');
    const msg=S.input==='mouse'?'Move mouse / finger to fly - click to start':S.mode==='pos'?'Move your hand over the sensor - click or press Space to start':'Dip your hand toward the sensor to flap';
    text(msg,cx,H*.32,S.portrait?24:22,'#fff');
    text('Best '+S.best,cx,H*.39,26,'#ffe38a')}
  if(state==='dead'&&bird.y>=GY-bird.r-1){
    const w=330,h=190,x=cx-w/2,y=H*.3;g.fillStyle='rgba(255,247,220,.96)';g.strokeStyle='#5a3a12';g.lineWidth=4;g.beginPath();g.roundRect(x,y,w,h,18);g.fill();g.stroke();
    text('Game over',cx,y+50,38,'#ff6b3d');
    g.font='700 22px "Trebuchet MS"';g.textAlign='center';g.fillStyle='#5a3a12';g.fillText('Score  '+score,cx,y+90);g.fillText('Best   '+S.best+(newBest?'  NEW!':''),cx,y+122);
    g.font='600 17px "Trebuchet MS"';g.fillText('Click or press Space to retry',cx,y+158)}
  if(!live&&S.input==='sensor'){text('Waiting for sensor...',cx,H-30,22,'#ffb4a8')}
  g.restore();
  if(flash>0){g.fillStyle=`rgba(255,255,255,${flash*.6})`;g.fillRect(0,0,W,H)}}

/* ---------- loop, UI wiring ---------- */
let last=performance.now();
function loop(n){const dt=Math.min(.05,(n-last)/1000);last=n;update(dt);render();
  const live=performance.now()-lastMsg<500&&dist!==null;
  $('dTxt').textContent=live?(ignored?('out of range ('+rawDist.toFixed(0)+' cm) - holding'):dist.toFixed(1)+' cm'):(S.input==='mouse'?'mouse mode':'no sensor');
  if(dist!==null){const p=v=>clamp(v/100,0,1)*100+'%';$('gPos').style.left=p(dist);$('gBand').style.left=p(S.near);$('gBand').style.width=p(S.far-S.near)}
  requestAnimationFrame(loop)}
function action(){ac();if(state==='dead'){if(bird.y>=GY-bird.r-1)reset()}else if(state==='ready'){if(S.mode==='pos'){state='play';sfx.start();spawnT=.5}else flap()}else if(S.mode==='flap')flap()}
cv.addEventListener('pointerdown',action);
addEventListener('keydown',e=>{if(e.code==='Space'){e.preventDefault();action()}if(e.key==='p')S.portrait=!S.portrait,orient(S.portrait)});
cv.addEventListener('pointermove',e=>{const r=cv.getBoundingClientRect();mouseY=clamp((e.clientY-r.top)/r.height,0,1)});
$('bOri').onclick=()=>orient(!S.portrait);
$('bSnd').onclick=()=>{S.mute=!S.mute;$('bSnd').textContent=S.mute?'Sound off':'Sound on';if(AC)master.gain.value=S.mute?0:S.vol/100;save()};
$('bSet').onclick=()=>$('panel').classList.toggle('show');
$('bNear').onclick=()=>{if(dist!==null){S.near=clamp(Math.round(dist),3,60);sync()}};
$('bFar').onclick=()=>{if(dist!==null){S.far=clamp(Math.round(dist),10,150);sync()}};
const bind=(id,key,num=true)=>{$(id).oninput=$(id).onchange=()=>{const e=$(id);S[key]=e.type==='checkbox'?e.checked:(num?+e.value:e.value);if(key==='far'&&S.far<=S.near+4)S.far=S.near+5;if(key==='near'&&S.near>=S.far-4)S.near=S.far-5;if(AC)master.gain.value=S.mute?0:S.vol/100;sync(true)}};
[['sMode','mode',0],['sIn','input',0],['sDiff','diff'],['sTheme','theme'],['sNear','near'],['sFar','far'],['sCut','cut'],['sSpd','speed'],['sSm','sm'],['sVol','vol'],['sInv','inv']].forEach(a=>bind(a[0],a[1],a[2]!==0));
function sync(fromUi){$('sMode').value=S.mode;$('sIn').value=S.input;$('sDiff').value=S.diff;$('sTheme').value=S.theme;$('sNear').value=S.near;$('sFar').value=S.far;$('sCut').value=S.cut;$('sSpd').value=S.speed;$('sSm').value=S.sm;$('sVol').value=S.vol;$('sInv').checked=S.inv;
  $('vNear').textContent=S.near;$('vFar').textContent=S.far;$('vCut').textContent=S.cut;$('vSpd').textContent=(+S.speed).toFixed(1)+'x';$('vSm').textContent=S.sm;$('bSnd').textContent=S.mute?'Sound off':'Sound on';save();if(fromUi&&state==='ready')reset()}
sync();orient(S.portrait);requestAnimationFrame(loop);
</script></body></html>
