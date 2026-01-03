<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>📡 Tri-Planetary DSN</title>
<style>
html,body{margin:0;overflow:hidden;background:#00030a;font-family:system-ui}
canvas{display:block}
.hud{
 position:absolute;left:16px;top:16px;
 padding:14px 18px;border-radius:14px;
 background:rgba(255,255,255,0.06);
 backdrop-filter:blur(10px);
 color:#d9f3ff;font-size:13px;min-width:340px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
.warn{color:#ffb366}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

// Scale
const AU=210;

// Sun
const sun={x:w/2,y:h/2};

// Planets
const earth={a:AU,angle:0,r:9,name:"Earth DSN"};
const mars={a:AU*1.52,angle:2.4,r:7,name:"Mars DSN"};

// Spacecraft
const ship={progress:0,angle:0.3,r:3};

// Storm
let storm={active:false,intensity:0};

// Radio
const txPower=160;

// Utils
function pos(a,ang){
 return {x:sun.x+Math.cos(ang)*a,y:sun.y+Math.sin(ang)*a};
}

// Solar storm
function updateStorm(){
 if(!storm.active && Math.random()<0.0008){
  storm.active=true;
  storm.intensity=1+Math.random()*3;
 }
 if(storm.active){
  storm.intensity*=0.995;
  if(storm.intensity<0.1) storm.active=false;
 }
}

// Deep-space link model
function linkQuality(distAU){
 let signal=txPower/(distAU*distAU*60);
 let noise=0.05+Math.random()*0.04;
 if(storm.active) noise+=storm.intensity*0.35;
 let snr=signal/noise;
 let latency=distAU*8.3; // minutes
 return {snr,latency};
}

function update(){
 earth.angle+=0.001;
 mars.angle+=0.0008;

 ship.progress=Math.min(ship.progress+0.00035,1);
 ship.angle=earth.angle*(1-ship.progress)+mars.angle*ship.progress;

 updateStorm();
}

function draw(){
 ctx.fillStyle="rgba(0,3,10,0.35)";
 ctx.fillRect(0,0,w,h);

 update();

 const pe=pos(earth.a,earth.angle);
 const pm=pos(mars.a,mars.angle);
 const ps={
  x:sun.x+Math.cos(ship.angle)*(earth.a+(mars.a-earth.a)*ship.progress),
  y:sun.y+Math.sin(ship.angle)*(earth.a+(mars.a-earth.a)*ship.progress)
 };

 // Orbits
 ctx.strokeStyle="rgba(255,255,255,0.05)";
 ctx.beginPath();ctx.arc(sun.x,sun.y,earth.a,0,Math.PI*2);ctx.stroke();
 ctx.beginPath();ctx.arc(sun.x,sun.y,mars.a,0,Math.PI*2);ctx.stroke();

 // Sun
 ctx.beginPath();ctx.arc(sun.x,sun.y,12,0,Math.PI*2);
 ctx.fillStyle="#ffcc66";ctx.fill();

 // Earth
 ctx.beginPath();ctx.arc(pe.x,pe.y,earth.r,0,Math.PI*2);
 ctx.fillStyle="#2a7fff";ctx.fill();

 // Mars
 ctx.beginPath();ctx.arc(pm.x,pm.y,mars.r,0,Math.PI*2);
 ctx.fillStyle="#ff5533";ctx.fill();

 // Ship
 ctx.beginPath();ctx.arc(ps.x,ps.y,ship.r,0,Math.PI*2);
 ctx.fillStyle="#ffffff";ctx.fill();

 // Distances
 const dES=Math.hypot(ps.x-pe.x,ps.y-pe.y)/AU;
 const dSM=Math.hypot(ps.x-pm.x,ps.y-pm.y)/AU;
 const dEM=Math.hypot(pe.x-pm.x,pe.y-pm.y)/AU;

 const linkEarth=linkQuality(dES);
 const linkMars=linkQuality(dSM);
 const relayPossible=linkMars.snr>3 && linkQuality(dEM).snr>3;

 // Choose best path
 let mode="DIRECT EARTH";
 let snr=linkEarth.snr;
 let latency=linkEarth.latency;

 if(relayPossible && linkMars.snr<linkEarth.snr){
  mode="RELAY VIA MARS";
  snr=Math.min(linkMars.snr,linkQuality(dEM).snr);
  latency=linkMars.latency+linkQuality(dEM).latency;
 }

 let cls=snr>8?"good":snr>4?"mid":"bad";

 // Draw links
 ctx.strokeStyle=cls==="good"?"rgba(120,220,255,0.6)":
                 cls==="mid" ?"rgba(255,200,120,0.6)":
                              "rgba(255,120,120,0.6)";
 ctx.beginPath();
 if(mode==="DIRECT EARTH"){
  ctx.moveTo(pe.x,pe.y);ctx.lineTo(ps.x,ps.y);
 }else{
  ctx.moveTo(pe.x,pe.y);ctx.lineTo(pm.x,pm.y);
  ctx.lineTo(ps.x,ps.y);
 }
 ctx.stroke();

 // HUD
 document.getElementById("hud").innerHTML=`
📡 Tri-Planetary DSN<br>
🚀 Mission Progress: ${(ship.progress*100).toFixed(1)}%<br>
🔁 Mode: <b>${mode}</b><br>
📊 SNR: <span class="${cls}">${snr.toFixed(2)}</span><br>
🕒 Latency: ${latency.toFixed(1)} min<br>
🌞 Solar Storm: ${storm.active?'<span class="warn">ACTIVE</span>':'NO'}<br>
📡 Link Status: <span class="${cls}">${snr>3?'CONNECTED':'UNUSABLE'}</span>
`;

 requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
