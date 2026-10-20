<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>地球中心・黄道と白道 3Dモデル</title>
<style>
html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#03050b;color:#eef4ff;font-family:system-ui,-apple-system,"Segoe UI","Noto Sans JP",sans-serif}
#view{position:fixed;inset:0}
canvas{display:block}
#timePanel{position:fixed;z-index:20;left:50%;bottom:16px;transform:translateX(-50%);width:min(480px,90vw);padding:10px 14px;border:1px solid #ffffff24;border-radius:12px;background:#07101db8;box-shadow:0 6px 20px #0008;backdrop-filter:blur(6px);box-sizing:border-box;display:flex;flex-direction:column;gap:8px}
.control-row{display:flex;align-items:center;gap:10px;width:100%}
#time{flex:1;height:20px;margin:0;cursor:pointer;accent-color:#ffd24d}
button{background:rgba(255,255,255,0.1);border:1px solid rgba(255,255,255,0.2);color:#fff;padding:5px 10px;border-radius:6px;cursor:pointer;font-size:11px}
button:hover{background:rgba(56,189,248,0.3);border-color:#38bdf8}
select{background:#0b1626;color:#fff;border:1px solid rgba(255,255,255,0.2);padding:4px 6px;border-radius:6px;font-size:11px}
#gestureHint{position:fixed;z-index:9;right:12px;bottom:12px;padding:5px 8px;border-radius:7px;background:#07101d99;border:1px solid #ffffff12;font-size:10px;color:#b9c7df;pointer-events:none}
#statusBanner{position:fixed;top:16px;left:50%;transform:translateX(-50%);z-index:20;padding:6px 14px;border-radius:20px;font-size:12px;font-weight:bold;background:rgba(15,23,42,0.85);border:1px solid rgba(255,255,255,0.2);color:#94a3b8;pointer-events:none;transition:all 0.3s}
#statusBanner.active{background:rgba(239,68,68,0.3);border-color:#ef4444;color:#fca5a5;box-shadow:0 0 12px rgba(239,68,68,0.5)}
.hiddenData{display:none}
#error{position:fixed;z-index:30;left:50%;top:50%;transform:translate(-50%,-50%);width:min(680px,90vw);padding:22px;border-radius:14px;background:#160b0b;border:1px solid #ff666688;color:#fff;display:none;line-height:1.6}
</style>
</head>
<body>
<div id="view"></div>
<div id="error"></div>
<div id="statusBanner">日食条件：新月かつ交点付近（±16.8°）</div>

<div id="timePanel">
  <div class="control-row">
    <label for="time" style="font-size:11px;color:#b9c7df;white-space:nowrap;">時間</label>
    <input id="time" type="range" min="0" max="365.2422" step="0.02" value="279">
  </div>
  <div class="control-row" style="justify-content:space-between;">
    <div style="display:flex;gap:6px;">
      <button id="play">▶ 再生</button>
      <button id="reset">↺ リセット</button>
    </div>
    <div style="display:flex;align-items:center;gap:6px;">
      <label for="speedSelect" style="font-size:11px;color:#b9c7df;">再生速度:</label>
      <select id="speedSelect">
        <option value="0.5">0.5倍</option>
        <option value="1" selected>1.0倍</option>
        <option value="2">2.0倍</option>
        <option value="5">5.0倍</option>
      </select>
    </div>
  </div>
</div>

<div id="gestureHint">ドラッグ：モデル回転 ／ ホイール：拡大・縮小</div>
<div class="hiddenData">
<span id="date"></span><span id="sunLon"></span><span id="moonLon"></span><span id="moonLat"></span><span id="elong"></span><span id="nodeDist"></span><span id="status"></span><span id="eclipseStatus"></span><span id="shadowInfo"></span><span id="events"></span>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.min.js"></script>
<script>
(function(){
'use strict';
const err=document.getElementById('error');
function showError(msg){err.style.display='block';err.innerHTML='<b>3Dモデルを起動できませんでした。</b><br>'+msg+'<br><br>インターネット接続を確認して、HTMLをブラウザで再読み込みしてください。';}
if(!window.THREE){showError('Three.jsの読み込みに失敗しました。');return;}

const DEG=Math.PI/180, DAY=86400000;
const YEAR=365.2422, SYN=29.530588, INC=5.1*DEG;
const LIMIT_ANGLE=16.8*DEG; // 日食限界（±16.8°）
const START=new Date('2026-01-01T00:00:00Z');

const NEW_MOONS=[
'2026-01-18T19:53:00Z','2026-02-17T12:02:00Z','2026-03-19T01:25:00Z','2026-04-17T11:53:00Z',
'2026-05-16T20:02:00Z','2026-06-15T02:55:00Z','2026-07-14T09:45:00Z','2026-08-12T17:38:00Z',
'2026-09-11T03:28:00Z','2026-10-10T15:51:00Z','2026-11-09T07:03:00Z','2026-12-09T00:53:00Z'
].map(x=>new Date(x));

const $=id=>document.getElementById(id);
function clamp(v,a,b){return Math.max(a,Math.min(b,v));}
function norm(a){a%=360;return a<0?a+360:a;}
function diff(a,b){let d=Math.abs(norm(a)-norm(b));return Math.min(d,360-d);}
function daysFromStart(t){return (t-START)/DAY;}
function sunLon(t){return norm(280.15+360*daysFromStart(t)/YEAR);}
function previousNewMoon(t){
 let p=new Date(NEW_MOONS[0].getTime()-SYN*DAY);
 for(let i=0;i<NEW_MOONS.length;i++){if(NEW_MOONS[i]<=t)p=NEW_MOONS[i];else break;}
 return p;
}
function moonLon(t){const p=previousNewMoon(t);return norm(sunLon(t)+360*((t-p)/DAY)/SYN);}
function nodeLon(t){
 const anchor=new Date('2026-01-07T11:23:00Z');
 // 19.35°/年 で交点が回転（退行）
 return norm(154 - 19.35 * ((t - anchor) / DAY) / YEAR);
}

// Scene
const scene=new THREE.Scene();
scene.background=new THREE.Color(0x03050b);
const camera=new THREE.PerspectiveCamera(42,innerWidth/innerHeight,0.01,1000);
camera.position.set(0,8.8,15.5);
camera.lookAt(0,0,0);
const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setPixelRatio(Math.min(window.devicePixelRatio||1,2));
renderer.setSize(innerWidth,innerHeight);
$('view').appendChild(renderer.domElement);
const model=new THREE.Group();scene.add(model);
scene.add(new THREE.AmbientLight(0xffffff,0.8));
const light=new THREE.PointLight(0xffffff,2.2,100);light.position.set(5,8,6);scene.add(light);

// Earth
const earth=new THREE.Mesh(new THREE.SphereGeometry(0.72,48,32),new THREE.MeshStandardMaterial({color:0x2870d8,roughness:0.8,metalness:0}));
model.add(earth);
const axis=new THREE.Line(new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0,-1.25,0),new THREE.Vector3(0,1.25,0)]),new THREE.LineBasicMaterial({color:0xffffff,transparent:true,opacity:.35}));model.add(axis);

function circle(radius,color){const p=[];for(let i=0;i<257;i++){const a=i/256*Math.PI*2;p.push(new THREE.Vector3(radius*Math.cos(a),0,radius*Math.sin(a)));}return new THREE.LineLoop(new THREE.BufferGeometry().setFromPoints(p),new THREE.LineBasicMaterial({color,transparent:true,opacity:.8}));}
model.add(circle(7.2,0xffcf4a));
const eclPlane=new THREE.Mesh(new THREE.CircleGeometry(7.2,96),new THREE.MeshBasicMaterial({color:0xffcf4a,transparent:true,opacity:.035,side:THREE.DoubleSide}));eclPlane.rotation.x=Math.PI/2;model.add(eclPlane);

const lunarPath=new THREE.Line(new THREE.BufferGeometry(),new THREE.LineBasicMaterial({color:0x72a9ff,transparent:true,opacity:.95}));model.add(lunarPath);
const lunarPlane=new THREE.Mesh(new THREE.CircleGeometry(7.2,96),new THREE.MeshBasicMaterial({color:0x72a9ff,transparent:true,opacity:.025,side:THREE.DoubleSide}));model.add(lunarPlane);

function body(r,color){return new THREE.Mesh(new THREE.SphereGeometry(r,32,20),new THREE.MeshStandardMaterial({color,emissive:color,emissiveIntensity:.1}));}
const sun=body(.40,0xffd24d),moon=body(.19,0xbfd8ff);model.add(sun,moon);

function label(text,color){const c=document.createElement('canvas');c.width=512;c.height=96;const ctx=c.getContext('2d');ctx.font='bold 34px sans-serif';ctx.textAlign='center';ctx.fillStyle=color;ctx.fillText(text,256,60);const s=new THREE.Sprite(new THREE.SpriteMaterial({map:new THREE.CanvasTexture(c),transparent:true,depthTest:false}));s.scale.set(2.8,.53,1);return s;}
const sunLabel=label('太陽','#ffe27a'),moonLabel=label('月','#d5e6ff');model.add(sunLabel,moonLabel);
const asc=body(.12,0x70f0ad),desc=body(.12,0xff7777);model.add(asc,desc);
const ascLabel=label('昇交点','#86ffc1'),descLabel=label('降交点','#ff9292');model.add(ascLabel,descLabel);

// Eclipse geometry
const eclipseLine=new THREE.Line(new THREE.BufferGeometry(),new THREE.LineBasicMaterial({color:0xffe38a,transparent:true,opacity:.9}));model.add(eclipseLine);
const umbra=new THREE.Mesh(new THREE.CylinderGeometry(.12,.28,4.5,32,1,true),new THREE.MeshBasicMaterial({color:0x000000,transparent:true,opacity:.42,depthWrite:false,side:THREE.DoubleSide}));
const penumbra=new THREE.Mesh(new THREE.CylinderGeometry(.55,.95,4.5,32,1,true),new THREE.MeshBasicMaterial({color:0x6b6257,transparent:true,opacity:.16,depthWrite:false,side:THREE.DoubleSide}));model.add(umbra,penumbra);
const shadowSpot=new THREE.Mesh(new THREE.CircleGeometry(.20,40),new THREE.MeshBasicMaterial({color:0x000000,transparent:true,opacity:.7,depthWrite:false,side:THREE.DoubleSide}));
const penumbraSpot=new THREE.Mesh(new THREE.CircleGeometry(.48,40),new THREE.MeshBasicMaterial({color:0x111111,transparent:true,opacity:.22,depthWrite:false,side:THREE.DoubleSide}));model.add(shadowSpot,penumbraSpot);
const eclipseLabel=label('日食条件成立','#ffe38a');eclipseLabel.scale.set(3.5,.65,1);model.add(eclipseLabel);

function orbitBasis(t){
 const O=nodeLon(t)*DEG,c=Math.cos(INC),s=Math.sin(INC),co=Math.cos(O),so=Math.sin(O);
 const p=new THREE.Vector3(co,0,so);
 const q=new THREE.Vector3(-c*so,s,c*co);
 const n=new THREE.Vector3(s*so,c,-s*co);
 return {p,q,n};
}
function moonPos(t,r=7.2){const u=(moonLon(t)-nodeLon(t))*DEG,b=orbitBasis(t);return b.p.clone().multiplyScalar(r*Math.cos(u)).add(b.q.clone().multiplyScalar(r*Math.sin(u)));}
function sunPos(t,r=7.2){const a=sunLon(t)*DEG;return new THREE.Vector3(r*Math.cos(a),0,r*Math.sin(a));}
function updatePath(t){const p=[];for(let i=0;i<=256;i++){const tt=new Date(t.getTime()+i/256*SYN*DAY);p.push(moonPos(tt));}lunarPath.geometry.dispose();lunarPath.geometry=new THREE.BufferGeometry().setFromPoints(p);const b=orbitBasis(t);lunarPlane.quaternion.setFromUnitVectors(new THREE.Vector3(0,1,0),b.n);}
function orientCylinder(mesh,a,b){const d=b.clone().sub(a),len=d.length();mesh.position.copy(a).add(b).multiplyScalar(.5);mesh.scale.set(1,len/4.5,1);mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0,1,0),d.normalize());}

function updateEclipse(t,s,m){
 const ml=moonLon(t),sl=sunLon(t),nl=nodeLon(t);
 const elong=diff(ml,sl);
 const moonRelNode=norm(ml-nl)*(Math.PI/180);
 const distAsc=Math.min(moonRelNode, 2*Math.PI-moonRelNode);
 const distDesc=Math.abs(moonRelNode-Math.PI);
 const inLimit=(distAsc<=LIMIT_ANGLE)||(distDesc<=LIMIT_ANGLE);

 const active=elong<6&&inLimit;
 eclipseLine.visible=umbra.visible=penumbra.visible=shadowSpot.visible=penumbraSpot.visible=eclipseLabel.visible=active;
 
 const banner=$('statusBanner');
 if(active){
  banner.classList.add('active');
  banner.textContent='● 日食発生！ (新月 ＆ 日食限界±16.8°内)';
 }else if(inLimit){
  banner.classList.remove('active');
  banner.textContent='月が日食限界ゾーン（±16.8°）内に位置';
 }else{
  banner.classList.remove('active');
  banner.textContent='日食条件：新月かつ交点付近（±16.8°）';
 }

 if(!active){return;}
 const d=earth.position.clone().sub(m).normalize();const end=m.clone().addScaledVector(d,5);
 orientCylinder(umbra,m,end);orientCylinder(penumbra,m,end);
 eclipseLine.geometry.dispose();eclipseLine.geometry=new THREE.BufferGeometry().setFromPoints([s.clone(),m.clone(),earth.position.clone()]);
 shadowSpot.position.copy(d.clone().multiplyScalar(.73));penumbraSpot.position.copy(d.clone().multiplyScalar(.725));
 const q=new THREE.Quaternion().setFromUnitVectors(new THREE.Vector3(0,0,1),d);shadowSpot.quaternion.copy(q);penumbraSpot.quaternion.copy(q);eclipseLabel.position.copy(d.clone().multiplyScalar(.98));
}

function setTime(t){
 const s=sunPos(t),m=moonPos(t);sun.position.copy(s);moon.position.copy(m);sunLabel.position.copy(s).add(new THREE.Vector3(0,.55,0));moonLabel.position.copy(m).add(new THREE.Vector3(0,.4,0));
 updatePath(t);

 const b=orbitBasis(t);
 asc.position.copy(b.p).multiplyScalar(7.2);
 desc.position.copy(b.p).multiplyScalar(-7.2);
 ascLabel.position.copy(asc.position).add(new THREE.Vector3(0,.35,0));
 descLabel.position.copy(desc.position).add(new THREE.Vector3(0,.35,0));

 updateEclipse(t,s,m);
 $('time').value=clamp(daysFromStart(t),0,365.2422);
}

$('time').addEventListener('input',()=>setTime(new Date(START.getTime()+Number($('time').value)*DAY)));

let playing=false,last=performance.now();
const playBtn=$('play');
const speedSelect=$('speedSelect');

playBtn.addEventListener('click',()=>{
 playing=!playing;
 playBtn.textContent=playing?'⏸ 停止':'▶ 再生';
 last=performance.now();
});

$('reset').addEventListener('click',()=>{
 playing=false;
 playBtn.textContent='▶ 再生';
 setTime(START);
});

function frame(now){
 requestAnimationFrame(frame);
 if(playing){
  const speed=parseFloat(speedSelect.value);
  let v=Number($('time').value)+((now-last)/1000)*1.5*speed;
  if(v>365.2422)v=0;
  setTime(new Date(START.getTime()+v*DAY));
 }
 last=now;
 renderer.render(scene,camera);
}

addEventListener('resize',()=>{camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();renderer.setSize(innerWidth,innerHeight);});

let dragging=false,px=0,py=0;
let rotX=0,rotY=0;
const canvas=renderer.domElement;
canvas.style.touchAction='none';
function applyRotation(){model.rotation.order='YXZ';model.rotation.y=rotY;model.rotation.x=rotX;}
canvas.addEventListener('pointerdown',e=>{dragging=true;px=e.clientX;py=e.clientY;canvas.setPointerCapture(e.pointerId);});
canvas.addEventListener('pointermove',e=>{if(!dragging)return;const dx=e.clientX-px,dy=e.clientY-py;px=e.clientX;py=e.clientY;rotY+=dx*0.008;rotX+=dy*0.008;rotX=clamp(rotX,-Math.PI/2,Math.PI/2);applyRotation();});
canvas.addEventListener('pointerup',e=>{dragging=false;try{canvas.releasePointerCapture(e.pointerId);}catch(_){};});
canvas.addEventListener('pointercancel',()=>{dragging=false;});
canvas.addEventListener('wheel',e=>{e.preventDefault();camera.position.z=clamp(camera.position.z*Math.exp(e.deltaY*0.001),5.2,30);camera.lookAt(0,0,0);},{passive:false});

let pinchStart=0,pinchDistance=0;
canvas.addEventListener('touchstart',e=>{if(e.touches.length===2){pinchStart=camera.position.z;pinchDistance=Math.hypot(e.touches[0].clientX-e.touches[1].clientX,e.touches[0].clientY-e.touches[1].clientY);}},{passive:true});
canvas.addEventListener('touchmove',e=>{if(e.touches.length!==2)return;const d=Math.hypot(e.touches[0].clientX-e.touches[1].clientX,e.touches[0].clientY-e.touches[1].clientY);camera.position.z=clamp(pinchStart*(pinchDistance/d),5.2,30);camera.lookAt(0,0,0);e.preventDefault();},{passive:false});

applyRotation();
setTime(new Date('2026-10-07T01:20:00Z'));
requestAnimationFrame(frame);
})();
</script>
</body>
</html>
