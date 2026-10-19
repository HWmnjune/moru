# moru
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="theme-color" content="#120d0b">
<title>모루</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/galmuri@2.40.3/dist/galmuri.css">
<style>
html,body{margin:0;height:100%;background:#0c0908;overflow:hidden;touch-action:none;-webkit-user-select:none;user-select:none;-webkit-tap-highlight-color:transparent;overscroll-behavior:none}
#c{position:fixed;left:0;top:0;display:block}
</style>
</head>
<body>
<canvas id="c"></canvas>
<script>
'use strict';
// ===== Core =====
const cv=document.getElementById('c'),cx=cv.getContext('2d');
let DPR=1,SC=1,OX=0,W=360,H=640,NOW=0;
const R=Math.random,PI=Math.PI,TAU=PI*2;
const clamp=(v,a,b)=>v<a?a:v>b?b:v,lerp=(a,b,t)=>a+(b-a)*t,rr=(a,b)=>a+R()*(b-a),pick=a=>a[Math.floor(R()*a.length)];
const eOut=t=>1-Math.pow(1-t,3),eIn=t=>t*t*t,eBack=t=>{const c=1.70158;return 1+(c+1)*Math.pow(t-1,3)+c*Math.pow(t-1,2)};
function eBounce(t){const n=7.5625,d=2.75;if(t<1/d)return n*t*t;if(t<2/d)return n*(t-=1.5/d)*t+.75;if(t<2.5/d)return n*(t-=2.25/d)*t+.9375;return n*(t-=2.625/d)*t+.984375}
function srand(s){s=(s>>>0)||1;return()=>{s^=s<<13;s>>>=0;s^=s>>>17;s^=s<<5;s>>>=0;return s/4294967296}}
function hash(str){let h=2166136261;for(let i=0;i<str.length;i++){h^=str.charCodeAt(i);h=Math.imul(h,16777619)}return h>>>0}
function hx(h){const n=parseInt(h.slice(1),16);return[n>>16&255,n>>8&255,n&255]}
function toHex(r,g,b){return'#'+((1<<24)|(r<<16)|(g<<8)|b).toString(16).slice(1)}
function shade(h,a){const c=hx(h).map(v=>clamp(Math.round(a>0?v+(255-v)*a:v*(1+a)),0,255));return toHex(c[0],c[1],c[2])}
function mix(a,b,t){const x=hx(a),y=hx(b);return toHex(Math.round(lerp(x[0],y[0],t)),Math.round(lerp(x[1],y[1],t)),Math.round(lerp(x[2],y[2],t)))}
function rgba(h,a){const c=hx(h);return'rgba('+c[0]+','+c[1]+','+c[2]+','+a+')'}
function fmt(n){n=Math.floor(n);if(n<10000)return''+n;const u=['','만','억','조','경','해'];let i=0,v=n;while(v>=10000&&i<5){v/=10000;i++}return(v<100?Math.floor(v*10)/10:Math.floor(v))+u[i]}
function mkC(w,h){const c=document.createElement('canvas');c.width=w;c.height=h;return c}

// ===== Pixel grid =====
function Grid(w,h){w=Math.ceil(w);h=Math.ceil(h);return{w,h,d:new Array(w*h).fill(null)}}
function gp(g,x,y,c){x=Math.floor(x);y=Math.floor(y);if(x>=0&&y>=0&&x<g.w&&y<g.h)g.d[y*g.w+x]=c}
function gg(g,x,y){return(x<0||y<0||x>=g.w||y>=g.h)?null:g.d[y*g.w+x]}
function gE(g,cx0,cy0,rx,ry,c){for(let y=Math.floor(cy0-ry-1);y<=cy0+ry+1;y++)for(let x=Math.floor(cx0-rx-1);x<=cx0+rx+1;x++){const dx=(x+.5-cx0)/rx,dy=(y+.5-cy0)/ry;if(dx*dx+dy*dy<=1)gp(g,x,y,c)}}
function gR(g,x,y,w,h,c){x=Math.round(x);y=Math.round(y);w=Math.max(1,Math.round(w));h=Math.max(1,Math.round(h));for(let j=0;j<h;j++)for(let i=0;i<w;i++)gp(g,x+i,y+j,c)}
function gL(g,x0,y0,x1,y1,c,t){t=t||1;const n=Math.ceil(Math.max(Math.abs(x1-x0),Math.abs(y1-y0)))||1;for(let i=0;i<=n;i++){const x=lerp(x0,x1,i/n),y=lerp(y0,y1,i/n);if(t>1)gR(g,x-t/2,y-t/2,t,t,c);else gp(g,Math.round(x),Math.round(y),c)}}
function gP(g,pts,c){let x0=1e9,x1=-1e9,y0=1e9,y1=-1e9;for(const p of pts){x0=Math.min(x0,p[0]);x1=Math.max(x1,p[0]);y0=Math.min(y0,p[1]);y1=Math.max(y1,p[1])}for(let y=Math.floor(y0);y<=y1;y++)for(let x=Math.floor(x0);x<=x1;x++){const px=x+.5,py=y+.5;let ins=false;for(let i=0,j=pts.length-1;i<pts.length;j=i++){const a=pts[i],b=pts[j];if((a[1]>py)!==(b[1]>py)&&px<(b[0]-a[0])*(py-a[1])/(b[1]-a[1])+a[0])ins=!ins}if(ins)gp(g,x,y,c)}}
function bake(g,o){o=o||{};const ol=o.ol===undefined?'#140e10':o.ol,pd=ol?1:0,c=mkC(g.w+pd*2,g.h+pd*2),x=c.getContext('2d');
 for(let y=0;y<g.h;y++)for(let i=0;i<g.w;i++){let col=g.d[y*g.w+i];if(!col)continue;
  if(col[0]==='!')col=col.slice(1);else if(o.sh!==false){if(!gg(g,i,y-1))col=shade(col,.24);else if(!gg(g,i,y+1))col=shade(col,-.3);else if(!gg(g,i-1,y))col=shade(col,.1);else if(!gg(g,i+1,y))col=shade(col,-.16)}
  x.fillStyle=col;x.fillRect(i+pd,y+pd,1,1)}
 if(ol){x.fillStyle=ol;for(let y=-1;y<=g.h;y++)for(let i=-1;i<=g.w;i++){if(gg(g,i,y))continue;if(gg(g,i-1,y)||gg(g,i+1,y)||gg(g,i,y-1)||gg(g,i,y+1))x.fillRect(i+1,y+1,1,1)}}
 return c}
function tintOf(img,col){const k='_t'+col;if(img[k])return img[k];const c=mkC(img.width,img.height),x=c.getContext('2d');x.drawImage(img,0,0);x.globalCompositeOperation='source-in';x.fillStyle=col;x.fillRect(0,0,c.width,c.height);img[k]=c;return c}
function spr(img,x,y,s,o){o=o||{};const ax=o.ax===undefined?.5:o.ax,ay=o.ay===undefined?1:o.ay;cx.save();cx.translate(x,y);if(o.rot)cx.rotate(o.rot);cx.scale((o.sx||1)*(o.flip?-1:1),o.sy||1);if(o.a!==undefined)cx.globalAlpha*=o.a;const w=img.width*s,h=img.height*s;cx.drawImage(img,-w*ax,-h*ay,w,h);if(o.fl>0){cx.globalAlpha*=Math.min(1,o.fl);cx.drawImage(tintOf(img,o.flc||'#ffffff'),-w*ax,-h*ay,w,h)}cx.restore()}
function prect(x,y,w,h,c){cx.fillStyle=c;cx.fillRect(x+2,y,w-4,h);cx.fillRect(x,y+2,w,h-4)}

// ===== Text =====
const FN='Galmuri11',FT='Galmuri14',FB=',"Apple SD Gothic Neo","Malgun Gothic",sans-serif';
let CW={};
function font(sz,b,t){cx.font=(b?'bold ':'')+sz+'px '+(t?FT:FN)+FB}
function cw(ch){const k=cx.font+ch;let v=CW[k];if(v===undefined)v=CW[k]=cx.measureText(ch).width||6;return v}
function tw(s,sz,b,t){font(sz,b,t);let w=0;for(const c of s)w+=cw(c);return w}
function text(s,x,y,o){o=o||{};s=''+s;const sz=o.size||11;font(sz,o.bold,o.title);cx.textBaseline='middle';
 const col=o.color||'#f3e9dc',sh=o.shadow===undefined?'#140e10':o.shadow,al=o.align||'center';
 if(!(o.wave||o.pop||o.shake||o.drop||o.colors)){cx.textAlign=al;if(sh){cx.fillStyle=sh;cx.fillText(s,x+1,y+1);if(o.ol){cx.fillText(s,x-1,y);cx.fillText(s,x+1,y-1);cx.fillText(s,x,y-1)}}cx.fillStyle=col;cx.fillText(s,x,y);return}
 cx.textAlign='left';const ch=[...s];let w=0;for(const c of ch)w+=cw(c);
 let px=al==='center'?x-w/2:al==='right'?x-w:x;const t=NOW-(o.t0||0),gap=o.gap||.035;
 for(let i=0;i<ch.length;i++){const c=ch[i],cwid=cw(c);let dy=0,dx=0,sc=1,a=1;
  if(o.pop){const lt=t-i*gap;if(lt<0){px+=cwid;continue}const k=clamp(lt/.22,0,1);sc=lerp(.3,1,eBack(k));dy+=(1-eOut(k))*7;a=clamp(lt/.06,0,1)}
  if(o.drop){const lt=t-i*gap;if(lt<0){px+=cwid;continue}const k=clamp(lt/.38,0,1);dy-=(1-eBounce(k))*26;a=clamp(lt/.05,0,1)}
  if(o.wave)dy+=Math.sin(NOW*(o.spd||5)-i*.6)*o.wave;
  if(o.shake){dx+=(R()-.5)*o.shake;dy+=(R()-.5)*o.shake}
  cx.save();cx.globalAlpha*=a;cx.translate(px+cwid/2+dx,y+dy);if(sc!==1)cx.scale(sc,sc);
  if(sh){cx.fillStyle=sh;cx.fillText(c,-cwid/2+1,1);if(o.ol){cx.fillText(c,-cwid/2-1,0);cx.fillText(c,-cwid/2,-1)}}
  cx.fillStyle=o.colors?o.colors(i,ch.length):col;cx.fillText(c,-cwid/2,0);cx.restore();px+=cwid}}
const TX=[],TW=[];
function ptext(L,s,x,y,o){o=o||{};L.push({s,x,y,t0:NOW,l:o.l||1,vy:o.vy===undefined?-26:o.vy,o:Object.assign({t0:NOW},o)});if(L.length>40)L.shift();if(o.say)AU.blips([...s].length,o.gap||.035)}
function drawTexts(L){for(let i=L.length-1;i>=0;i--){const p=L[i],a=(NOW-p.t0);if(a>p.l){L.splice(i,1);continue}const fa=a>p.l-.25?(p.l-a)/.25:1;cx.save();cx.globalAlpha=fa;const yy=p.y+p.vy*eOut(Math.min(1,a/p.l))*p.l;
 if(p.o.big){const k=clamp(a/.14,0,1),sc=a<.14?lerp(.4,1.35,eOut(k)):lerp(1.35,1,clamp((a-.14)/.12,0,1));cx.translate(p.x,yy);cx.scale(sc,sc);text(p.s,0,0,p.o)}else text(p.s,p.x,yy,p.o);cx.restore()}}

// ===== Particles =====
const PW=[],PS=[],PU=[];
function part(L,o){if(L.length>520)L.shift();L.push(Object.assign({x:0,y:0,vx:0,vy:0,g:0,dr:0,t:0,l:1,s:2,c:'#fff',k:'sq',a:1},o))}
function burst(L,x,y,n,o){for(let i=0;i<n;i++){const an=o.an!==undefined?o.an+rr(-o.sp,o.sp):rr(0,TAU),v=rr(o.v0||40,o.v1||160);part(L,{x,y,vx:Math.cos(an)*v,vy:Math.sin(an)*v,g:o.g||0,dr:o.dr===undefined?2:o.dr,l:rr(o.l0||.3,o.l1||.7),s:o.s?(Array.isArray(o.s)?rr(o.s[0],o.s[1]):o.s):2,c:Array.isArray(o.c)?pick(o.c):o.c,k:o.k||'sq',sh:o.sh})}}
function ring(L,x,y,c,r1,s,l){part(L,{x,y,k:'ring',c,r0:3,r1:r1||40,s:s||3,l:l||.4})}
function glow(L,x,y,c,s,l,a){part(L,{x,y,k:'glow',c,s,l:l||.3,a:a||1})}
function homing(x,y,tx,ty,kind,col,cb){part(PU,{x,y,vx:rr(-90,90),vy:rr(-170,-60),g:300,dr:3,k:'home',l:4,tx,ty,dl:rr(.18,.32),kind,c:col,cb,s:2})}
function updParts(L,dt){for(let i=L.length-1;i>=0;i--){const p=L[i];p.t+=dt;if(p.t>=p.l){L.splice(i,1);continue}
 if(p.k==='home'&&p.t>p.dl){const dx=p.tx-p.x,dy=p.ty-p.y,d=Math.hypot(dx,dy),sp=260+1700*(p.t-p.dl);if(d<sp*dt+6){if(p.cb)p.cb();L.splice(i,1);continue}p.x+=dx/d*sp*dt;p.y+=dy/d*sp*dt;continue}
 p.vy+=p.g*dt;const d=Math.exp(-p.dr*dt);p.vx*=d;p.vy*=d;p.x+=p.vx*dt;p.y+=p.vy*dt}}
function drawParts(L){for(const p of L){const k=1-p.t/p.l;
 if(p.k==='sq'){cx.globalAlpha=Math.min(1,k*2)*p.a;cx.fillStyle=p.c;const s=p.sh?p.s*k:p.s;cx.fillRect(p.x-s/2,p.y-s/2,s,s)}
 else if(p.k==='spk'){cx.globalCompositeOperation='lighter';cx.globalAlpha=Math.min(1,k*1.6);cx.strokeStyle=p.c;cx.lineWidth=p.s;cx.beginPath();cx.moveTo(p.x,p.y);cx.lineTo(p.x-p.vx*.03,p.y-p.vy*.03);cx.stroke();cx.globalCompositeOperation='source-over'}
 else if(p.k==='ring'){cx.globalAlpha=k*.85;cx.strokeStyle=p.c;cx.lineWidth=Math.max(.5,p.s*k);cx.beginPath();cx.arc(p.x,p.y,p.r0+(p.r1-p.r0)*eOut(1-k),0,TAU);cx.stroke()}
 else if(p.k==='glow'){cx.globalCompositeOperation='lighter';cx.globalAlpha=k*p.a;const g=cx.createRadialGradient(p.x,p.y,0,p.x,p.y,p.s);g.addColorStop(0,p.c);g.addColorStop(1,'rgba(0,0,0,0)');cx.fillStyle=g;cx.fillRect(p.x-p.s,p.y-p.s,p.s*2,p.s*2);cx.globalCompositeOperation='source-over'}
 else if(p.k==='smoke'){cx.globalAlpha=k*.35;cx.fillStyle=p.c;const r=p.s*(1+(1-k)*2.2);cx.fillRect(p.x-r,p.y-r,r*2,r*2)}
 else if(p.k==='home'){cx.globalAlpha=1;if(p.kind==='coin'){const w=Math.abs(Math.cos(p.t*14))*6+1;cx.fillStyle='#8a5a1a';cx.fillRect(p.x-w/2-1,p.y-4,w+2,8);cx.fillStyle='#f2c454';cx.fillRect(p.x-w/2,p.y-3,w,6)}else{cx.fillStyle='#140e10';cx.fillRect(p.x-4,p.y-4,8,8);cx.fillStyle=p.c;cx.fillRect(p.x-3,p.y-3,6,6);cx.fillStyle='#fff';cx.fillRect(p.x-2,p.y-2,2,2)}}}
 cx.globalAlpha=1}

// ===== Audio =====
const AU={c:null,ok:false,
 init(){if(this.c)return;try{const A=window.AudioContext||window.webkitAudioContext;if(!A)return;const c=this.c=new A();const comp=c.createDynamicsCompressor();comp.threshold.value=-12;comp.ratio.value=4;comp.connect(c.destination);this.m=c.createGain();this.m.gain.value=.9;this.m.connect(comp);
  this.sg=c.createGain();this.sg.connect(this.m);const lp=c.createBiquadFilter();lp.type='lowpass';lp.frequency.value=3200;lp.connect(this.m);this.mg=c.createGain();this.mg.connect(lp);
  const b=c.createBuffer(1,c.sampleRate,c.sampleRate),d=b.getChannelData(0);for(let i=0;i<d.length;i++)d[i]=R()*2-1;this.nb=b;this.ok=true;this.vol();Music.start()}catch(e){this.ok=false}},
 vol(){if(!this.ok)return;this.sg.gain.value=S.set.sfx?.6:0;this.mg.gain.setTargetAtTime(S.set.mus?(this.dk?0:.24):0,this.c.currentTime,.08)},
 duck(sec){if(!this.ok)return;this.dk=1;this.vol();setTimeout(()=>{this.dk=0;this.vol()},sec*1000)},
 o(f,d,o){if(!this.ok)return;o=o||{};const c=this.c,t=c.currentTime+(o.at||0),g=c.createGain(),s=c.createOscillator();s.type=o.type||'sine';s.frequency.setValueAtTime(f,t);if(o.to)s.frequency.exponentialRampToValueAtTime(o.to,t+d);const v=o.v||.2;g.gain.setValueAtTime(0,t);g.gain.linearRampToValueAtTime(v,t+(o.a||.004));g.gain.exponentialRampToValueAtTime(.0001,t+d);s.connect(g);g.connect(o.dest||this.sg);s.start(t);s.stop(t+d+.03)},
 n(d,o){if(!this.ok)return;o=o||{};const c=this.c,t=c.currentTime+(o.at||0),s=c.createBufferSource();s.buffer=this.nb;const f=c.createBiquadFilter();f.type=o.ft||'bandpass';f.frequency.setValueAtTime(o.f||2000,t);if(o.fto)f.frequency.exponentialRampToValueAtTime(o.fto,t+d);f.Q.value=o.q||1;const g=c.createGain(),v=o.v||.2;g.gain.setValueAtTime(0,t);g.gain.linearRampToValueAtTime(v,t+(o.a||.003));g.gain.exponentialRampToValueAtTime(.0001,t+d);s.connect(f);f.connect(g);g.connect(o.dest||this.sg);s.start(t,R()*.5);s.stop(t+d+.03)},
 lastT:{},thr(k,ms){const n=performance.now();if(this.lastT[k]&&n-this.lastT[k]<ms)return true;this.lastT[k]=n;return false},
 clang(q,p){p=p||1;const f=(q>=1?620:q>.5?540:430)*p*rr(.98,1.02);[[1,.22,.9],[2.76,.12,.5],[5.4,.07,.3],[8.93,.04,.18]].forEach(a=>this.o(f*a[0],a[2]*(.6+q*.6),{v:a[1]}));this.n(.05,{f:4200,v:.28,q:.8});this.o(130,.14,{to:55,v:.32});if(q>=1){this.o(3300,.55,{v:.05});this.o(4400,.4,{v:.03,at:.02})}},
 blips(n,gap){if(!this.ok||this.thr('blip',60))return;const sc=[0,2,4,7,9];for(let i=0;i<Math.min(n,14);i++){const st=sc[i%5]+12*Math.floor(i/5);this.o(880*Math.pow(2,st/12),.05,{type:'square',v:.035,at:i*gap})}},
 tap(){this.o(560,.05,{type:'triangle',v:.08})},
 whoosh(k){this.n(.16+k*.06,{f:700,fto:2600,v:.14+k*.04,q:.7})},
 swing(){if(this.thr('sw',70))return;this.n(.1,{f:900,fto:2200,v:.07,q:.8})},
 hit(b){if(this.thr('hit',40))return;this.o(b?150:190,.13,{to:60,v:b?.4:.28});this.n(.06,{f:b?900:1400,v:b?.3:.2})},
 hurt(){if(this.thr('hu',90))return;this.o(240,.16,{to:110,type:'triangle',v:.2});this.n(.08,{f:600,v:.18})},
 coin(){if(this.thr('coin',45))return;this.o(1568,.07,{type:'triangle',v:.07});this.o(2093,.14,{type:'triangle',v:.07,at:.05})},
 ore(){if(this.thr('ore',60))return;this.o(660,.09,{type:'triangle',v:.1});this.o(990,.12,{type:'triangle',v:.08,at:.04})},
 pop(){if(this.thr('pop',50))return;this.o(320,.1,{to:900,v:.12});this.n(.05,{f:3000,v:.1})},
 rock(h){this.o(300+h*90,.08,{type:'triangle',v:.16});this.n(.06,{f:1800+h*500,v:.24})},
 hiss(){this.n(1,{ft:'highpass',f:2500,fto:6000,v:.16,a:.02});for(let i=0;i<6;i++)this.o(rr(500,900),.05,{v:.05,at:.1+i*.09,to:1200})},
 grind(){if(this.thr('gr',70))return;this.n(.1,{ft:'highpass',f:rr(3200,5200),v:.09});this.o(rr(1800,2600),.06,{type:'square',v:.012})},
 pour(){if(this.thr('po',90))return;this.o(rr(160,320),.09,{v:.07,to:rr(300,500)});this.n(.08,{ft:'lowpass',f:500,v:.08})},
 creak(v){if(this.thr('cr',110))return;this.o(140+v*360,.09,{type:'triangle',v:.07,to:170+v*380})},
 snap(){this.n(.14,{f:2200,v:.32});this.o(900,.18,{type:'triangle',to:180,v:.2})},
 gem(){[1568,2093,2637].forEach((f,i)=>this.o(f,.35,{v:.07,at:i*.06}))},
 crack(){this.n(.2,{ft:'highpass',f:3000,v:.25});[2400,1800,1300].forEach((f,i)=>this.o(f,.08,{type:'triangle',v:.06,at:i*.04}))},
 click(){this.o(1400,.04,{type:'square',v:.04});this.o(900,.06,{type:'triangle',v:.06,at:.03})},
 bell(g){const f=[523,659,784,988,1319][g];this.o(f,.6,{v:.14});this.o(f*2,.4,{v:.05});this.o(f*3.01,.25,{v:.025})},
 legend(){this.o(55,1.4,{v:.5,to:40});[261.6,329.6,392,493.9,587.3].forEach((f,i)=>{this.o(f,2.2,{type:'sawtooth',v:.04,a:.25});this.o(f*2,1.6,{v:.05,at:.15+i*.08})});[1047,1319,1568,2093].forEach((f,i)=>this.o(f,.7,{v:.07,at:.25+i*.11}));this.n(1.2,{ft:'highpass',f:5000,v:.08,a:.1})},
 thump(at){this.o(90,.25,{to:45,v:.45,at:at||0});this.n(.08,{f:500,v:.2,at:at||0})},
 heart(){this.o(70,.18,{to:40,v:.4});this.o(66,.16,{to:40,v:.3,at:.24})},
 okS(n){[0,4,7,12].forEach((s,i)=>this.o(523*Math.pow(2,(s+Math.min(n,12))/12),.22,{type:'triangle',v:.1,at:i*.06}))},
 fail(){this.o(200,.24,{to:120,type:'triangle',v:.2});this.n(.12,{f:400,v:.2})},
 shatter(){this.n(.7,{ft:'highpass',f:2500,v:.3});for(let i=0;i<8;i++)this.o(rr(1800,4200),.12,{type:'triangle',v:.05,at:i*.03});this.o(80,.6,{to:35,v:.5})},
 roar(){this.o(95,1,{to:50,type:'sawtooth',v:.12,a:.08});this.n(1.1,{ft:'lowpass',f:500,v:.3,a:.1})},
 tele(){this.o(220,.9,{to:520,type:'sawtooth',v:.06,a:.1})},
 cut(){this.clang(1,1.3);this.n(.2,{f:3000,fto:800,v:.2})},
 fire(){if(this.thr('fi',150))return;this.n(.25,{ft:'lowpass',f:900,v:.18})},
 ice(){this.n(.3,{ft:'highpass',f:4000,v:.2});this.o(1800,.2,{type:'triangle',v:.06,to:900})},
 lvl(){[0,4,7,12,16].forEach((s,i)=>this.o(659*Math.pow(2,s/12),.16,{type:'square',v:.04,at:i*.05}))},
 boom(){this.o(70,.9,{to:30,v:.6});this.n(.9,{ft:'lowpass',f:900,fto:200,v:.4})}};
const Music={timer:null,step:0,next:0,mode:'title',pat:null,dur:.3,
 start(){if(this.timer)return;this.build();this.timer=setInterval(()=>this.tick(),70)},
 set(m){if(this.mode===m&&this.pat)return;this.mode=m;this.build()},
 build(){const st=Math.min(S?S.stage:0,6),m=this.mode,rs=srand(77+st*131+m.length*17),root=[57,55,53,52,50,54,48][st];
  const maj=m==='forge'||m==='title',sc=maj?[0,2,4,7,9,12,14]:[0,3,5,7,10,12,15];
  this.dur=60/({title:72,forge:80,hunt:116,boss:138}[m])/2;const mel=[];for(let i=0;i<32;i++)mel.push(rs()<(m==='boss'?.62:.5)?sc[Math.floor(rs()*sc.length)]:null);
  this.pat={root,mel,prog:maj?[0,5,-3,-5]:[0,-4,-2,-5]}},
 tick(){if(!AU.ok)return;const c=AU.c;if(this.next<c.currentTime)this.next=c.currentTime+.05;while(this.next<c.currentTime+.3){this.play(this.step,this.next-c.currentTime);this.next+=this.dur;this.step++}},
 play(s,at){const p=this.pat,m=this.mode,bar=Math.floor(s/16)%4,b=p.root+p.prog[bar],mf=n=>440*Math.pow(2,(n-69)/12),dg=AU.mg;
  if(m==='forge'||m==='title'){if(s%8===0)AU.o(mf(b-12),this.dur*6,{type:'triangle',v:.17,at,dest:dg,a:.03});if(s%8===4)AU.o(mf(b-5),this.dur*3,{type:'triangle',v:.1,at,dest:dg,a:.03});const n=p.mel[s%32];if(n!==null&&s%2===0)AU.o(mf(b+12+n),this.dur*3,{type:'triangle',v:.09,at,dest:dg,a:.01})}
  else{if(s%2===0)AU.o(mf(b-12+(s%8===6?7:0)),this.dur*1.7,{type:'triangle',v:.16,at,dest:dg});if(s%4===0)AU.o(150,.18,{to:45,v:m==='boss'?.4:.28,at,dest:dg});if(s%4===2)AU.n(.04,{f:7000,q:.7,v:.05,at,dest:dg});if(m==='boss'&&s%8===4)AU.n(.12,{f:1800,v:.12,at,dest:dg});const n=p.mel[s%32];if(n!==null)AU.o(mf(b+12+n),this.dur*1.5,{type:m==='boss'?'sawtooth':'square',v:m==='boss'?.03:.035,at,dest:dg,a:.005})}}};

// ===== Data =====
const K={G:2.05,eHG:2.2,wAtk:10,eHp:24,eAtk:4.5,eG:2.06,hHp:90,hG:2,gold:4,need:45,needG:5,bossHp:46,bossAtk:3,spCd:4.2,tele:1.05,walk:70,pity:40,cd0:6,cd1:11,aps:1.7,bCost:12,gCost:3,gOre:6,hCost:4,wind:.28};
const GR=[{n:'일반',c:'#cfc7bb',m:1},{n:'고급',c:'#8bd67f',m:1.3},{n:'희귀',c:'#6fb2ff',m:1.7},{n:'영웅',c:'#cf97ff',m:2.3},{n:'전설',c:'#ffd257',m:3.2}];
const SHP={long:{n:'장검',a:1,s:1,r:32,p:1,kb:70,cc:0,cm:0,pose:'sword',hv:'sword',d:'고르게 좋다'},great:{n:'대검',a:1.6,s:.68,r:36,p:2,kb:150,cc:0,cm:0,pose:'axe',hv:'axe',d:'느리지만 둘을 함께 벤다'},
 rapier:{n:'세검',a:.6,s:1.75,r:34,p:1,kb:30,cc:.08,cm:0,pose:'dagger',hv:'dagger',d:'빠르다 · 치명 확률 +8%'},single:{n:'외날검',a:.92,s:1.12,r:32,p:1,kb:60,cc:0,cm:.5,pose:'sword',hv:'spear',d:'치명 피해 +50%'}};
const SHPK=['long','great','rapier','single'];
const HAM={n:'대장 망치',a:.55,s:1,r:28,p:1,kb:90,cc:0,cm:0,pose:'axe',hv:'hammer'};
const NOUN={long:['검','장검','기사검'],great:['대검','큰칼','양손검'],rapier:['세검','찌르개','바늘검'],single:['외날검','칼','곡도']};
const ADJ=[['무딘','투박한','평범한'],['단단한','잘 벼린','곧은'],['날 선','묵직한','푸른 결의'],['이름난','명장의','흠 없는']];
const LEG=['새벽날','달그림자','별가르개','고요','붉은 맹세','마지막 숨','긴 밤','하늘꿰기','천둥니','북극성'];
const TR=[{k:'crit',n:'치명',d:v=>'치명타 확률 +'+v+'%',v:[0,0,10,14,18]},{k:'multi',n:'연타',d:v=>v+'% 확률로 한 번 더',v:[0,0,12,16,20]},{k:'drain',n:'흡수',d:v=>'준 피해의 '+v+'% 회복',v:[0,0,3,5,7]},{k:'ore',n:'광부',d:v=>'재료 +'+v+'%',v:[0,0,30,45,60]},{k:'gold',n:'행운',d:v=>'금화 +'+v+'%',v:[0,0,30,45,60]}];
const EP=[100,100,95,90,85,75,65,55,45,40,35,30,25,20,15,12,10,8,6,5];
const EB=n=>n<8?0:n<10?15:n<12?25:35;
const MAXP={bl:20,gd:10,hd:10},PNAME={bl:'날',gd:'가드',hd:'손잡이'},ENHN={bl:'벼리기',gd:'보석 박기',hd:'조이기'};
const MATS=[['호박','#f2a33a','들쥐 가죽','#8a6446'],['비취','#4fc07a','늑대 가죽','#7a7f88'],['월장석','#d6e4ff','박쥐 가죽','#5a3f6e'],['석류석','#d0303e','흑단','#4a3434'],['얼음수정','#8ff0ff','서리 가죽','#bcd8e4'],['용의 눈','#ff8a2a','용 비늘','#8a3a5a'],['별조각','#b8a8ff','그림자 비단','#2e2a58']];
const ST=[
 {n:'구리 언덕',ore:'구리',oc:'#d98c55',sky:['#2a2140','#7a4a5c','#e9a26c'],far:'#5b3e52',mid:'#62743f',gr:'#8fa44c',dirt:'#5e4632',deco:'grass',amb:'pollen',
  en:[{n:'젤리',a:'slime',c:'#a6d86e'},{n:'들쥐',a:'beast',c:'#9b7556',k:.85}],boss:{n:'왕젤리',a:'slime',c:'#8fd35a',k:2.5,crown:1}},
 {n:'이끼 숲',ore:'철',oc:'#a9b3bf',sky:['#0f1f1b','#22403a','#55806a'],far:'#1d3a30',mid:'#2b5236',gr:'#4f803e',dirt:'#3b3226',deco:'tree',amb:'leaf',
  en:[{n:'이끼 덩이',a:'slime',c:'#5f9a4c',spots:'#cbdc8c'},{n:'늑대',a:'beast',c:'#7d848c'}],boss:{n:'회색 갈기',a:'beast',c:'#6e747c',k:2,mane:'#d2d5da',scar:1}},
 {n:'은빛 협곡',ore:'은',oc:'#e6edf4',sky:['#141a2c','#33405e','#8d9fc2'],far:'#3a4766',mid:'#555e78',gr:'#868ea2',dirt:'#3f4252',deco:'rock',amb:'dust',
  en:[{n:'해골 병사',a:'human',c:'#e8e2d0',bone:1,wpn:'#9aa3ad'},{n:'박쥐',a:'flyer',c:'#5a3f6e',bat:1}],boss:{n:'은갑 기사',a:'human',c:'#d9e1ea',k:2,armor:1,plume:'#c0392b',wpn:'#eef3f8'}},
 {n:'붉은 갱도',ore:'흑철',oc:'#6a5c7a',sky:['#120808','#3a1010','#8a2a14'],far:'#2a0f0c',mid:'#45160f',gr:'#5a2a22',dirt:'#1e1210',deco:'lava',amb:'ember',
  en:[{n:'불씨',a:'flyer',c:'#ff8a3a',wisp:1},{n:'용암 덩이',a:'golem',c:'#4a3a38',glow:'#ff7a2a'}],boss:{n:'화염 골렘',a:'golem',c:'#3e302e',k:2.1,glow:'#ffb03a'}},
 {n:'서리 동굴',ore:'미스릴',oc:'#7fe3ea',sky:['#081420','#123048','#3c7a9a'],far:'#16405a',mid:'#2b6a86',gr:'#cfeef6',dirt:'#4d7c92',deco:'ice',amb:'snow',
  en:[{n:'서리 늑대',a:'beast',c:'#cfe8f2',spikes:'#7fd3ea',eye:'!#2aa7c9'},{n:'서리 유령',a:'flyer',c:'#bde6ff',ghost:1}],boss:{n:'서리 거인',a:'golem',c:'#9fd0e4',k:2.2,glow:'#e8ffff',ice:1}},
 {n:'용의 둥지',ore:'용철',oc:'#e2654c',sky:['#1a0a14','#4a1630','#c2584a'],far:'#3a1424',mid:'#5c2232',gr:'#7a4a36',dirt:'#3a2018',deco:'bone',amb:'ash',
  en:[{n:'용인 전사',a:'human',c:'#7a3f8a',scale:1,horn:1,wpn:'#e0c27a'},{n:'새끼 용',a:'flyer',c:'#c0503e',drake:1}],boss:{n:'늙은 용',a:'dragon',c:'#8a3a5a',belly:'#e8c070'}},
 {n:'별무덤',ore:'운철',oc:'#8fa0ff',sky:['#05040e','#120c2a','#2c2466'],far:'#1a1640',mid:'#241d50',gr:'#3d3478',dirt:'#14102c',deco:'star',amb:'star',
  en:[{n:'별 그림자',a:'human',c:'#1e1a3c',shade:1,eye:'#bfc8ff'},{n:'운석 덩이',a:'golem',c:'#2e2a48',glow:'#9fb0ff'}],boss:{n:'검은 별',a:'orb',c:'#191433',glow:'#c8b0ff'}}];
const ARCH={slime:{h:1,a:.8,s:34,i:1.3},beast:{h:.8,a:1,s:62,i:1.05},human:{h:1.2,a:1.1,s:39,i:1.2},flyer:{h:.7,a:.9,s:52,i:1.1,fly:1},golem:{h:1.6,a:1.3,s:24,i:1.5},dragon:{h:1,a:1,s:34,i:1.35},orb:{h:1,a:1,s:29,i:1.3,fly:1}};
const BMOV=[['slam','split'],['charge','slam'],['shield','slam'],['slam','slam'],['freeze','slam'],['breath','slam'],['laser','shards']];
const BPAS=[0,0,0,'heat',0,0,0];
const MVN={slam:'내려찍기',split:'분열',charge:'돌진',shield:'방패',freeze:'얼리기',breath:'불 숨결',laser:'눈빛',shards:'파편 방패',heat:'달아오름'};
const MVH={slam:['빛날 때 강타로 끊자!',0],split:['휩쓸기로 쓸어버리자!',1],charge:['부딪히기 직전 강타!',0],shield:['강타로 방패를 깨자!',0],freeze:['빛날 때 강타! 얼면 주인공 연타',0],breath:['숨 들이쉴 때 강타!',0],laser:['빛날 때 강타로 끊자!',0],shards:['휩쓸기로 파편을 깨자!',1],heat:['휩쓸기로 식히자!',1]};
function stageDef(i){if(i<ST.length)return ST[i];const b=ST[ST.length-1],n=i-ST.length+2;return Object.assign({},b,{n:b.n+' '+n+'층',ore:b.ore+' '+n})}
const oreName=t=>stageDef(t).ore;
const oreCol=t=>t<ST.length?ST[t].oc:mix('#8fa0ff','#ffffff',Math.min(.6,(t-6)*.12));
const matName=(k,t)=>{if(k==='o')return oreName(t);const n=MATS[Math.min(t,6)][k==='g'?0:2];return t>6?n+' '+(t-5):n};
const matCol=(k,t)=>k==='o'?oreCol(t):MATS[Math.min(t,6)][k==='g'?1:3];
const enhMul=p=>1+.12*p+.006*p*p;
const xpNeed=lv=>Math.round(15*Math.pow(1.28,lv-1));
const needN=()=>Math.min(80,K.need+K.needG*S.stage);
const hammerA=()=>K.wAtk*Math.pow(K.G,S.stage)*HAM.a;
const hGold=t=>Math.round(20*Math.pow(K.G,t));

// ===== State =====
const SKEY='moru_save_v1';
let S=null;
function newState(){return{v:2,gold:30,mat:{o:[18],g:[3],h:[4]},stage:0,bl:null,gd:null,hd:null,lv:1,xp:0,prog:0,pity:0,charm:0,daily:null,tut:{},set:{sfx:1,mus:1,shk:1,spd:1},st:{forge:0,leg:0,kills:0}}}
function load(){try{const s=JSON.parse(localStorage.getItem(SKEY)||'null');if(s&&s.v===2){const n=newState();for(const k in s)n[k]=s[k];n.set=Object.assign(newState().set,s.set||{});n.tut=s.tut||{};n.st=Object.assign(newState().st,s.st||{});n.mat=Object.assign({o:[0],g:[0],h:[0]},s.mat||{});return n}}catch(e){}
 try{const o=JSON.parse(localStorage.getItem('moru_save_v0')||'null');if(o&&o.v===1){const n=newState(),m={sword:'long',axe:'great',dagger:'rapier',spear:'single'};n.gold=o.gold||0;n.mat.o=o.ore||[0];n.stage=o.stage||0;n.lv=o.lv||1;n.xp=o.xp||0;n.prog=o.prog||0;n.pity=o.pity||0;n.charm=o.charm||0;n.st=Object.assign(n.st,o.st||{});n.set=Object.assign(n.set,o.set||{});
  n.mat.g=[];n.mat.h=[];n.mat.g[n.stage]=3;n.mat.h[n.stage]=4;if(o.w)n.bl={t:o.w.t,g:o.w.g,sh:m[o.w.ty]||'long',a:o.w.a,q:1,p:o.w.p,tr:o.w.tr||[],name:o.w.name};return n}}catch(e){}return newState()}
function save(){try{localStorage.setItem(SKEY,JSON.stringify(S))}catch(e){}}
const matN=(k,t)=>S.mat[k][t]||0;
function addMat(k,t,n){const a=S.mat[k];while(a.length<=t)a.push(0);a[t]=(a[t]||0)+n}
function trait(k){if(!S.bl)return 0;let v=0;for(const t of S.bl.tr)if(t.k===k)v+=t.v;return v}
const shapeOf=b=>b?SHP[b.sh]:HAM;
const gdStat=g=>g?{cc:(.02*(g.t+1)+.02*g.p)*g.q,cm:.1+.05*g.t+.06*g.p}:{cc:0,cm:0};
const hdStat=h=>h?(.05*(h.t+1)+.035*h.p)*h.q:0;
const bladeAtk=b=>b?b.a*b.q*enhMul(b.p):hammerA();
function stats(b,g,h){if(b===undefined)b=S.bl;if(g===undefined)g=S.gd;if(h===undefined)h=S.hd;const sh=shapeOf(b),gs=gdStat(g);let tc=0;if(b)for(const t of b.tr)if(t.k==='crit')tc+=t.v;
 const cc=Math.min(.6,.05+gs.cc+sh.cc+tc/100),cm=1.5+gs.cm+sh.cm,spd=sh.s*(1+hdStat(h));return{atk:bladeAtk(b),cc,cm,spd,aps:K.aps*spd,sh}}
function power(b,g,h){const s=stats(b,g,h);if(b===undefined)b=S.bl;let p=s.atk*s.aps*(1+s.cc*(s.cm-1))*(s.sh.p>1?1.25:1);if(b)for(const t of b.tr){if(t.k==='multi')p*=1+t.v/125;if(t.k==='drain')p*=1+t.v/50}return Math.round(p)}
function heroAtk(){return bladeAtk(S.bl)*(1+.04*(S.lv-1))}
function heroMaxHp(){return Math.round(K.hHp*Math.pow(K.hG,S.stage)*(1+.06*(S.lv-1)))}
function gainXp(n){S.xp+=n;let up=0;while(S.xp>=xpNeed(S.lv)){S.xp-=xpNeed(S.lv);S.lv++;up=1}if(up){if(Hn.on){Hn.mhp=heroMaxHp();Hn.hp=Hn.mhp;ptext(TW,'레벨 업!',Hn.x+4,L.gy-132,{size:14,bold:1,color:'#ffe08a',pop:1,wave:1.5,l:1.4,say:1});burst(PW,Hn.x,L.gy-52,24,{c:['#ffe08a','#fff6c8'],v0:40,v1:140,g:-60,s:3,sh:1})}AU.lvl()}}
// crafting logic
function canCraft(k,t){if(k==='bl')return matN('o',t)>=K.bCost;if(k==='gd')return matN('g',t)>=K.gCost&&matN('o',t)>=K.gOre;return matN('h',t)>=K.hCost&&S.gold>=hGold(t)}
function craftTier(k){const n=Math.max(S.mat.o.length,S.mat.g.length,S.mat.h.length);for(let t=n-1;t>=0;t--)if(canCraft(k,t))return t;return -1}
function payCraft(k,t){if(k==='bl')S.mat.o[t]-=K.bCost;else if(k==='gd'){S.mat.g[t]-=K.gCost;S.mat.o[t]-=K.gOre}else{S.mat.h[t]-=K.hCost;S.gold-=hGold(t)}S.st.forge++;qprog('forge')}
function gradeW(Q,pf,charm){const base=[55,28,12,4.2,.8],f=.3+2.2*Q,w=base.map((b,g)=>b*Math.pow(f,g*.7));if(pf>=5)w[4]*=1.5;if(charm)w[4]*=3;if(pf>=5){w[0]=0;w[1]=0}else if(pf>=4)w[0]=0;return w}
function gradeOdds(Q,pf,charm){if(S.pity>=K.pity)return[0,0,0,0,1];const w=gradeW(Q,pf,charm);let s=0;for(const v of w)s+=v;return w.map(v=>v/s)}
function rollGrade(Q,pf,charm){const o=gradeOdds(Q,pf,charm);let r=R();for(let g=0;g<5;g++){if((r-=o[g])<0)return g}return 0}
function effQ(acc){if(!acc.length)return .5;let a=acc.reduce((x,y)=>x+y,0)/acc.length,st=0,mx=0;for(const q of acc){st=q>=1?st+1:0;mx=Math.max(mx,st)}if(mx>=3)a+=.05*(mx-2);return Math.min(1.15,a)}
const perfN=acc=>acc.filter(q=>q>=1).length;
function mkBlade(t,sh,g,q){const a=Math.round(K.wAtk*Math.pow(K.G,t)*GR[g].m*SHP[sh].a*rr(.95,1.05)),tr=[],pool=TR.slice(),nt=g>=4?2:g>=2?1:0;for(let i=0;i<nt;i++){const T=pool.splice(Math.floor(R()*pool.length),1)[0];tr.push({k:T.k,v:T.v[g]})}
 return{t,g,sh,a,q,p:0,tr,name:g===4?pick(LEG):pick(ADJ[g])+' '+oreName(t)+' '+pick(NOUN[sh])}}
const mkGuard=(t,q)=>({t,q,p:0,name:matName('g',t)+' 가드'});
const mkHandle=(t,q)=>({t,q,p:0,name:matName('h',t)+' 손잡이'});
const qWord=q=>q>=1.12?'완벽':q>=1.05?'훌륭함':q>=.95?'좋음':q>=.86?'보통':'거침';
function meltV(k,p){const b=Math.pow(K.G,p.t);return Math.round(k==='bl'?15*b*GR[p.g].m*(1+.4*p.p):k==='gd'?20*b*(1+.5*p.p):12*b*(1+.5*p.p))}
// enhance logic
function enhCost(k){const p=S[k];if(!p)return null;const b=Math.pow(K.G,p.t);if(k==='bl')return{g:Math.round(8*b*Math.pow(p.p+1,1.5))};if(k==='gd')return{g:Math.round(6*b*(p.p+1)),m:'g',n:1,t:p.t};return{g:Math.round(5*b*(p.p+1)),m:'h',n:1,t:p.t}}
function canEnh(k){const p=S[k],c=enhCost(k);if(!p||p.p>=MAXP[k])return false;return S.gold>=c.g&&(!c.m||matN(c.m,c.t)>=c.n)}
function payEnh(k){const c=enhCost(k);S.gold-=c.g;if(c.m)S.mat[c.m][c.t]-=c.n}
const enhChance=q=>clamp(EP[S.bl.p]+(q>=1?15:q>.5?5:-5),3,100);
const enhBreak=q=>EB(S.bl.p)*(q>=1?.5:1);
const gWin=p=>({perf:.1,ok:clamp(.42-.026*p,.15,.42)});
const hZone=p=>clamp(.24-.013*p,.09,.24);
function doEnh(k,q){const p=S[k];if(k==='bl'){if(R()*100<enhChance(q)){p.p++;qprog('enh');return'ok'}if(R()*100<enhBreak(q)){S.bl=null;return'br'}return'fail'}
 if(q>0){p.p++;if(q>=1&&R()<.3&&p.p<MAXP[k])p.p++;qprog('enh');if(k==='gd')qprog('gem');return'ok'}return'fail'}
// daily
const QP=[{k:'kill',t:60,d:n=>'적 '+n+'마리 처치'},{k:'vein',t:12,d:n=>'광맥 '+n+'개 캐기'},{k:'perfect',t:10,d:n=>'완벽 타격 '+n+'번'},{k:'forge',t:3,d:n=>'부품 '+n+'개 만들기'},{k:'enh',t:5,d:n=>'강화 성공 '+n+'번'},{k:'rare',t:1,d:()=>'희귀 이상 날 만들기'},{k:'skill',t:15,d:n=>'강타 '+n+'번 쓰기'},{k:'gem',t:2,d:n=>'보석 박기 성공 '+n+'번'}];
function today(){const d=new Date();return d.getFullYear()+'-'+(d.getMonth()+1)+'-'+d.getDate()}
function ensureDaily(){const td=today();if(S.daily&&S.daily.date===td)return;const rs=srand(hash(td)),pool=QP.slice(),q=[];while(q.length<3){const i=Math.floor(rs()*pool.length);q.push({k:pool[i].k,t:pool[i].t,p:0,c:0});pool.splice(i,1)}S.daily={date:td,q,b:0}}
function qprog(k,n){if(!S.daily)return;for(const q of S.daily.q)if(q.k===k&&q.p<q.t){q.p=Math.min(q.t,q.p+(n||1));if(q.p>=q.t)toast('의뢰 완료: '+QP.find(x=>x.k===k).d(q.t),'#ffe08a')}}
const qReward=()=>({g:Math.round(40*Math.pow(K.G,S.stage)),o:5,gm:1});
function claimQ(i){const q=S.daily.q[i];if(q.c||q.p<q.t)return;q.c=1;const r=qReward();S.gold+=r.g;addMat('o',S.stage,r.o);addMat('g',S.stage,r.gm);UI.bump.g=1;UI.bump.o=1;AU.coin();if(S.daily.q.every(x=>x.c)&&!S.daily.b){S.daily.b=1;S.charm++;toast('행운 숯을 얻었다','#ffd257')}save()}
const qReady=()=>S.daily&&S.daily.q.some(q=>q.p>=q.t&&!q.c);

// ===== Sprites =====
const SPR={};
const HPAL=[
 {body:'#8a5a3a',trim:'#c9a26a',legs:'#4a3a30',boot:'#2e2420'},
 {body:'#6a4630',trim:'#b08858',legs:'#3e3028',boot:'#2a1e18',pad:'#8a6040'},
 {body:'#8e979f',trim:'#5c646c',legs:'#4a4e56',boot:'#2a2a30',pad:'#a8b0b8'},
 {body:'#bcc6d0',trim:'#7a8590',legs:'#5a6470',boot:'#30343c',pad:'#d6dee6',helm:'#bcc6d0'},
 {body:'#d8e6ee',trim:'#5fd3e0',legs:'#5a7a8a',boot:'#2c3e4a',pad:'#eef6fa',helm:'#d8e6ee',crest:'#5fd3e0'},
 {body:'#a8342a',trim:'#e8b64a',legs:'#5a2420',boot:'#2a1210',pad:'#e8b64a',helm:'#a8342a',horn:'#efe0c0'},
 {body:'#24245a',trim:'#9fb0ff',legs:'#1a1840',boot:'#100e24',pad:'#3a3a80',helm:'#24245a',crest:'#c8d0ff',star:1}];
const SCARF=['#c0392b','#c0392b','#b8452e','#2e6ab0','#5fd3e0','#e8b64a','#c8b0ff'];
function genHero(tier,f){const p=HPAL[Math.min(tier,6)],g=Grid(15,24),oy=3,b=(f===1||f===3)?-1:0,lf=[[1,-1],[0,0],[-1,1],[0,0]][f],skin='#efc39a',hair='#3b2418';
 const L0=oy+14+b;gR(g,6+lf[1],L0,2,oy+19-L0,shade(p.legs,-.2));gR(g,6+lf[1],oy+19,3,2,shade(p.boot,-.15));gR(g,8+lf[0],L0,2,oy+19-L0,p.legs);gR(g,8+lf[0],oy+19,3,2,p.boot);
 const T0=oy+7+b;gR(g,4,T0,6,7,p.body);gR(g,5,T0,4,1,p.trim);gR(g,4,T0+5,6,1,'#3a2a20');gp(g,7,T0+5,'!#e8b64a');if(p.pad)gR(g,3,T0,3,3,p.pad);if(p.star){gp(g,6,T0+2,'!#dfe6ff');gp(g,8,T0+3,'!#9fb0ff')}
 const H0=oy+1+b;gR(g,5,H0,6,6,skin);gR(g,4,H0,6,2,hair);gR(g,4,H0+2,2,3,hair);gp(g,9,H0+3,'!#2a1a14');gp(g,10,H0+5,shade(skin,-.18));
 if(p.helm){gR(g,4,H0-1,7,3,p.helm);gR(g,4,H0+2,2,3,p.helm);gp(g,10,H0+1,shade(p.helm,-.3))}
 if(p.crest){gR(g,5,H0-3,4,2,p.crest);gp(g,4,H0-2,p.crest)}
 if(p.horn){gL(g,5,H0-1,3,H0-3,p.horn);gL(g,9,H0-1,11,H0-3,p.horn)}
 return bake(g)}
// detailed sword (tip up)
const SHD={long:{L:34,w:5,tip:5,hl:12,gx:6},great:{L:37,w:8,tip:6,hl:14,gx:6},rapier:{L:38,w:3,tip:7,hl:12,gx:6},single:{L:34,w:5,tip:6,hl:12,gx:5}};
function gMetal(gd){return gd?mix(shade(oreCol(gd.t),-.1),'#e8b64a',.35):'#6a6e78'}
function genSwordBig(b,gd,hd,only){const sh=b?b.sh:'long',D=SHD[sh],gr=b?b.g:0,pt=b?Math.min(4,Math.floor(b.p/5)):0;
 const gW=gd?D.w+D.gx*2:D.w+6,Wd=Math.max(gW,D.w+4)+2,cm=Math.floor(Wd/2),Hh=D.L+4+D.hl+3,g=Grid(Wd,Hh);
 const sB=!only||only==='bl',sG=!only||only==='gd',sH=!only||only==='hd';
 if(b&&sB){const oc=oreCol(b.t),core=pt>=3?mix(oc,'#ffffff',.2):oc,eg='!'+shade(oc,(b.q>=1.05||pt>=1)?.62:.38),sp=shade(oc,-.32),fl=shade(core,-.24);
  for(let y=0;y<D.L;y++){const tk=y<D.tip?(y+1)/D.tip:1,hw=Math.max(.5,D.w/2*tk),off=sh==='single'?Math.round(Math.pow(1-y/D.L,2)*2):0,x0=Math.round(cm-hw+off),x1=Math.max(x0,Math.round(cm+hw+off)-1);
   for(let x=x0;x<=x1;x++)gp(g,x,y,core);if(sh==='single'){gp(g,x0,y,sp);gp(g,x1,y,eg)}else{gp(g,x0,y,eg);gp(g,x1,y,eg)}
   if(sh!=='rapier'&&y>D.tip+1&&y<D.L-2){const fx=cm+off-(sh==='great'?1:0);gp(g,fx,y,fl);if(sh==='great')gp(g,fx+1,y,fl);
    if(pt>=2&&y%4===0)gp(g,fx,y,'!'+GR[gr].c);else if(gr>=4&&y%4===2)gp(g,fx,y,'!#fff6c8');else if(gr>=2&&y%3===0)gp(g,fx,y,shade(fl,.3))}
   else if(sh==='rapier'&&pt>=2&&y%5===0&&y>D.tip)gp(g,cm,y,'!'+GR[gr].c)}
  if(gr>=3)gR(g,cm-Math.floor(D.w/2),D.L-2,D.w,2,'#e8b64a')}
 const gy=D.L,gm=gMetal(gd),gc=gd?matCol('g',gd.t):null;
 if(sG){const xL=cm-Math.floor(gW/2);for(let x=0;x<gW;x++){gp(g,xL+x,gy+1,gm);gp(g,xL+x,gy+2,shade(gm,-.15))}gp(g,xL,gy,gm);gp(g,xL+gW-1,gy,gm);gR(g,cm-2,gy,5,1,gm);gR(g,cm-1,gy+3,3,1,shade(gm,-.2));
  if(gc){gp(g,cm,gy+1,'!'+gc);gp(g,cm,gy+2,'!'+shade(gc,-.3));gp(g,cm-1,gy+1,'!'+shade(gc,-.12));gp(g,cm+1,gy+1,'!'+shade(gc,-.12));gp(g,cm,gy,'!'+shade(gc,.55));const ns=Math.min(4,Math.floor(gd.p/2.5));for(let i=0;i<ns;i++){const ox=(i%2?1:-1)*(3+Math.floor(i/2)*2);gp(g,cm+ox,gy+1,'!'+gc)}}}
 const hy=gy+4,hc=hd?matCol('h',hd.t):'#9a8a7a',hp=hd?hd.p:0;
 if(sH){for(let y=0;y<D.hl;y++){for(let x=-1;x<=1;x++)gp(g,cm+x,hy+y,(y+x+3)%3===0?shade(hc,-(.16+.025*hp)):hc);if(hp>=5&&y%4===1)gp(g,cm,hy+y,'!#e8b64a')}if(hp>=10)for(let y=0;y<D.hl;y+=2){gp(g,cm-1,hy+y,'!#e8b64a');gp(g,cm+1,hy+y,'!#e8b64a')}
  const py=hy+D.hl;gR(g,cm-1,py,3,2,gm);gp(g,cm,py+2,gm);if(gd&&gd.p>=5)gp(g,cm,py,'!'+gc)}
 const c=bake(g);c.pv=[(cm+1.5)/c.width,(hy+D.hl/2+1)/c.height];c.fb=(D.L/2+1)/c.height;c.fg=(gy+2+1)/c.height;c.fh=(hy+D.hl/2+1)/c.height;c.ft=1/c.height;c.fe=(D.L+1)/c.height;return c}
const SSD={long:{L:12,w:3},great:{L:14,w:5},rapier:{L:15,w:1},single:{L:12,w:3}};
function genSwordSmall(b,gd,hd){const D=SSD[b.sh],oc=oreCol(b.t),Wd=D.w+6,cm=Math.floor(Wd/2),g=Grid(Wd,D.L+7),hw=(D.w-1)/2;
 for(let y=0;y<D.L;y++){const k=y<2?(y+1)/3:1;for(let x=Math.round(cm-hw*k);x<=Math.round(cm+hw*k);x++)gp(g,x,y,oc)}
 for(let y=1;y<D.L;y++)gp(g,cm,y,(b.q>=1.05||b.p>=5)?'!'+shade(oc,.6):shade(oc,.35));
 const gm=gMetal(gd);gR(g,cm-hw-2,D.L,D.w+4,1,gm);if(gd)gp(g,cm,D.L,'!'+matCol('g',gd.t));gR(g,cm,D.L+1,1,4,hd?matCol('h',hd.t):'#9a8a7a');gp(g,cm,D.L+5,gm);
 const c=bake(g);c.pv=[(cm+1.5)/c.width,(D.L+3+1)/c.height];return c}
function genHammer(){const g=Grid(9,15);gR(g,0,0,9,5,'#7a7f88');gR(g,0,0,9,1,'#9aa0a8');gR(g,4,5,1,10,'#6a4428');gR(g,3,5,3,1,'#4a4e56');const c=bake(g);c.pv=[5.5/c.width,13.5/c.height];return c}
const SWC={};
function swKey(b,g,h,only){return(only||'')+'|'+(b?b.sh+b.t+'.'+b.g+'.'+Math.min(4,b.p/5|0)+'.'+(b.q>=1.05?1:0):'-')+'|'+(g?g.t+'.'+Math.min(4,g.p/2.5|0)+'.'+(g.p>=5?1:0):'-')+'|'+(h?h.t+'.'+h.p:'-')}
function swBig(b,g,h,only){const k='B'+swKey(b,g,h,only);return SWC[k]||(SWC[k]=genSwordBig(b,g,h,only))}
function swSmall(b,g,h){if(!b)return SPR.ham;const k='S'+swKey(b,g,h);return SWC[k]||(SWC[k]=genSwordSmall(b,g,h))}
// enemies
function genSlime(o,f){const k=o.k||1,top=o.crown?4*k:0,w=Math.round(17*k),h=Math.round(14*k+top),g=Grid(w,h),c=o.c,m=w/2,cy=top+8.4*k,rx=7.6*k*(f?1.04:1),ry=5.6*k*(f?.94:1);
 for(let y=0;y<h;y++)for(let x=0;x<w;x++){const dx=(x+.5-m)/rx,dy=(y+.5-cy)/ry;if(dx*dx+dy*dy<=1||(y+.5>=cy&&Math.abs(x+.5-m)<=rx*(1-.12*((y+.5-cy)/(h-cy)))))gp(g,x,y,c)}
 gE(g,m,h-2*k,rx*.6,1.5*k,shade(c,.12));
 if(o.spots){const rs=srand(5);for(let i=0;i<7*k;i++)gp(g,m+(rs()-.5)*rx*1.4,cy+(rs()-.6)*ry*1.1,o.spots)}
 const ew=Math.max(1,Math.round(k)),eh=Math.max(2,Math.round(1.7*k));gR(g,m+1.6*k,cy-1.2*k,ew,eh,'!#1a1214');gR(g,m+4.2*k,cy-1.2*k,ew,eh,'!#1a1214');gp(g,m-3*k,cy-ry*.55,'!#ffffff');
 if(o.crown){const y0=cy-ry;gR(g,m-3.4*k,y0-2*k,6.8*k,2*k,'#e8b64a');for(let i=0;i<3;i++)gR(g,m-3.4*k+i*2.7*k,y0-3.6*k,1.4*k,1.7*k,'#e8b64a');gp(g,m,y0-1*k,'!#c0392b')}
 return bake(g)}
function genBeast(o,f){const k=o.k||1,w=Math.round(25*k),h=Math.round(16*k),g=Grid(w,h),c=o.c,dk=shade(c,-.25),s=v=>v*k,lg=f?1:-1,lw=Math.max(2,Math.round(1.8*k));
 gL(g,s(4),s(7.5),s(1),s(3.5),dk,Math.max(1,Math.round(1.6*k)));
 gR(g,s(6)-lg*s(.9),s(9.5),lw,s(5.5),dk);gR(g,s(14)+lg*s(.9),s(9.5),lw,s(5.5),dk);
 gE(g,s(10.5),s(8),s(7),s(3.7),c);
 gR(g,s(8)+lg*s(.9),s(10),lw,s(5.5),c);gR(g,s(16)-lg*s(.9),s(10),lw,s(5.5),c);
 if(o.mane)gE(g,s(15.5),s(6.5),s(3.6),s(4.4),o.mane);
 gE(g,s(18),s(5.8),s(3.2),s(2.9),c);gR(g,s(20),s(5.4),s(3.8),s(2.2),c);gp(g,s(23.4),s(5.6),'!#1a1214');
 gP(g,[[s(16),s(4.2)],[s(16.8),s(.6)],[s(18.8),s(3.8)]],dk);
 gR(g,s(19.4),s(4.4),Math.max(1,Math.round(k)),Math.max(1,Math.round(k)),o.eye||'!#1a1214');
 if(o.spikes)for(let i=0;i<4;i++)gP(g,[[s(6+i*2.6),s(5)],[s(7.2+i*2.6),s(1.6+(i%2))],[s(8.4+i*2.6),s(5)]],o.spikes);
 if(o.scar)gL(g,s(17.2),s(3.4),s(19.8),s(7.2),'!#ead8c8');
 gR(g,s(7),s(10.8),s(8),Math.max(1,Math.round(k)),shade(c,.15));return bake(g)}
function genHuman(o,f){const k=o.k||1,w=Math.round(17*k),h=Math.round(25*k),g=Grid(w,h),s=v=>v*k,c=o.c,lf=f?1:-1,u=Math.max(1,Math.round(k));
 const skin=o.bone?'#e8e2d0':o.shade?'#16132e':o.scale?shade(c,.12):'#d9b08c',lw=Math.max(2,Math.round(2*k));
 if(o.shade)gP(g,[[s(5),s(9)],[s(1.5),s(23)],[s(9),s(22)]],shade(c,-.25));
 gR(g,s(6)+lf*s(.9),s(16),lw,s(8),o.bone?'#cfc7b2':shade(c,-.3));gR(g,s(9)-lf*s(.9),s(16),lw,s(8),o.bone?'#e8e2d0':shade(c,-.12));
 if(o.bone){for(let y=0;y<7;y+=2)gR(g,s(5),s(9+y),s(6),u,'#e8e2d0');gR(g,s(7.5),s(9),u,s(7),'#d8d0bc')}else{gR(g,s(5),s(9),s(6),s(7),c);if(o.armor){gR(g,s(4),s(9),s(3),s(2.5),shade(c,.2));gR(g,s(5),s(14),s(6),u,'#8a7a5a')}}
 if(o.scale){gR(g,s(6),s(10),s(4),s(5),shade(c,.28));gL(g,s(5),s(15),s(1),s(21),c,Math.max(1,Math.round(1.6*k)))}
 gR(g,s(5.5),s(2.5),s(6),s(6),skin);
 if(o.bone){gR(g,s(9),s(4.5),Math.max(1,Math.round(1.4*k)),Math.max(1,Math.round(1.4*k)),'!#2a1a20');gR(g,s(6),s(7),s(5),u,'!#bfb6a0')}else gR(g,s(9.5),s(4.5),u,u,o.eye?'!'+o.eye:'!#1a1214');
 if(o.armor){gR(g,s(5),s(1.5),s(7),s(4),c);gR(g,s(8.5),s(4),s(3.5),u,'!#1a1214');if(o.plume)gP(g,[[s(7),s(1.6)],[s(5),0],[s(1.5),s(3.5)],[s(5.5),s(3)]],o.plume)}
 if(o.horn){gL(g,s(6),s(2.5),s(4),s(.2),'#efe0c0');gL(g,s(10),s(2.5),s(12),s(.4),'#efe0c0')}
 const wc=o.wpn||'#9aa3ad';
 if(o.scale){gL(g,s(13),s(2),s(13),s(24),'#7a5030',u);gP(g,[[s(13),0],[s(14.6),s(3.5)],[s(11.4),s(3.5)]],wc)}
 else if(!o.shade){gR(g,s(12.6),s(1.5),Math.max(1,Math.round(1.5*k)),s(11),wc);gR(g,s(11),s(12.5),s(4.6),u,'#5a5a62');gR(g,s(12.6),s(13.5),u,s(3),'#5a3a24')}
 gR(g,s(10.5),s(10),s(2.5),s(4),o.bone?'#e8e2d0':o.shade?shade(c,.1):skin);
 if(o.shade)for(let i=0;i<5;i++)gp(g,s(3.5)+i*s(1.6),s(11+i*2.2),'!#dfe6ff');return bake(g)}
function genFlyer(o,f){const k=o.k||1,w=Math.round(20*k),h=Math.round(16*k),g=Grid(w,h),s=v=>v*k,c=o.c,u=Math.max(1,Math.round(k));
 if(o.wisp){gE(g,s(10),s(10.5),s(4.2),s(4.5),c);gP(g,[[s(5.8),s(10)],[s(8+(f?1:0)),s(1+(f?1:0))],[s(10),s(6)],[s(12),s(f?2:0)],[s(14.2),s(10)]],c);gE(g,s(10),s(11),s(2.3),s(2.6),'!#fff2a0');gR(g,s(11),s(9.5),u,Math.max(1,Math.round(1.5*k)),'!#5a1a08');return bake(g,{ol:'#3a1208'})}
 if(o.ghost){gE(g,s(10),s(6.5),s(5),s(5),c);gR(g,s(5),s(6.5),s(10),s(5.5),c);for(let i=0;i<4;i++)gE(g,s(6.2+i*2.6),s(12.2)+((f?i:i+1)%2)*k,s(1.4),s(1.5),c);gR(g,s(11),s(5.5),Math.max(1,Math.round(1.4*k)),Math.max(1,Math.round(2*k)),'!#26364a');gR(g,s(13.4),s(5.5),Math.max(1,Math.round(1.4*k)),Math.max(1,Math.round(2*k)),'!#26364a');return bake(g,{ol:'#2a3a52'})}
 const wy=f?s(13):s(1),wc=shade(c,o.bat?-.25:-.12);
 gP(g,[[s(8.5),s(7)],[s(1),wy],[s(4),s(10)]],wc);gP(g,[[s(11.5),s(7)],[s(19),wy],[s(16),s(10)]],shade(wc,.12));
 gE(g,s(10),s(8),s(3.7),s(3.3),c);
 if(o.bat){gP(g,[[s(8),s(6)],[s(8.4),s(2.6)],[s(9.8),s(5.4)]],c);gP(g,[[s(10.6),s(5.4)],[s(12),s(2.6)],[s(12.4),s(6)]],c);gp(g,s(11.4),s(7.6),'!#ff5a4a');gp(g,s(9),s(7.6),'!#ff5a4a')}
 if(o.drake){gE(g,s(14.5),s(6),s(2.7),s(2.1),c);gR(g,s(16),s(5.8),s(3),s(1.6),c);gp(g,s(15.2),s(5.4),'!#ffd257');gL(g,s(13.4),s(4.2),s(12.2),s(2),'#efe0c0');gL(g,s(7),s(9.5),s(1.5),s(11.5),c,u);gR(g,s(8),s(9.8),s(4),u,shade(c,.3))}
 return bake(g)}
function genGolem(o,f){const k=o.k||1,w=Math.round(22*k),h=Math.round(24*k),g=Grid(w,h),s=v=>v*k,c=o.c,dk=shade(c,-.2),a=f?s(.8):0;
 gR(g,s(6),s(17),s(3.5),s(7),dk);gR(g,s(12),s(17),s(3.5),s(7),c);gR(g,s(4),s(7),s(13),s(11),c);gR(g,s(5),s(6),s(11),s(1),c);gR(g,s(7.5),s(2),s(7),s(5.5),shade(c,.06));
 gR(g,s(11.5),s(4),Math.max(1,Math.round(1.6*k)),Math.max(1,Math.round(1.2*k)),'!'+(o.glow||'#ffd257'));
 gR(g,s(1),s(8)+a,s(3.5),s(10),dk);gR(g,s(16.5),s(8)-a,s(4),s(11),c);gR(g,s(16),s(18)-a,s(5),s(3),shade(c,.08));
 if(o.glow){const rs=srand(hash(o.c+o.glow));for(let i=0;i<6*k;i++){let x=s(5)+rs()*s(11),y=s(8)+rs()*s(9);for(let j=0;j<3*k;j++){gp(g,x,y,'!'+o.glow);x+=rs()<.5?1:0;y+=1}}}
 if(o.ice){gP(g,[[s(4),s(7)],[s(5),s(.5)],[s(7.5),s(7)]],'#e8ffff');gP(g,[[s(14),s(7)],[s(16),0],[s(17.5),s(7)]],'#e8ffff');gP(g,[[s(9),s(2)],[s(10.5),0],[s(12),s(2)]],'#e8ffff')}
 return bake(g)}
function genDragon(o,f){const g=Grid(58,42),c=o.c,dk=shade(c,-.25),bl=o.belly,wu=f?1:9;
 gL(g,14,30,3,22,c,4);gL(g,3,22,0,16,c,2);gP(g,[[26,18],[10,wu],[19,wu+2],[23,wu+11],[30,20]],shade(dk,-.12));
 gR(g,18,30,5,11,dk);gE(g,26,26,13,8,c);gE(g,27,30,9,3.6,bl);gL(g,34,22,42,11,c,6);gL(g,36,23,42,14,bl,2);
 gE(g,45,9,5.5,4,c);gR(g,48,8,9,4,c);gR(g,49,11,7,2,dk);gp(g,46,7,'!#ffd257');gp(g,47,7,'!#ffd257');gL(g,43,6,36,1,'#efe0c0',2);gL(g,46,5,43,0,'#efe0c0');
 gR(g,30,31,5,10,c);gR(g,29,39,8,2,dk);for(let i=0;i<5;i++)gP(g,[[15+i*4,19],[17+i*4,14.5],[19+i*4,19]],'#efe0c0');
 gP(g,[[28,19],[38,wu-1],[47,wu+3],[41,wu+10],[34,22]],shade(c,.05));gL(g,29,19,38,wu,dk);gL(g,38,wu,41,wu+10,dk);return bake(g)}
function genOrb(o,f){const g=Grid(46,46),c=o.c;for(let i=0;i<8;i++){const a=i/8*TAU+.2,x=23+Math.cos(a)*17,y=23+Math.sin(a)*17;gP(g,[[x-Math.sin(a)*3,y+Math.cos(a)*3],[23+Math.cos(a)*22.5,23+Math.sin(a)*22.5],[x+Math.sin(a)*3,y-Math.cos(a)*3]],shade(c,.12))}
 gE(g,23,23,17,17,c);gE(g,23,23,9,5.6,'!#ece6ff');gE(g,23+(f?1.2:-.6),23,3.8,3.8,'!#c8b0ff');gE(g,23+(f?1.2:-.6),23,1.6,1.6,'!#0a0814');gp(g,20,21,'!#ffffff');
 for(let i=0;i<10;i++){const a=i*2.4,r=10+(i%3)*2.5;gp(g,23+Math.cos(a)*r,23+Math.sin(a)*r*.9+(Math.sin(a)>0?5:-5),'!#9fb0ff')}return bake(g,{ol:'#06040c'})}
const GEN={slime:genSlime,beast:genBeast,human:genHuman,flyer:genFlyer,golem:genGolem,dragon:genDragon,orb:genOrb};
const PH=4;function genEnemy(d){const fr=[GEN[d.a](d,0),GEN[d.a](d,1)];return{fr,hw:fr[0].width*PH*.36}}
function genVein(t){const sd=stageDef(t),g=Grid(18,13),rk=mix(sd.dirt,'#77706a',.5),oc=oreCol(t);gE(g,9,8.5,8.4,4.6,rk);gR(g,1,9,16,4,rk);gE(g,6,6,4,3,shade(rk,.06));
 [[[4,9],[6,2],[8,9]],[[8,9],[11,0],[13,9]],[[12,10],[15,5],[16,10]]].forEach(p=>{gP(g,p,oc);gL(g,p[1][0],p[1][1]+1,p[1][0]-.5,p[0][1]-1,'!'+shade(oc,.55))});gp(g,15,6,'!'+matCol('g',t));gp(g,11,2,'!#ffffff');return bake(g)}
function genAnvil(){const g=Grid(36,16),c='#4c515c';gR(g,7,0,27,4,c);gP(g,[[0,1.4],[7,0],[7,4],[3.5,3]],c);gR(g,13,4,12,2,shade(c,-.1));gR(g,15,6,8,4,shade(c,-.15));gP(g,[[10,15.5],[13,10],[25,10],[28,15.5]],c);gR(g,8,14,22,2,shade(c,-.2));gR(g,8,0,25,1,'!#8a909c');return bake(g)}
function genBarrel(){const g=Grid(18,20),c='#7a4e2e';gR(g,1,2,16,17,c);gR(g,0,5,18,11,c);gR(g,0,4,18,2,'#3c3a40');gR(g,0,14,18,2,'#3c3a40');for(let x=4;x<16;x+=4)gL(g,x,2,x,18,shade(c,-.12));gE(g,9,2,8,2,'!#2f5c7c');gR(g,4,1,6,1,'!#6aa0c4');return bake(g)}
function genIcons(){const I={};let g=Grid(7,7);gE(g,3.5,3.5,3.5,3.5,'#e8b64a');gE(g,3.5,3.5,2,2,'#f6d27a');gp(g,3,3,'!#c08a2a');gp(g,2,1,'!#fff4c8');I.coin=bake(g);
 I.ore=[];I.gem=[];I.hide=[];for(let t=0;t<12;t++){g=Grid(8,7);const oc=oreCol(t);gP(g,[[0,5],[2,1],[5,0],[8,3],[7,7],[1,7]],oc);gp(g,3,2,'!'+shade(oc,.6));gp(g,4,2,'!'+shade(oc,.4));gp(g,5,5,'!'+shade(oc,-.3));I.ore.push(bake(g));
  g=Grid(7,7);const gc=matCol('g',t);gP(g,[[3.5,0],[7,3],[3.5,7],[0,3]],gc);gp(g,2,2,'!'+shade(gc,.6));gp(g,3,1,'!#ffffff');gp(g,4,5,'!'+shade(gc,-.3));I.gem.push(bake(g));
  g=Grid(8,7);const hc=matCol('h',t);gR(g,0,1,8,5,hc);gE(g,1,3.5,1.5,2.6,shade(hc,.15));gL(g,3,1,5,6,'!'+shade(hc,-.3));gp(g,6,2,'!'+shade(hc,.3));I.hide.push(bake(g))}
 g=Grid(10,10);gR(g,1,1,8,8,'#e8d8b8');gR(g,0,0,10,2,'#b89a70');gR(g,0,8,10,2,'#b89a70');for(let y=3;y<8;y+=2)gR(g,2,y,6,1,'!#8a7458');I.scroll=bake(g);
 g=Grid(11,11);for(let i=0;i<8;i++){const a=i/8*TAU;gR(g,5.5+Math.cos(a)*4.5-1,5.5+Math.sin(a)*4.5-1,2,2,'#a8a8b0')}gE(g,5.5,5.5,3.8,3.8,'#a8a8b0');gE(g,5.5,5.5,1.6,1.6,'!#221a17');I.gear=bake(g);
 g=Grid(7,9);gP(g,[[3.5,0],[7,5],[3.5,9],[0,5]],'#3a3030');gE(g,3.5,5.5,2.2,2.6,'#ff9a3a');gE(g,3.5,6,1,1.4,'!#fff2a0');I.coal=bake(g);return I}
function genBG(i){const d=stageDef(i),rs=srand(1000+i*77),out={};
 let g=Grid(200,72);const ph=[rs()*TAU,rs()*TAU,rs()*TAU];for(let x=0;x<200;x++){const t=x/200*TAU;const hh=30+Math.sin(t*2+ph[0])*12+Math.sin(t*5+ph[1])*6+Math.sin(t*11+ph[2])*2.5;for(let y=Math.floor(hh);y<72;y++)gp(g,x,y,y<hh+1?shade(d.far,.14):d.far)}
 if(d.deco==='lava'||d.deco==='ice')for(let x=0;x<200;x+=rs()*14+6){const l=rs()*16+6;for(let y=0;y<l;y++)gR(g,x-(1-y/l)*2,y,Math.max(1,(1-y/l)*4),1,shade(d.far,-.15))}
 out.far=bake(g,{ol:null,sh:false});
 g=Grid(200,64);const p2=[rs()*TAU,rs()*TAU];const hf=x=>40+Math.sin(x/200*TAU*3+p2[0])*5+Math.sin(x/200*TAU*7+p2[1])*2;for(let x=0;x<200;x++){const hh=hf(x);for(let y=Math.floor(hh);y<64;y++)gp(g,x,y,y<hh+1?shade(d.mid,.16):d.mid)}
 const dk=shade(d.mid,-.18),hl=shade(d.mid,.2);
 for(let x=6;x<194;x+=10+rs()*16){const b=Math.floor(hf(x))+1;
  if(d.deco==='grass'){gE(g,x,b-2,rs()*4+3,rs()*2+2,shade(d.mid,.06));if(rs()<.4)gp(g,x,b-5,'#f2d27a')}
  else if(d.deco==='tree'){const th=rs()*16+16;gR(g,x,b-5,2,6,'#3a2a20');for(let j=0;j<3;j++){const w=8-j*2;gP(g,[[x-w,b-4-j*th/4],[x+1,b-th-(2-j)],[x+2+w,b-4-j*th/4]],j%2?dk:shade(d.mid,-.08))}}
  else if(d.deco==='rock'){const th=rs()*22+10,wd=rs()*6+4;gR(g,x,b-th,wd,th,dk);for(let j=0;j<wd;j++)gp(g,x+j,b-th-(rs()*3|0),dk);gL(g,x,b-th,x,b,hl)}
  else if(d.deco==='lava'){const th=rs()*12+6;gP(g,[[x-5,b],[x,b-th],[x+5,b]],dk);if(rs()<.5)gL(g,x,b-th+3,x+1,b,'!#ff7a2a')}
  else if(d.deco==='ice'){const th=rs()*18+8;gP(g,[[x-3,b],[x,b-th],[x+3,b]],'#a8e4f0');gL(g,x,b-th+1,x,b,'!#effcff')}
  else if(d.deco==='bone'){if(rs()<.5){for(let j=0;j<4;j++)gL(g,x+j*4,b,x+j*4+3,b-14+j,'#d8ccb0',2)}else gE(g,x,b-3,4,3,'#d8ccb0')}
  else if(d.deco==='star'){const yy=b-10-rs()*20;gP(g,[[x-4,yy],[x,yy-5],[x+4,yy],[x,yy+7]],dk);gp(g,x,yy-1,'!#c8d0ff')}}
 out.mid=bake(g,{ol:null,sh:false});
 g=Grid(200,44);const gc=d.gr,dc=d.dirt;for(let x=0;x<200;x++){for(let y=4;y<44;y++)gp(g,x,y,y<7?(y===4?shade(gc,.15):gc):(rs()<.04?shade(dc,.15):dc));if(rs()<.3)gp(g,x,3,gc);if(rs()<.08)gp(g,x,2,gc)}
 for(let k=0;k<12;k++)gE(g,rs()*200,rs()*30+12,rs()*2+1,1,shade(dc,-.2));out.near=bake(g,{ol:null,sh:false});return out}
function genForgeBG(){const g=Grid(120,140),wall='#3a2622',mo='#24161a',rs=srand(9);
 for(let y=0;y<128;y++)for(let x=0;x<120;x++){const row=y>>2,off=(row%2)*4;const m=(y%4===3)||((x+off)%8===7);gp(g,x,y,m?mo:wall)}
 for(let r=0;r<32;r++)for(let c=0;c<16;c++){if(rs()<.35){const v=shade(wall,(rs()-.5)*.25);const off=(r%2)*4;for(let y=0;y<3;y++)for(let x=0;x<7;x++)gp(g,c*8+x-off,r*4+y,v)}}
 for(let y=128;y<140;y++)for(let x=0;x<120;x++)gp(g,x,y,(y===128)?'#6a4630':(x%20===0||y===134)?'#2e1c12':'#4a3020');
 gR(g,10,0,18,72,'#2e2422');for(let y=0;y<72;y+=6)gR(g,10,y,18,1,'#241c1a');
 gR(g,2,72,34,56,'#56463f');gE(g,19,74,17,6,'#56463f');for(let y=74;y<128;y+=6)for(let x=2;x<36;x++)if((x+((y/6|0)%2)*5)%10===0)gL(g,x,y,x,y+5,'#40332e');
 gR(g,9,98,20,30,'#120808');gE(g,19,98,10,5,'#120808');
 gR(g,86,22,24,24,'#4a3020');gR(g,88,24,20,20,'!#1a2546');gR(g,97,24,2,20,'#4a3020');gR(g,88,33,20,2,'#4a3020');gp(g,92,28,'!#e8ecff');gp(g,103,38,'!#aab4e0');gp(g,105,27,'!#cfd8ff');
 gR(g,74,66,44,3,'#5a3a24');gR(g,76,69,2,3,'#3a2416');gR(g,114,69,2,3,'#3a2416');
 gL(g,80,69,78,90,'#55555e',2);gL(g,84,69,86,90,'#55555e',2);gR(g,92,69,1,12,'#6a4428');gR(g,89,80,7,5,'#6a6e78');gR(g,103,69,1,16,'#6a4428');gP(g,[[104,72],[109,71],[110,78],[104,77]],'#7a7f88');
 gR(g,44,112,10,16,'#3a2a20');gR(g,43,110,12,3,'#4a3626');gE(g,49,110,5,1.5,'!#1a1210');
 return bake(g,{ol:null,sh:false})}
function buildSprites(){SPR.hero=[];for(let t=0;t<7;t++)SPR.hero.push([0,1,2,3].map(f=>genHero(t,f)));SPR.ico=genIcons();SPR.anvil=genAnvil();SPR.barrel=genBarrel();SPR.fbg=genForgeBG();SPR.ham=genHammer();
 SPR.en=[];SPR.boss=[];SPR.vein=[];SPR.bg=[];for(let i=0;i<7;i++){SPR.en.push(ST[i].en.map(genEnemy));SPR.boss.push(genEnemy(ST[i].boss));SPR.vein.push(genVein(i));SPR.bg.push(genBG(i))}}

// ===== UI =====
const UI={hits:[],spr:{},bump:{},dg:0,dm:[0,0,0],tab:'bl',ht:{}};
let PT={x:0,y:0,down:false,id:null};
function hit(id,x,y,w,h,cb,dn){UI.hits.push({id,x,y,w,h,cb,dn})}
function hitAt(x,y){for(let i=UI.hits.length-1;i>=0;i--){const h=UI.hits[i];if(x>=h.x&&x<=h.x+h.w&&y>=h.y&&y<=h.y+h.h)return h}return null}
function spring(id,pr){let s=UI.spr[id];if(!s)s=UI.spr[id]={v:1,vv:0,tg:1};s.tg=pr?.9:1;return s.v}
function kick(id,a){const s=UI.spr[id]||(UI.spr[id]={v:1,vv:0,tg:1});s.vv+=a}
function button(id,x,y,w,h,label,o){o=o||{};const pr=PT.down&&PT.id===id,dis=!!o.dis,sc=spring(id,pr&&!dis);
 cx.save();cx.translate(x+w/2,y+h/2);cx.scale(sc,sc);const X=-w/2,Y=-h/2,base=dis?'#3a302c':(o.col||'#4a3a33'),dy=pr?2:0;
 prect(X,Y+4,w,h,'#0b0807');prect(X,Y+dy,w,h-dy,shade(base,-.4));prect(X,Y+dy,w,h-4,base);cx.fillStyle=shade(base,.2);cx.fillRect(X+3,Y+dy+2,w-6,2);
 if(o.glow&&!dis){cx.globalAlpha=.25+.2*Math.sin(NOW*5);prect(X-2,Y-2+dy,w+4,h,'#ffd257');cx.globalAlpha=1;prect(X,Y+dy,w,h-4,base);cx.fillStyle=shade(base,.2);cx.fillRect(X+3,Y+dy+2,w-6,2)}
 if(o.sel){cx.strokeStyle='#ffd257';cx.lineWidth=2;cx.strokeRect(X+1,Y+dy+1,w-2,h-6)}
 const ty=dy-2-(o.sub?7:0);text(label,0,ty,{size:o.size||14,bold:1,color:dis?'#7a6a62':(o.tc||'#fff4e6')});
 if(o.sub)text(o.sub,0,dy+10,{size:11,color:dis?'#6a5a52':(o.sc||'#f0d8c0')});cx.restore();
 hit(id,x,y,w,h,dis?(o.dcb||null):o.cb,o.dn)}
function panel(x,y,w,h,o){o=o||{};prect(x,y+3,w,h,'#080605');prect(x,y,w,h,'#0e0a09');prect(x+2,y+2,w-4,h-4,o.c||'#2b211d');cx.fillStyle=o.hl||'#3f312b';cx.fillRect(x+4,y+2,w-8,2)}
function bar(x,y,w,h,k,c,bg){cx.fillStyle=bg||'#140e10';cx.fillRect(x-1,y-1,w+2,h+2);cx.fillStyle='#3a2e2a';cx.fillRect(x,y,w,h);cx.fillStyle=c;cx.fillRect(x,y,Math.max(0,w*clamp(k,0,1)),h);cx.fillStyle='rgba(255,255,255,.22)';cx.fillRect(x,y,Math.max(0,w*clamp(k,0,1)),Math.max(1,h/3|0))}
const TOAST=[];function toast(s,c){TOAST.push({s,c:c||'#f3e9dc',t0:NOW});if(TOAST.length>3)TOAST.shift();AU.blips(Math.min(8,[...s].length),.03)}
function drawToasts(){let i=0;for(let j=TOAST.length-1;j>=0;j--){const t=TOAST[j],a=NOW-t.t0;if(a>2.4){TOAST.splice(j,1);continue}const k=eBack(clamp(a/.3,0,1)),fa=a>2.1?(2.4-a)/.3:1,y=58+i*26+(1-k)*-16;cx.save();cx.globalAlpha=fa;const w=tw(t.s,11)+28;prect(W/2-w/2,y,w,22,'#0e0a09');prect(W/2-w/2+1,y+1,w-2,20,'#2b211d');text(t.s,W/2,y+11,{size:11,color:t.c,wave:1,pop:1,t0:t.t0,gap:.02});cx.restore();i++}}
function hintB(s,x,y,id){const b=Math.sin(NOW*4)*3,w=tw(s,11)+20;x=clamp(x,w/2+4,W-w/2-4);cx.save();cx.translate(0,b);prect(x-w/2,y-12,w,24,'#0e0a09');prect(x-w/2+1,y-11,w-2,22,'#f3e3c4');cx.fillStyle='#f3e3c4';cx.beginPath();cx.moveTo(x-5,y+11);cx.lineTo(x+5,y+11);cx.lineTo(x,y+17);cx.fill();text(s,x,y,{size:11,color:'#3a2418',shadow:null,wave:1.2,pop:1,t0:UI.ht[id]||(UI.ht[id]=NOW),gap:.03});cx.restore()}

// ===== Scenes / modal / transition =====
let SCN='title',M=null,TRN=null,STOP=0,SLOW=1,SLOWT=0,SHK=0,FLASH=0,FLC='#ffffff',saveT=0;
function shake(a){if(S.set.shk)SHK=Math.max(SHK,a)}
function flash(a,c){FLASH=Math.max(FLASH,a);FLC=c||'#ffffff'}
function openM(k,d){M={k,d:d||{},t0:NOW,cl:0};AU.tap()}
function closeM(){if(M&&!M.cl)M.cl=NOW}
function go(s){if(TRN)return;TRN={t:0,to:s,sw:0}}
function enterScene(s){SCN=s;PS.length=0;TX.length=0;if(s==='forge'){fReset('idle');Music.set('forge');ensureDaily();save()}else if(s==='hunt'){S.tut.h1=1;huntStart();Music.set('hunt')}else Music.set('title')}
let L={};
function layout(){L.hud=50;L.fb=H-268;L.card=L.fb+8;L.tabs=L.fb+88;L.row=L.fb+142;L.go=H-64;L.vb=H-172;L.gy=L.vb-46;L.AX=226;L.AY=L.fb-8-18*3+6;L.FX=57;L.BX=322;GS.x=L.AX-44;GS.y=L.fb-60}
function resize(){DPR=Math.min(window.devicePixelRatio||1,2.5);const iw=window.innerWidth||360,ih=window.innerHeight||640;SC=Math.min(iw/360,ih/640);W=360;H=ih/SC;OX=(iw-360*SC)/2;cv.width=Math.round(iw*DPR);cv.height=Math.round(ih*DPR);cv.style.width=iw+'px';cv.style.height=ih+'px';layout()}

// ===== Minigames =====
const MG={k:null};
function mgTap(spd,hot){Object.assign(MG,{k:'tap',mk:0,dir:1,spd,hot:hot===undefined?rr(.25,.75):hot,lock:0})}
function mgTapUpd(dt){if(MG.lock>0){MG.lock-=dt;return}MG.mk+=MG.dir*MG.spd*dt;if(MG.mk>1){MG.mk=2-MG.mk;MG.dir=-1}if(MG.mk<0){MG.mk=-MG.mk;MG.dir=1}}
function mgTapQ(){const d=Math.abs(MG.mk-MG.hot);return d<.055?1:d<.14?.6:.15}
function newHot(){let h,i=0;do{h=rr(.12,.88)}while(Math.abs(h-MG.mk)<.3&&i++<20);return h}
function mgHold(zc,zw,spd){Object.assign(MG,{k:'hold',v:0,zc,zw,spd,on:0,done:0})}
function mgHoldUpd(dt){if(MG.on&&!MG.done){MG.v+=MG.spd*(.35+1.1*MG.v)*dt;if(MG.v>=1.05){MG.v=1.05;holdRelease()}}}
function mgHoldQ(){const d=Math.abs(MG.v-MG.zc);if(d<=MG.zw*.22)return 1;if(d<=MG.zw/2)return .6;return MG.v<MG.zc?-1:-2}
function mgRing(spd,win){Object.assign(MG,{k:'ring',a:rr(.5,TAU-.5)-PI/2,spd,win,lock:0,tg:-PI/2})}
function mgRingUpd(dt){if(MG.lock>0){MG.lock-=dt;return}MG.a=(MG.a+MG.spd*TAU*dt)%TAU}
function mgRingQ(){const d=Math.abs((((MG.a-MG.tg+PI)%TAU)+TAU)%TAU-PI);return d<=MG.win.perf?1:d<=MG.win.ok?.6:0}

// ===== Forge =====
const GS={x:0,y:0,r:38,ang:0};
const F={mode:'idle',st:'',t:0,z:1,zt:1,fx:226,fy:300,tfx:226,tfy:300,slamT:-9,acc:[],n:0,res:null,kind:'bl',od:[0,0,0]};
function fReset(mode){Object.assign(F,{mode,st:'',t:0,acc:[],n:0,combo:0,res:null,step:0,named:0,dark:0,chg:0,chgT:0,hs:0,rvI:0,rvN:0,fast:0,q:0,out:null,sq:0,hit1:0,msg:'',sh:'long',zt:1,tfx:L.AX,tfy:L.AY-20,t0p:0});MG.k=null}
function setCam(x,y,z){F.tfx=x;F.tfy=y;F.zt=z}
const SWP=()=>({x:L.AX+6,y:L.AY-8});
function swCenter(img,frac,px,py,P,rot){const d=(frac-.5)*img.height*P;return{x:px+d*Math.sin(rot),y:py-d*Math.cos(rot)}}
function swPoint(img,frac,ccx,ccy,P,rot){const d=(frac-.5)*img.height*P;return{x:ccx-d*Math.sin(rot),y:ccy+d*Math.cos(rot)}}
function startCraft(k){if(F.mode!=='idle'||M)return;const t=craftTier(k);if(t<0)return;fReset(k==='bl'?'blade':k==='gd'?'guard':'handle');F.kind=k;F.tier=t;
 if(k==='bl'){F.st='pick';return}payCraft(k,t);S.tut['c'+k]=1;save();
 if(k==='gd'){F.st='heat';setCam(L.AX,L.AY-30,1.3);AU.n(.5,{ft:'lowpass',f:600,v:.2,a:.05})}
 else{F.st='wrap';F.res=mkHandle(t,.9);mgRing(.8,{perf:.13,ok:.36});setCam(L.AX,L.AY-12,1.6)}}
function pickShape(sh){if(F.st!=='pick')return;payCraft('bl',F.tier);F.sh=sh;F.st='heat';F.t=0;S.tut.cbl=1;save();AU.n(.5,{ft:'lowpass',f:600,v:.2,a:.05})}
function cancelCraft(){if(F.st==='pick')fReset('idle')}
const barLen=()=>lerp(34,112,F.n/5)*(1+.1*F.sq),barTh=()=>lerp(11,6,F.n/5)*(1-.3*F.sq);
function scr(x,y){return{x:F.fx+(x-F.fx)*F.z,y:F.fy+(y-F.fy)*F.z}}
function strikeFx(q,x,y,small){const sp=scr(x,y);if(q>=1){F.combo++;STOP=Math.max(STOP,.07);shake(5);burst(PS,x,y,30,{an:-PI/2,sp:PI*.55,v0:140,v1:420,g:600,dr:1.2,c:['#fff6c8','#ffd257','#ff9a3a'],k:'spk',s:2,l0:.25,l1:.6});ring(PS,x,y,'#ffd257',46,3);glow(PS,x,y,'#ffb040',70,.25);qprog('perfect');
  ptext(TX,F.combo>=2&&!small?'완벽 ×'+F.combo:'완벽!',sp.x,sp.y-70,{size:F.combo>=2?20:22,bold:1,color:'#ffd257',pop:1,wave:1.5,l:.9,vy:-18,big:1,ol:1})}
 else if(q>.5){F.combo=0;STOP=Math.max(STOP,.035);shake(2.5);burst(PS,x,y,16,{an:-PI/2,sp:PI*.6,v0:100,v1:300,g:600,dr:1.4,c:['#ffd257','#ff9a3a'],k:'spk',s:2});ring(PS,x,y,'#ff9a3a',30,2);ptext(TX,'좋아',sp.x,sp.y-66,{size:16,bold:1,color:'#fff4e6',pop:1,l:.75,vy:-14,ol:1})}
 else{F.combo=0;shake(1.5);burst(PS,x,y,6,{an:-PI/2,sp:PI*.7,v0:60,v1:160,g:500,c:['#ff9a3a','#a04a20'],k:'spk',s:2});ptext(TX,'아쉽다',sp.x,sp.y-62,{size:14,bold:1,color:'#a8968a',l:.7,vy:10,shake:1.5,ol:1})}}
function bladeStrike(){if(MG.lock>0||F.n>=5)return;const q=mgTapQ();F.acc.push(q);F.n++;MG.lock=.26;F.sq=1;F.slamT=NOW;
 const Lb=barLen(),x=L.AX-Lb/2+MG.mk*Lb,y=L.AY-barTh()/2;strikeFx(q,x,y);AU.clang(q,1+F.n*.03);
 if(F.n>=5){const Q=effQ(F.acc),pf=perfN(F.acc),ch=S.charm>0;F.g=rollGrade(Q,pf,ch);if(ch)S.charm--;if(F.g===4){S.pity=0;S.st.leg++}else S.pity++;if(F.g>=2)qprog('rare');F.res=mkBlade(F.tier,F.sh,F.g,1);F.pf=pf;
  if(pf>=5)ptext(TX,'완벽 5연속!',W/2,L.hud+50,{size:20,bold:1,color:'#ffd257',pop:1,wave:2,l:1.8,vy:0,big:1,ol:1,say:1});else if(pf>=4)ptext(TX,'완벽 4번!',W/2,L.hud+50,{size:16,bold:1,color:'#ffe08a',pop:1,wave:1.5,l:1.4,vy:0,ol:1});save()}}
function ringPos(){return{x:F.fx+Math.cos(MG.a)*60,y:F.fy+Math.sin(MG.a)*60}}
function wrapTap(){if(MG.lock>0||F.n>=3)return;const q=mgRingQ();F.acc.push(q);F.n++;MG.lock=.3;MG.spd+=.28;const p=ringPos();
 if(q>0){if(q>=1){F.combo++;qprog('perfect');ptext(TX,'완벽!',p.x,p.y-30,{size:18,bold:1,color:'#ffd257',pop:1,wave:1.5,l:.8,big:1,ol:1})}else ptext(TX,'좋아',p.x,p.y-30,{size:14,bold:1,color:'#fff4e6',pop:1,l:.7,ol:1});burst(PU,p.x,p.y,14,{c:['#ffd257','#fff6c8'],v0:60,v1:200,dr:2,k:'spk',s:2});shake(2);AU.creak(.4+F.n*.2);AU.click()}else{AU.fail();ptext(TX,'빗나감',p.x,p.y-30,{size:14,bold:1,color:'#a8968a',l:.7,shake:1.5,ol:1})}
 if(F.n>=3){const a=F.acc.reduce((s,v)=>s+(v||.15),0)/3;F.res=mkHandle(F.tier,.72+.43*a);F.msg='품질 '+Math.round(F.res.q*100)+'%';F.msgT=NOW;F.st='reveal1';F.t=0;F.hit1=0;MG.k=null}}
function holdRelease(){if(MG.k!=='hold'||!MG.on||MG.done)return;MG.on=0;MG.done=1;const q=mgHoldQ(),s=F.st;
 if(s==='grind'){const sharp=q===1?1.15:q===.6?1.07:q===-1?.9+.12*clamp(MG.v/MG.zc,0,1):.84;F.res.q=sharp;F.st='grindEnd';F.t=0;resultText(q,q===1?'완벽하게 섰다!':q===.6?'잘 섰다':q===-1?'덜 갈렸다':'너무 깎았다');F.msg='날카로움 '+Math.round(sharp*100)+'%';F.msgT=NOW}
 else if(s==='pour'){const gq=q===1?1.13:q===.6?1.03:q===-1?.8+.15*clamp(MG.v/MG.zc,0,1):.82;F.res=mkGuard(F.tier,gq);F.st='set';F.t=0;F.hs=0;resultText(q,q===1?'딱 맞게 부었다!':q===.6?'잘 부었다':q===-1?'덜 찼다':'넘쳤다');F.msg='품질 '+Math.round(gq*100)+'%';F.msgT=NOW}
 else if(s==='hold'){F.q=q>0?q:0;F.st='roll';F.t=0;F.under=q===-1}}
function resultText(q,s){const good=q>0;ptext(TX,s,W/2,L.hud+64,{size:q===1?20:16,bold:1,color:q===1?'#ffd257':good?'#fff4e6':'#ffb0a0',pop:1,wave:good?1.5:0,shake:good?0:1.2,l:1.2,vy:-10,big:q===1,ol:1});if(q===1){AU.okS(4);shake(3);qprog('perfect')}else if(!good)AU.fail();else AU.okS(1)}
function revStrike(i){F.step=i;F.slamT=NOW;F.zt=1.25+.2*i;F.tfy=L.AY-10;const p=SWP(),x=p.x,y=p.y-4,c=GR[i].c;
 shake(1.5+i*2.2);STOP=Math.max(STOP,.05+.03*i);burst(PS,x,y,18+18*i,{an:-PI/2,sp:PI*.7,v0:120,v1:300+90*i,g:600,dr:1.2,c:[c,'#fff6c8','#ffd257'],k:'spk',s:2,l0:.25,l1:.6+.1*i});ring(PS,x,y,c,40+22*i,2+i);glow(PS,x,y,c,60+20*i,.35,.9);
 const sp=scr(x,y);ptext(TX,'깡!',sp.x+rr(-8,8),sp.y-60-i*6,{size:18+i*6,bold:1,title:1,color:c,big:1,ol:1,l:.55,vy:-30});AU.clang(1,1+i*.1);AU.bell(i);if(i>=2)AU.thump();
 if(i===4){F.dark=0;F.chg=0;flash(1,'#fff6d0');shake(12);STOP=.15;F.zt=2.3;ring(PS,x,y,'#ffd257',200,6,.7);burst(PS,x,y,100,{c:['#ffd257','#fff6c8','#ffb040'],v0:140,v1:560,dr:1.6,g:120,k:'spk',s:2,l0:.5,l1:1.1});AU.legend()}}
function revealUpd(dt){const g=F.res.g;F.rvN-=dt*(F.fast?2.5:1);
 if(F.rvI<=g){if(g===4&&F.rvI===4){if(!F.chg){F.chg=1;F.chgT=0;AU.duck(2.4);AU.heart()}F.chgT+=dt;F.dark=clamp(F.chgT,0,1)}
  if(F.rvN<=0){revStrike(F.rvI);F.rvI++;F.rvN=(g===4&&F.rvI===4)?1.5:Math.max(.36,.56-.05*F.rvI)}}
 else if(F.rvN<=-.35){F.st='rise';F.t=0;F.zt=1;F.dark=0;F.tfy=L.AY-20;AU.click();ring(PS,SWP().x,SWP().y,'#ffffff',60,3)}}
function reveal1Upd(){if(F.t>=.3&&!F.hit1){F.hit1=1;F.slamT=NOW;F.zt=F.zt+.35;const p=SWP(),c=F.res.q>=1.05?'#ffd257':'#fff4e6';shake(4);STOP=.06;burst(PS,p.x,p.y-4,30,{an:-PI/2,sp:PI*.7,v0:120,v1:360,g:600,c:[c,'#ff9a3a'],k:'spk',s:2});ring(PS,p.x,p.y,c,60,3);const sp=scr(p.x,p.y);ptext(TX,'깡!',sp.x,sp.y-64,{size:24,bold:1,title:1,color:c,big:1,ol:1,l:.55,vy:-30});AU.clang(1,1.05);AU.bell(F.res.q>=1.12?3:F.res.q>=1.05?2:1)}
 if(F.t>=1.3){F.st='card';openM('card')}}
function enhOpen(k){if(F.mode!=='idle'||M||!S[k])return;fReset('enh'+k);F.kind=k;F.st='ready';const img=swBig(S.bl,S.gd,S.hd),p=SWP(),c=swCenter(img,.5,p.x,p.y,3,PI/2),fr=k==='bl'?img.fb:k==='gd'?img.fg:img.fh,pt=swPoint(img,fr,c.x,c.y,3,PI/2);setCam(pt.x,pt.y,k==='bl'?1.25:k==='gd'?1.9:1.8)}
function enhGo(){const k=F.kind;if(!(F.st==='ready'||F.st==='done'))return;const p=S[k];if(!p){fReset('idle');return}if(p.p>=MAXP[k])return;if(!canEnh(k)){toast(enhLack(k),'#ff8a6a');return}payEnh(k);F.out=null;F.under=0;
 if(k==='bl'){F.st='strike';mgTap(1.05+.05*p.p);F.n=0}else if(k==='gd'){F.st='ring';mgRing(.7+.09*p.p,gWin(p.p))}else{F.st='hold';mgHold(rr(.5,.8),hZone(p.p),.45+.03*p.p)}save()}
function enhLack(k){const c=enhCost(k);if(S.gold<c.g)return'금화가 모자라다';return matName(c.m,c.t)+'이(가) 모자라다'}
function bladeBar(){const img=swBig(S.bl,S.gd,S.hd),p=SWP(),c=swCenter(img,.5,p.x,p.y,3,PI/2),a=swPoint(img,img.fe,c.x,c.y,3,PI/2),b=swPoint(img,img.ft,c.x,c.y,3,PI/2);return{x0:a.x+4,x1:b.x-4,y:p.y-2}}
function enhStrike(){if(F.st!=='strike'||MG.lock>0)return;const q=mgTapQ();F.q=q;MG.lock=9;F.slamT=NOW;const b=bladeBar(),x=b.x0+MG.mk*(b.x1-b.x0);strikeFx(q,x,b.y,1);AU.clang(q,1.1);F.st='roll';F.t=0}
function gemTap(){if(F.st!=='ring'||MG.lock>0)return;const q=mgRingQ();F.q=q;MG.lock=9;F.gp=ringPos();F.st='roll';F.t=0;AU.click()}
function rollUpd(dt){const k=F.kind,t0=F.t-dt;
 if(k==='bl')for(const tt of[.12,.28]){if(t0<tt&&F.t>=tt){F.slamT=NOW;const b=bladeBar();AU.clang(.7,1.15+tt);shake(2);burst(PS,lerp(b.x0,b.x1,rr(.2,.8)),b.y,10,{an:-PI/2,sp:PI*.6,v0:100,v1:260,g:500,c:['#ffd257','#fff6c8'],k:'spk',s:2})}}
 const at=k==='bl'?.5:k==='gd'?.32:.25;if(t0<at&&F.t>=at)resolveEnh()}
function resolveEnh(){const k=F.kind,out=doEnh(k,F.q),p=S[k];F.out=out;F.st='done';F.t=0;save();const x=F.fx,y=F.fy,RY=L.hud+(L.fb-L.hud)*.4;
 if(out==='ok'){F.zt+=.15;flash(.35,'#fff6d0');shake(4+(p.p>=10?4:0));STOP=.08;ring(PS,x,y,'#ffd257',80,4);burst(PS,x,y,40,{c:['#ffd257','#fff6c8','#ffffff'],v0:80,v1:300,dr:2,k:'spk',s:2});AU.okS(p.p);
  ptext(TX,'+'+p.p,W/2,RY-30,{size:34,bold:1,title:1,color:'#ffd257',big:1,pop:1,wave:2,l:1.4,vy:-12,ol:1});if(k==='gd'){AU.gem();ptext(TX,'보석이 박혔다',W/2,RY+10,{size:12,bold:1,color:matCol('g',p.t),pop:1,l:1.2,vy:0,ol:1})}if(k==='hd')ptext(TX,'단단히 감겼다',W/2,RY+10,{size:12,bold:1,color:'#f3e3c4',pop:1,l:1.2,vy:0,ol:1})}
 else if(out==='br'){AU.shatter();shake(12);flash(.6,'#ff6a4a');STOP=.12;for(let i=0;i<50;i++)part(PS,{x:x+rr(-70,70),y:y+rr(-8,8),vx:rr(-280,280),vy:rr(-340,60),g:800,dr:.5,l:rr(.7,1.3),s:rr(2,5),c:pick(['#cfd4dc','#ffffff','#8a909c']),k:'sq'});ptext(TX,'펑!',W/2,RY-30,{size:36,bold:1,title:1,color:'#ff6a4a',big:1,shake:2,l:1.4,vy:-10,ol:1});ptext(TX,'날이 부서졌다',W/2,RY+12,{size:14,bold:1,color:'#ffb0a0',pop:1,l:2,vy:0,ol:1});setCam(L.AX,L.AY-20,1)}
 else{AU.fail();shake(2);if(k==='gd'){AU.crack();const gp=F.gp||{x,y};burst(PS,(gp.x-F.fx)/F.z+F.fx,(gp.y-F.fy)/F.z+F.fy,18,{c:[matCol('g',p.t),'#ffffff'],v0:60,v1:200,g:500,s:[2,4],k:'sq'});ptext(TX,'보석이 깨졌다',W/2,RY-20,{size:18,bold:1,color:'#ffb0a0',shake:1.5,l:1.2,vy:-10,ol:1})}
  else if(k==='hd'){if(F.under)ptext(TX,'덜 조였다',W/2,RY-20,{size:18,bold:1,color:'#ffb0a0',shake:1.2,l:1.2,vy:-10,ol:1});else{AU.snap();shake(5);ptext(TX,'툭! 끊어졌다',W/2,RY-20,{size:18,bold:1,color:'#ff8a7a',big:1,shake:2,l:1.2,vy:-10,ol:1})}}
  else ptext(TX,'실패',W/2,RY-20,{size:22,bold:1,color:'#a8968a',shake:1.5,l:1.1,vy:6,ol:1})}}
function forgeTap(){const f=F;
 if(f.mode==='blade'&&f.st==='strike')bladeStrike();
 else if(f.mode==='handle'&&f.st==='wrap')wrapTap();
 else if(f.mode==='enhbl'&&f.st==='strike')enhStrike();
 else if(f.mode==='enhgd'&&f.st==='ring')gemTap();
 else if(MG.k==='hold'&&!MG.done&&(f.st==='grind'||f.st==='pour'||f.st==='hold'))MG.on=1;
 else if(f.st==='reveal'||f.st==='rise'||f.st==='named')f.fast=1}
function updForge(dt){const f=F;f.t+=dt;f.sq=Math.max(0,f.sq-dt*6);const kz=1-Math.exp(-dt*7);f.z+=(f.zt-f.z)*kz;f.fx+=(f.tfx-f.fx)*kz;f.fy+=(f.tfy-f.fy)*kz;
 if(MG.k==='tap'){const was=MG.lock>0;mgTapUpd(dt);if(was&&MG.lock<=0&&f.mode==='blade'&&f.n<5)MG.hot=newHot()}else if(MG.k==='hold')mgHoldUpd(dt);else if(MG.k==='ring')mgRingUpd(dt);
 const m=f.mode,s=f.st;
 if(m==='blade'){
  if(s==='heat'&&f.t>=.7){f.st='strike';f.t=0;mgTap(.85)}
  else if(s==='strike'){MG.spd=.85+.24*f.n;if(f.n>=5&&MG.lock<=0){f.st='flash';f.t=0;MG.k=null;flash(.5,'#ffe0a0');ring(PS,L.AX,L.AY-4,'#fff6c8',90,4);AU.thump()}}
  else if(s==='flash'&&f.t>=.5){f.st='quench';f.t=0;f.hs=0}
  else if(s==='quench'){if(f.t>=.45&&!f.hs){f.hs=1;AU.hiss();for(let i=0;i<26;i++)part(PS,{x:L.BX+rr(-22,22),y:L.fb-66,vx:rr(-20,20),vy:rr(-90,-30),dr:1.5,l:rr(.8,1.6),s:rr(3,6),c:'#e8e4e0',k:'smoke'})}if(f.t>=1.1){f.st='grind';f.t=0;mgHold(rr(.6,.82),.2,.5);setCam(GS.x+16,GS.y-34,1.45)}}
  else if(s==='grind'){if(MG.on){GS.ang+=dt*16;const cxp=GS.x+10,cyp=GS.y-GS.r-2;if(R()<.9)burst(PS,cxp,cyp,2,{an:-.3,sp:.5,v0:120,v1:300,g:500,c:['#fff6c8','#ffd257','#ff9a3a'],k:'spk',s:1.5,l0:.2,l1:.45});AU.grind()}else GS.ang+=dt*1.5}
  else if(s==='grindEnd'&&f.t>=.8){f.st='place';f.t=0;setCam(L.AX,L.AY-20,1)}
  else if(s==='place'&&f.t>=.45){f.st='reveal';f.t=0;f.rvI=0;f.rvN=.3;setCam(L.AX,L.AY-10,1.15)}
  else if(s==='reveal')revealUpd(dt);
  else if(s==='rise'&&f.t>=(f.fast?.35:.7)){f.st='named';f.t=0;f.named=1;f.nameT=NOW;const n=[...f.res.name].length;if(f.res.g===4)for(let i=0;i<n;i++)AU.thump(i*.12);else AU.blips(n,.04)}
  else if(s==='named'&&f.t>=(f.res.g===4?1.6:.9)/(f.fast?2:1)){f.st='card';openM('card')}}
 else if(m==='guard'){
  if(s==='heat'&&f.t>=.6){f.st='pour';f.t=0;mgHold(rr(.55,.8),.2,.48);setCam(L.AX,L.AY-34,1.55)}
  else if(s==='pour'){if(MG.on){AU.pour();if(MG.v>MG.zc+MG.zw/2&&R()<.5)part(PS,{x:L.AX+rr(-42,42),y:L.AY-10,vy:rr(10,40),g:300,l:.6,s:3,c:'#ff9a3a',k:'sq'})}}
  else if(s==='set'){if(f.t>=.35&&!f.hs){f.hs=1;AU.gem();const p=SWP();burst(PS,p.x,p.y-8,16,{c:[matCol('g',f.tier),'#ffffff'],v0:40,v1:140,dr:2,s:2,k:'spk'})}if(f.t>=.8){f.st='reveal1';f.t=0;f.hit1=0;setCam(L.AX,L.AY-14,1.4)}}
  else if(s==='reveal1')reveal1Upd()}
 else if(m==='handle'){if(s==='reveal1')reveal1Upd()}
 else if(m.startsWith('enh')){if(s==='roll')rollUpd(dt);if(s==='hold'&&MG.on)AU.creak(MG.v)}}
function equipNew(){const k=F.kind,p=F.res;if(!p)return;const cur=S[k];if(cur){const v=meltV(k,cur);S.gold+=v;toast('쓰던 '+PNAME[k]+'을(를) 녹여 금화 +'+fmt(v),'#ffe2a0')}S[k]=p;fReset('idle');closeM();UI.bump.g=1;save();AU.okS(2)}
function meltNew(){const k=F.kind,p=F.res;if(!p)return;const v=meltV(k,p);S.gold+=v;if(k==='bl')addMat('o',p.t,3);fReset('idle');closeM();UI.bump.g=1;toast('녹여서 금화 +'+fmt(v),'#ffe2a0');save();AU.coin()}

// ===== Hunt =====
const Hn={on:false,en:[],veins:[]};
const HEAVY={sword:{d:.44,h:[.52],m:5,r:10,n:2,kb:240},axe:{d:.54,h:[.58],m:5,r:14,n:3,kb:340},dagger:{d:.52,h:[.2,.35,.5,.65,.8],m:1.1,r:6,n:1,kb:40},spear:{d:.4,h:[.5],m:4.6,r:16,n:3,kb:220},hammer:{d:.54,h:[.58],m:4.5,r:12,n:2,kb:280}};
let VID=0;
function huntStart(){const t=S.stage,mh=heroMaxHp();Object.assign(Hn,{on:true,t,def:stageDef(t),x:0,cam:-100,mhp:mh,hp:mh,en:[],veins:[],nextSpawn:140,nextVein:rr(80,150),cd:[0,0],boss:null,bm:false,dead:false,deadT:0,atkT:.3,sw:null,dash:null,loot:{g:0,o:0,x:0,k:0,gm:0,h:0},walk:0,winT:0,fin:false,q:[],intro:0,hfl:0,frz:0,mh:null});PW.length=0;TW.length=0;HV.sc=null}
function gapOf(e){return e.x-e.hw-(Hn.x+12)}
function front(){let b=null,bg=1e9;for(const e of Hn.en){if(e.dead||e.flee)continue;const g=gapOf(e);if(g>-40&&g<bg){bg=g;b=e}}return[b,bg]}
function mkEnemy(d,x,boss,spr){const t=Hn.t,ar=ARCH[d.a],hp=K.eHp*Math.pow(K.eHG,t)*ar.h*(boss?K.bossHp*(1+.12*Math.min(t,10)):1);return{d,ar,spr,x,hp,mhp:hp,atk:K.eAtk*Math.pow(K.eG,t)*ar.a*(boss?K.bossAtk*(1+.05*Math.min(t,10)):1),spd:ar.s*(boss?.7:1)*rr(.9,1.1),atkT:rr(.3,.9),wind:0,kb:0,fl:0,dead:0,ph:R()*10,boss,sp:boss?3:0,stun:0,hw:spr.hw,lunge:0,fly:ar.fly?1:0,bi:Math.min(t,6),mi:0,mv:null,shield:0,heat:0,shards:0,vul:0}}
function spawnGroup(){const t=Math.min(Hn.t,6),n=R()<.55?1:R()<.7?2:3;for(let i=0;i<n;i++){const j=Math.floor(R()*2);Hn.en.push(mkEnemy(Hn.def.en[j],Hn.x+300+i*30,false,SPR.en[t][j]))}}
function spawnVein(){Hn.veins.push({x:Hn.x+rr(250,310),hp:3,sh:0,dead:0,id:++VID,hits:0})}
const HUDX={g:22,o:94,gm:150,h:204};
function tapVein(v){if(!Hn.on||Hn.fin||v.dead)return;v.hp--;v.hits++;v.sh=1;const sx=v.x-Hn.cam,sy=L.gy-20;AU.rock(3-v.hp);burst(PW,v.x,L.gy-22,7,{an:-PI/2,sp:1.1,v0:60,v1:180,g:500,c:[oreCol(Hn.t),'#8a8078'],s:3,l0:.3,l1:.6});
 if(v.hp<=0){v.dead=.0001;const bon=1+trait('ore')/100,n=Math.round((2+R()*1.4)*bon);addMat('o',Hn.t,n);Hn.loot.o+=n;qprog('vein');S.tut.v1=1;AU.ore();ring(PW,v.x,L.gy-22,oreCol(Hn.t),44,3);ptext(TW,'+'+n,v.x,L.gy-66,{size:14,bold:1,color:mix(oreCol(Hn.t),'#ffffff',.3),pop:1,l:.8});for(let i=0;i<Math.min(n,5);i++)homing(sx,sy,HUDX.o,24,'ore',oreCol(Hn.t),()=>{UI.bump.o=1;AU.ore()});
  if(R()<.22*bon){addMat('g',Hn.t,1);Hn.loot.gm++;ptext(TW,matName('g',Hn.t)+'!',v.x,L.gy-88,{size:12,bold:1,color:matCol('g',Hn.t),pop:1,wave:1,l:1});homing(sx,sy,HUDX.gm,24,'gem',matCol('g',Hn.t),()=>{UI.bump.gm=1;AU.gem()})}}}
function hurtHero(d,e,big){if(Hn.dead||Hn.fin)return;d*=rr(.9,1.1);Hn.hp-=d;Hn.hfl=1;AU.hurt();ptext(TW,fmt(d),Hn.x+rr(-6,6),L.gy-116,{size:big?16:12,bold:1,color:'#ff6a5a',l:.7,vy:-30,big:big});if(big||d>Hn.mhp*.08)shake(big?9:3);burst(PW,Hn.x+10,L.gy-52,big?14:5,{c:['#ff6a5a','#ffffff'],v0:40,v1:160,g:300,s:2});if(Hn.hp<=0){Hn.hp=0;Hn.dead=true;Hn.deadT=0;Hn.frz=0;SLOW=.35;SLOWT=.7;AU.boom();shake(8)}}
function eY(e){return L.gy-(e.fly?80:40)-(e.boss?60:0)}
function dmgEnemy(e,d,cr,kb,k){if(e.boss){if(e.shield>0){d*=.1;ptext(TW,'막힘',e.x-e.hw*.5,eY(e)-30,{size:11,bold:1,color:'#9fc8ff',l:.5,vy:-20});AU.clang(.2,1.6)}if(e.vul>0)d*=1.5;if(e.shards>0&&k==='atk'){e.shards--;d*=.25;burst(PW,e.x,eY(e),10,{c:['#c8b0ff','#ffffff'],v0:60,v1:200,s:3});AU.crack()}if(e.heat>0)e.heat=Math.max(0,e.heat-.025)}
 e.hp-=d;e.fl=1;e.kb+=kb*(e.boss?.12:1)*(cr?1.3:1);const y=eY(e);
 ptext(TW,fmt(d),e.x+rr(-8,8),y-20,{size:cr?19:k==='atk'?13:16,bold:1,color:cr?'#ffd257':k==='atk'?'#fff4e6':'#ffb070',l:.75,vy:-34,big:1,ol:1});
 burst(PW,e.x-e.hw*.6,y+10,cr?14:6,{an:PI,sp:1.1,v0:80,v1:cr?320:200,g:400,c:['#fff6c8','#ffd257',e.d.c],k:'spk',s:2,l0:.15,l1:.35});
 if(cr){STOP=Math.max(STOP,.05);ring(PW,e.x-e.hw*.5,y+10,'#ffd257',30,2)}AU.hit(cr||k!=='atk');if(e.hp<=0)killEnemy(e)}
function bossLabel(e,s,c){ptext(TW,s,e.x,eY(e)-110,{size:20,bold:1,color:c||'#ffffff',pop:1,wave:2,l:1.1,big:1,ol:1})}
const TELE={slam:1,laser:1,freeze:1,breath:1};
function dealTargets(m,r,n,kb,k){const tg=Hn.en.filter(e=>!e.dead&&!e.flee&&gapOf(e)<=r&&gapOf(e)>-50).sort((a,b)=>gapOf(a)-gapOf(b)).slice(0,n);let tot=0;const st=stats();
 for(const e of tg){let d=heroAtk()*m*rr(.9,1.1);const cr=R()<st.cc;if(cr)d*=st.cm;
  if(e.boss){const mv=e.mv;
   if(k==='heavy'){if(mv&&mv.ph==='tele'&&TELE[mv.k]){endMove(e);e.stun=1.5;d*=1.5;S.tut['m'+mv.k]=9;AU.cut();shake(8);STOP=.12;flash(.35);bossLabel(e,'끊었다!')}
    if(e.shield>0){e.shield=0;e.stun=2;e.vul=2.5;if(e.mv&&e.mv.k==='shield')endMove(e);S.tut.mshield=9;AU.cut();shake(8);STOP=.1;bossLabel(e,'방패를 깼다!','#9fc8ff');burst(PW,e.x-e.hw,eY(e),24,{c:['#9fc8ff','#ffffff'],v0:80,v1:300,g:400,s:3})}
    if(e.heat>0)e.heat=Math.max(0,e.heat-.2)}
   if(k==='sweep'){if(e.heat>.05){e.heat=Math.max(0,e.heat-.45);S.tut.mheat=9;bossLabel(e,'식혔다!','#9fe0ff');burst(PW,e.x,eY(e),20,{c:['#e8e4e0','#9fe0ff'],v0:40,v1:160,g:-80,s:4,k:'smoke'})}if(e.shards>0){e.shards=0;S.tut.mshards=9;bossLabel(e,'파편을 깼다!','#c8b0ff');AU.crack();burst(PW,e.x,eY(e),30,{c:['#c8b0ff','#ffffff'],v0:80,v1:320,s:3})}}}
  tot+=d;dmgEnemy(e,d,cr,kb,k)}
 const dr=trait('drain');if(dr&&tot>0)Hn.hp=Math.min(Hn.mhp,Hn.hp+tot*dr/100);if(k!=='atk'&&tg.length)shake(k==='heavy'?5:4);return tg.length}
function killEnemy(e){e.dead=.0001;const t=Hn.t,gm=1+trait('gold')/100,g=Math.round(K.gold*Math.pow(2,t)*gm*(e.boss?30:e.mini?.3:1)*rr(.85,1.15));S.gold+=g;Hn.loot.g+=g;Hn.loot.k++;S.st.kills++;qprog('kill');const xp=Math.round(2*Math.pow(1.9,t)*(e.boss?20:e.mini?.3:1));Hn.loot.x+=xp;gainXp(xp);
 const y=eY(e),sx=e.x-Hn.cam;burst(PW,e.x,y,e.boss?60:16,{c:[e.d.c,shade(e.d.c,.4),'#ffffff'],v0:60,v1:e.boss?420:220,g:300,s:[2,5],sh:1,l0:.3,l1:.8});ring(PW,e.x,y,'#ffffff',e.boss?120:34,3);AU.pop();
 for(let i=0;i<(e.boss?12:e.mini?1:3);i++)homing(sx,y,HUDX.g,24,'coin',null,()=>{UI.bump.g=1;AU.coin()});
 if(!e.boss&&!e.mini){const bon=1+trait('ore')/100;if(R()<.12*bon){addMat('o',t,1);Hn.loot.o++;homing(sx,y,HUDX.o,24,'ore',oreCol(t),()=>{UI.bump.o=1;AU.ore()})}if(R()<.18*bon){addMat('h',t,1);Hn.loot.h++;homing(sx,y,HUDX.h,24,'hide',matCol('h',t),()=>{UI.bump.h=1;AU.ore()})}
  if(!Hn.bm&&S.prog<needN()){S.prog++;if(S.prog>=needN()){toast('보스가 깨어났다','#ff9a7a');AU.roar()}}}
 if(e.boss){SLOW=.25;SLOWT=1;STOP=.15;flash(.9,'#ffffff');shake(14);AU.boom();Hn.winT=1.8;for(const o of Hn.en)if(o.mini&&!o.dead)o.flee=1}}
function mkSwing(k){const st=stats(),sh=st.sh,pose=k==='heavy'?sh.hv:sh.pose;let d,h;if(k==='atk'){d=Math.min(.36,.7/st.aps);h=[.5]}else if(k==='heavy'){d=HEAVY[sh.hv].d;h=HEAVY[sh.hv].h}else{d=.5;h=[.35,.65]}return{k,pose,hv:sh.hv,t:0,d,h,done:h.map(()=>0),alt:R()<.5}}
function swingHit(s){const st=stats(),sh=st.sh,reach=sh.r-14;if(s.k==='atk'){AU.swing();dealTargets(1,reach+4,sh.p,sh.kb,'atk');if(R()*100<trait('multi'))Hn.q.push({t:.1,m:.8,r:reach+4,n:sh.p,kb:sh.kb*.5,k:'atk'})}
 else if(s.k==='heavy'){const Hd=HEAVY[s.hv];dealTargets(Hd.m*sh.s,reach+Hd.r,Hd.n,Hd.kb,'heavy');if(s.hv==='axe'||s.hv==='hammer'){burst(PW,Hn.x+44,L.gy,18,{an:-PI/2,sp:1.3,v0:80,v1:260,g:600,c:[Hn.def.gr,Hn.def.dirt],s:3});ring(PW,Hn.x+44,L.gy,'#fff6c8',60,3)}}
 else dealTargets(1.2*sh.s,120,99,170,'sweep')}
function skill(i){if(!Hn.on||Hn.dead||Hn.fin||Hn.cd[i]>0||Hn.frz>0)return false;if(Hn.sw&&Hn.sw.k!=='atk')return false;const[f,g]=front(),st=stats(),reach=st.sh.r-14;
 if(i===0){const b=Hn.boss;if(b&&!b.dead&&b.mv&&b.mv.k==='charge'&&b.mv.ph==='dash'&&gapOf(b)<170){Hn.cd[0]=K.cd0;endMove(b);b.stun=2.2;b.vul=2;b.kb=0;b.x=Hn.x+12+b.hw+30;Hn.sw=mkSwing('heavy');qprog('skill');S.tut.mcharge=9;AU.cut();shake(10);STOP=.14;flash(.4);bossLabel(b,'받아쳤다!','#ffd257');dmgEnemy(b,heroAtk()*8*st.sh.s,true,0,'heavy');kick('sk0',-8);return true}
  if(!f||g>150)return false;Hn.cd[0]=K.cd0;if(g>reach-4)Hn.dash={t:0,d:.12,x0:Hn.x,x1:Hn.x+(g-(reach-8))};Hn.sw=mkSwing('heavy');qprog('skill');AU.whoosh(1);kick('sk0',-8)}
 else{if(!f||g>130)return false;Hn.cd[1]=K.cd1;Hn.sw=mkSwing('sweep');AU.whoosh(2);kick('sk1',-8)}
 Hn.atkT=Math.max(Hn.atkT,.25);return true}
function bossTxt(s,c){ptext(TW,s,Hn.x,L.gy-140,{size:18,bold:1,color:c,pop:1,wave:1.5,l:1,big:1,ol:1})}
function breakIce(){if(Hn.frz<=0||Hn.dead)return;Hn.frz-=.75;AU.rock(2);burst(PW,Hn.x+rr(-14,14),L.gy-rr(20,90),8,{c:['#dff6ff','#9fe0ff','#ffffff'],v0:60,v1:200,g:500,s:3});shake(2);if(Hn.frz<=0){Hn.frz=0;AU.ice();bossTxt('깼다!','#dff6ff');S.tut.mfreeze=9;burst(PW,Hn.x,L.gy-50,30,{c:['#dff6ff','#9fe0ff','#ffffff'],v0:80,v1:300,g:500,s:4})}}
function bossStart(){if(!Hn.on||Hn.bm||Hn.dead||Hn.fin||S.prog<needN())return;Hn.bm=true;Hn.intro=1.6;for(const e of Hn.en)if(!e.dead)e.flee=1;const d=Hn.def.boss,b=mkEnemy(d,Hn.x+340,true,SPR.boss[Math.min(Hn.t,6)]);Hn.en.push(b);Hn.boss=b;Music.set('boss');AU.roar();shake(6);ptext(TX,d.n,W/2,L.hud+110,{size:28,bold:1,title:1,color:'#ffb0a0',pop:1,wave:1.5,l:2.4,gap:.09,vy:0,ol:1,say:1});ptext(TX,Hn.def.n+'의 주인',W/2,L.hud+142,{size:11,color:'#e8c8c0',pop:1,l:2.4,vy:0,gap:.03})}
function startMove(e){const ms=BMOV[e.bi],k=ms[e.mi%ms.length];e.mi++;const m={k,t:0,ph:'tele'};e.mv=m;bossLabel(e,MVN[k],TELE[k]?'#ff9a7a':'#ffe08a');
 if((S.tut['m'+k]||0)<2){S.tut['m'+k]=(S.tut['m'+k]||0)+1;Hn.mh={k,t:3.5}}
 if(TELE[k]){AU.tele();if(k==='breath')AU.n(1,{ft:'lowpass',f:300,fto:900,v:.18,a:.3})}
 else if(k==='split'){for(let i=0;i<3;i++){const mn=mkEnemy(Hn.def.en[0],e.x-20+i*36,false,SPR.en[Math.min(Hn.t,6)][0]);mn.mini=1;mn.hp=mn.mhp=mn.hp*1.2;mn.kb=rr(80,200);Hn.en.push(mn)}AU.pop();burst(PW,e.x,eY(e),30,{c:[e.d.c,'#ffffff'],v0:80,v1:260,g:300,s:4});endMove(e)}
 else if(k==='charge'){m.ph='back';AU.roar()}
 else if(k==='shield'){e.shield=3.4;AU.clang(.6,.7);m.ph='hold'}
 else if(k==='shards'){e.shards=5;AU.gem();burst(PW,e.x,eY(e),20,{c:['#c8b0ff','#ffffff'],v0:40,v1:160,s:3});endMove(e)}}
function endMove(e){e.mv=null;e.sp=K.spCd*rr(.85,1.15)}
function updBoss(e,dt,gap){if(e.vul>0)e.vul-=dt;if(e.shield>0)e.shield-=dt;
 if(BPAS[e.bi]==='heat'){if(!e.heatOn&&gap<80){e.heatOn=1;bossLabel(e,MVN.heat,'#ffb03a');if((S.tut.mheat||0)<2){S.tut.mheat=(S.tut.mheat||0)+1;Hn.mh={k:'heat',t:3.5}}}if(e.heatOn){e.heat+=dt/8.5;if(e.heat>=1){e.heat=0;hurtHero(e.atk*4.5,e,true);AU.boom();shake(12);flash(.5,'#ff8a3a');burst(PW,e.x-e.hw,eY(e),50,{c:['#ff8a3a','#ffd257','#ff4a2a'],v0:120,v1:420,g:200,s:4});bossLabel(e,'폭발!','#ff8a3a')}}}
 const m=e.mv;if(!m){e.sp-=dt;if(e.sp<=0&&gap<70)startMove(e);return false}m.t+=dt;
 if(TELE[m.k]){if(m.ph==='tele'){if(m.t>=K.tele*(m.k==='breath'?1.15:1)){m.t=0;
    if(m.k==='slam'||m.k==='laser'){hurtHero(e.atk*4,e,true);e.lunge=1;if(m.k==='laser'){m.ph='beam';AU.n(.3,{f:3000,v:.25});flash(.3,'#c8b0ff')}else{burst(PW,Hn.x+20,L.gy,20,{an:-PI/2,sp:1.3,v0:80,v1:260,g:600,c:[Hn.def.gr,'#ffffff'],s:3});endMove(e)}}
    else if(m.k==='freeze'){Hn.frz=3;Hn.sw=null;AU.ice();shake(5);bossTxt('얼었다!','#9fe0ff');if((S.tut.mfreeze||0)<9)Hn.mh={k:'freeze',t:3};endMove(e)}
    else{m.ph='fire';m.tick=0}}}
  else if(m.ph==='beam'){if(m.t>=.25)endMove(e)}
  else if(m.ph==='fire'){m.tick-=dt;if(m.tick<=0){m.tick=.2;hurtHero(e.atk*.7,e,false);AU.fire()}const sx=e.x-e.hw*.9,sy=eY(e)-20;for(let i=0;i<3;i++)part(PW,{x:sx,y:sy,vx:rr(-520,-380),vy:rr(-20,90),dr:1,l:rr(.25,.45),s:rr(4,7),c:pick(['#ff8a3a','#ffd257','#ff4a2a']),k:'sq',sh:1});if(m.t>=1.6)endMove(e)}
  return true}
 if(m.k==='charge'){if(m.ph==='back'){e.x+=290*dt;if(m.t>=.55){m.ph='tele2';m.t=0;AU.tele()}}else if(m.ph==='tele2'){if(R()<.5)part(PW,{x:e.x+rr(-20,20),y:L.gy-2,vx:rr(20,60),vy:rr(-30,-10),l:.5,s:4,c:'#b8a898',k:'smoke'});if(m.t>=.55){m.ph='dash';m.t=0}}
  else if(m.ph==='dash'){e.x-=600*dt;if(R()<.8)part(PW,{x:e.x+e.hw,y:L.gy-rr(10,50),vx:200,vy:0,l:.3,s:3,c:'#e8e0d8',k:'sq'});if(gapOf(e)<=8){hurtHero(e.atk*4,e,true);Hn.x-=24;e.lunge=1;endMove(e)}}return true}
 if(m.k==='shield'){if(e.shield<=0)endMove(e);return false}
 endMove(e);return false}
function updHero(dt){Hn.cd[0]=Math.max(0,Hn.cd[0]-dt);Hn.cd[1]=Math.max(0,Hn.cd[1]-dt);Hn.hfl=Math.max(0,Hn.hfl-dt*5);if(Hn.mh){Hn.mh.t-=dt;if(Hn.mh.t<=0)Hn.mh=null}
 if(Hn.frz>0){Hn.frz-=dt;if(Hn.frz<=0){Hn.frz=0;AU.ice()}return}
 for(let i=Hn.q.length-1;i>=0;i--){const q=Hn.q[i];q.t-=dt;if(q.t<=0){Hn.q.splice(i,1);dealTargets(q.m,q.r,q.n,q.kb,q.k)}}
 const st=stats(),reach=st.sh.r-14;
 if(Hn.dash){const d=Hn.dash;d.t+=dt;Hn.x=lerp(d.x0,d.x1,eOut(clamp(d.t/d.d,0,1)));if(d.t>=d.d)Hn.dash=null}
 if(Hn.sw){const s=Hn.sw;s.t+=dt;for(let i=0;i<s.h.length;i++)if(!s.done[i]&&s.t>=s.h[i]*s.d){s.done[i]=1;swingHit(s)}if(s.t>=s.d)Hn.sw=null}
 const[f,g]=front();
 if((!f||g>reach)&&!Hn.dash&&(!Hn.sw||Hn.sw.k==='atk')&&!Hn.winT){Hn.x+=K.walk*dt;Hn.walk+=dt}
 Hn.atkT-=dt;if(f&&g<=reach&&Hn.atkT<=0&&!Hn.sw){Hn.sw=mkSwing('atk');Hn.atkT=1/st.aps}
 if(!f||g>90)Hn.hp=Math.min(Hn.mhp,Hn.hp+Hn.mhp*.04*dt);
 if(!Hn.bm){if(Hn.x>=Hn.nextSpawn){spawnGroup();Hn.nextSpawn=Hn.x+rr(150,230)}if(Hn.x>=Hn.nextVein){spawnVein();Hn.nextVein=Hn.x+rr(220,340)}}}
function updEnemy(e,dt){e.ph+=dt;e.fl=Math.max(0,e.fl-dt*6);e.lunge=Math.max(0,e.lunge-dt*5);if(e.dead){e.dead+=dt;return}if(e.flee){e.x+=140*dt;e.fa=(e.fa===undefined?1:e.fa)-dt*1.6;return}
 e.x+=e.kb*dt;e.kb*=Math.exp(-7*dt);if(e.stun>0){e.stun-=dt;return}const gap=gapOf(e);
 if(!(e.mv&&e.mv.k==='charge')&&gap>8&&Math.abs(e.kb)<40)e.x-=e.spd*dt*Math.min(1,(gap-8)/12+.35);if(Hn.dead||Hn.fin)return;
 if(e.boss&&updBoss(e,dt,gap))return;
 if(gap<=14){e.atkT-=dt;if(e.wind>0){e.wind-=dt;if(e.wind<=0){hurtHero(e.atk,e);e.lunge=1;e.atkT=e.ar.i*rr(.9,1.1)}}else if(e.atkT<=0)e.wind=K.wind}}
function updHunt(dt){if(!Hn.on)return;Hn.intro=Math.max(0,Hn.intro-dt);
 if(Hn.dead){Hn.deadT+=dt;if(Hn.deadT>1.6&&!Hn.fin){Hn.fin=1;openM('sum',{dead:1})}}else if(!Hn.fin)updHero(dt);
 if(!Hn.fin||Hn.dead)for(const e of Hn.en)updEnemy(e,dt);
 Hn.en=Hn.en.filter(e=>!(e.dead>.35)&&!(e.flee&&e.fa<=0)&&e.x>Hn.cam-140);
 for(const v of Hn.veins){v.sh=Math.max(0,v.sh-dt*6);if(v.dead)v.dead+=dt}Hn.veins=Hn.veins.filter(v=>!(v.dead>.4)&&v.x>Hn.cam-70);
 Hn.cam=lerp(Hn.cam,Hn.x-100,1-Math.exp(-dt*8));
 if(Hn.winT>0){Hn.winT-=dt;if(Hn.winT<=0){Hn.winT=0;stageClear()}}}
function stageClear(){const nt=Hn.t+1;if(nt>S.stage)S.stage=nt;addMat('o',nt,12);addMat('g',nt,3);addMat('h',nt,4);const g=Math.round(K.gold*Math.pow(2,Hn.t)*25);S.gold+=g;Hn.loot.g+=g;S.prog=0;Hn.fin=1;save();openM('stage',{t:nt,g});AU.okS(7)}
function leaveHunt(){if(Hn.fin||Hn.dead)return;Hn.fin=1;openM('sum',{dead:0})}
function toForge(){closeM();go('forge')}

// ===== Drawing: forge =====
const HV={f:1,ft:0,lx:0,sc:null};
function scarfUpd(sc,ax,ay,dt,wind){if(!sc.length){for(let i=0;i<6;i++)sc.push({x:ax-i*5,y:ay,px:ax-i*5,py:ay})}sc[0].x=ax;sc[0].y=ay;for(let i=1;i<sc.length;i++){const p=sc[i],vx=(p.x-p.px)*.9,vy=(p.y-p.py)*.9;p.px=p.x;p.py=p.y;p.x+=vx-wind*dt*60;p.y+=vy+dt*40+Math.sin(NOW*6+i)*.25}for(let k=0;k<3;k++)for(let i=1;i<sc.length;i++){const a=sc[i-1],b=sc[i],dx=b.x-a.x,dy=b.y-a.y,d=Math.hypot(dx,dy)||1;b.x=a.x+dx/d*5.2;b.y=a.y+dy/d*5.2}}
function scarfDraw(sc,col,P){for(let i=sc.length-1;i>=1;i--){const p=sc[i],s=Math.max(2,P*(1.6-i*.15));cx.fillStyle='#140e10';cx.fillRect(p.x-s/2-1,p.y-s/2-1,s+2,s+2)}for(let i=sc.length-1;i>=1;i--){const p=sc[i],s=Math.max(2,P*(1.6-i*.15));cx.fillStyle=i%2?col:shade(col,-.2);cx.fillRect(p.x-s/2,p.y-s/2,s,s)}}
const SMS=[];
function slamK(){const k=NOW-F.slamT;return k<.05?k/.05:k<.1?1:1-clamp((k-.1)/.25,0,1)}
function smithTarget(){const f=F,sl=slamK();
 if(f.mode==='blade'&&f.st==='strike'){const Lb=barLen(),bx=L.AX-Lb/2+MG.mk*Lb;return{x:bx,y:lerp(L.AY-80,L.AY-barTh()/2-11,sl)}}
 if(f.mode==='blade'&&(f.st==='heat'||f.st==='flash'))return{x:L.AX+20,y:L.AY-70};
 if(f.st==='reveal'||f.st==='reveal1'){let hy=L.AY-90;if(f.chg&&f.rvI===4)hy=L.AY-118+Math.sin(NOW*40)*2;return{x:L.AX+6,y:lerp(hy,L.AY-20,sl)}}
 if(f.mode==='enhbl'&&S.bl){const b=bladeBar(),bx=f.st==='strike'||f.st==='roll'?b.x0+MG.mk*(b.x1-b.x0):lerp(b.x0,b.x1,.5);return{x:bx,y:lerp(L.AY-84,L.AY-22,sl)}}
 return{x:L.AX+34,y:L.AY-5+Math.sin(NOW*2.2)}}
function drawSmith(dt){const P=4,x=L.AX-10,feet=L.fb-14,tier=Math.min(S.stage,6),br=Math.sin(NOW*2.2)*.015,img=SPR.hero[tier][1];
 scarfUpd(SMS,x-6,feet-16*P,dt,.5);scarfDraw(SMS,SCARF[tier],P*.8);spr(img,x,feet,P,{sy:1+br,sx:1-br});
 const shX=x,shY=feet-15*P*(1+br),tg=smithTarget(),ham=SPR.ham,hl=10*P,dx=tg.x-shX,dy=tg.y-shY,dd=Math.hypot(dx,dy)||1,ux=dx/dd,uy=dy/dd,al=clamp(dd-hl,4,7*P),hx2=shX+ux*al,hy=shY+uy*al,rot=Math.atan2(dy,dx)+PI/2;
 cx.strokeStyle='#140e10';cx.lineWidth=P*2+2;cx.lineCap='square';cx.beginPath();cx.moveTo(shX,shY);cx.lineTo(hx2,hy);cx.stroke();cx.strokeStyle=HPAL[tier].body;cx.lineWidth=P*2;cx.beginPath();cx.moveTo(shX,shY);cx.lineTo(hx2,hy);cx.stroke();
 spr(ham,hx2,hy,P,{ax:ham.pv[0],ay:ham.pv[1],rot});cx.fillStyle='#efc39a';cx.fillRect(hx2-P,hy-P,P*2,P*2)}
function drawFire(){const x=L.FX,y=L.fb-48,st=Math.floor(NOW*12),rs=srand(st*7+1),cols=['#c0392b','#ff8a3a','#ffd257','#fff2a0'];for(let l=0;l<4;l++)for(let i=0;i<9;i++){const fx=x-26+i*6+(rs()*2|0),h=(26-l*6)*(0.6+rs()*.6)*(1-Math.abs(i-4)/7);cx.fillStyle=cols[l];cx.fillRect(fx+l,y-h,5-l,h)}
 if(R()<.3)part(PS,{x:x+rr(-20,20),y:y-30,vx:rr(-12,12),vy:rr(-70,-30),dr:.4,l:rr(1,2.2),s:2,c:pick(['#ffd257','#ff8a3a']),k:'sq',sh:1})}
function hotBar(x,y,Lb,th,hot){glowAt(x,y,'#ff8c28',Lb*.75,.45);
 cx.fillStyle='#140e10';cx.fillRect(x-Lb/2-1,y-th/2-1,Lb+2,th+2);cx.fillStyle='#d9531e';cx.fillRect(x-Lb/2,y-th/2,Lb,th);cx.fillStyle='#ff9a3a';cx.fillRect(x-Lb/2,y-th/2,Lb,Math.max(2,th*.55));cx.fillStyle='#ffd8a0';cx.fillRect(x-Lb/2+2,y-th/2,Lb-4,1.5);
 if(hot>=0){const hx2=x-Lb/2+hot*Lb;glowAt(hx2,y,'#ffe68c',.14*Lb*1.4*(1+Math.sin(NOW*10)*.12),.75);const pw=.14*Lb,c2=Math.max(3,.055*Lb*2);cx.fillStyle='#ffe08a';cx.fillRect(hx2-pw,y-th/2,pw*2,th);cx.fillStyle='#fffbe8';cx.fillRect(hx2-c2/2,y-th/2-1,c2,th+2)}}
function tapBar(x0,x1,y,mk,hot){const Lb=x1-x0;cx.fillStyle='rgba(20,14,16,.75)';cx.fillRect(x0-3,y-6,Lb+6,12);cx.fillStyle='#4a3a33';cx.fillRect(x0,y-3,Lb,6);const hx2=x0+hot*Lb;cx.fillStyle='#d8c8b4';cx.fillRect(hx2-.14*Lb,y-3,.28*Lb,6);cx.fillStyle='#ffd257';cx.fillRect(hx2-.055*Lb,y-4,.11*Lb,8);const mx=x0+mk*Lb;cx.fillStyle='#fff6c8';cx.beginPath();cx.moveTo(mx-6,y-13);cx.lineTo(mx+6,y-13);cx.lineTo(mx,y-5);cx.fill();cx.fillRect(mx-1,y-5,2,10)}
function drawGrindstone(){const g=GS,show=F.mode==='blade'&&(F.st==='grind'||F.st==='grindEnd'||(F.st==='quench'&&F.t>.8)||(F.st==='place'&&F.t<.3));if(!show)return;const a=F.st==='quench'?clamp((F.t-.8)/.3,0,1):F.st==='place'?1-F.t/.3:1;cx.save();cx.globalAlpha=a;
 cx.strokeStyle='#3a2a20';cx.lineWidth=6;cx.beginPath();cx.moveTo(g.x-26,L.fb-8);cx.lineTo(g.x,g.y);cx.lineTo(g.x+26,L.fb-8);cx.stroke();
 cx.fillStyle='#140e10';cx.beginPath();cx.arc(g.x,g.y,g.r+2,0,TAU);cx.fill();cx.fillStyle='#8a8278';cx.beginPath();cx.arc(g.x,g.y,g.r,0,TAU);cx.fill();cx.fillStyle='#6f685f';cx.beginPath();cx.arc(g.x,g.y,g.r-8,0,TAU);cx.fill();cx.fillStyle='#a8a096';cx.beginPath();cx.arc(g.x,g.y,g.r,PI*1.1,PI*1.6);cx.arc(g.x,g.y,g.r-4,PI*1.6,PI*1.1,true);cx.fill();
 for(let i=0;i<8;i++){const an=g.ang+i/8*TAU;cx.fillStyle='#5a544c';cx.save();cx.translate(g.x+Math.cos(an)*(g.r-14),g.y+Math.sin(an)*(g.r-14));cx.rotate(an);cx.fillRect(-4,-2,8,4);cx.restore()}
 cx.fillStyle='#3a2a20';cx.beginPath();cx.arc(g.x,g.y,6,0,TAU);cx.fill();cx.fillStyle='#2f5c7c';cx.fillRect(g.x-30,L.fb-22,60,8);cx.restore()}
function drawMold(){const f=F;if(f.mode!=='guard'||!(f.st==='heat'||f.st==='pour'||f.st==='set'))return;const x=L.AX+6,y=L.AY-12;cx.fillStyle='#140e10';cx.fillRect(x-46,y-12,92,20);cx.fillStyle='#3a3030';cx.fillRect(x-44,y-10,88,16);cx.fillStyle='#1a1414';cx.fillRect(x-38,y-5,76,6);cx.fillRect(x-6,y-9,12,14);
 if(MG.k==='hold'||f.st==='set'){const fill=f.st==='set'?1:clamp(MG.v/MG.zc,0,1),w=76*fill;cx.fillStyle='#ff8a2a';cx.fillRect(x-w/2,y-5,w,6);if(fill>.3){const h=14*clamp((fill-.3)/.7,0,1);cx.fillRect(x-6,y+5-h,12,h)}cx.fillStyle='#ffd8a0';cx.fillRect(x-w/2,y-5,w,1.5);glowAt(x,y,'#ff8a2a',50*fill+10,.4)}
 if(f.st!=='set'){const pt=MG.on?.95:.25,px=x+28,py=y-62;cx.save();cx.translate(px,py);cx.rotate(-pt);cx.fillStyle='#140e10';cx.fillRect(-15,-13,30,26);cx.fillStyle='#5a5a62';cx.fillRect(-13,-11,26,22);cx.fillStyle='#ff9a3a';cx.fillRect(-11,-11,22,5);cx.fillStyle='#fff2a0';cx.fillRect(-7,-11,10,2);cx.restore();glowAt(px,py-6,'#ff8a2a',30,.35);
  if(MG.on&&!MG.done){cx.strokeStyle='#ff9a3a';cx.lineWidth=4+Math.sin(NOW*40);cx.beginPath();cx.moveTo(px-14,py-4);cx.quadraticCurveTo(px-20,py+20,x,y-6);cx.stroke();cx.strokeStyle='#fff2a0';cx.lineWidth=1.5;cx.stroke()}}
 if(f.st==='set'&&f.t<.4){const k=f.t/.4,gy=lerp(y-90,y-4,eIn(k));spr(SPR.ico.gem[Math.min(f.tier,11)],x,gy,3,{ay:.5})}}
function drawForgePiece(){const f=F,m=f.mode,s=f.st,ax=L.AX,ay=L.AY,P=3;
 if(m==='blade'){
  if(s==='heat'){const k=eOut(clamp(f.t/.7,0,1)),x=lerp(L.FX,ax,k),y=lerp(L.fb-70,ay-5,k)-Math.sin(k*PI)*60;hotBar(x,y,30,10,-1)}
  else if(s==='strike'){const Lb=barLen(),th=barTh(),x0=ax-Lb/2,y=ay-th/2;hotBar(ax,y,Lb,th,f.n<5?MG.hot:-1);
   if(f.n<5&&MG.lock<=0){const mx=x0+MG.mk*Lb;cx.fillStyle='rgba(255,246,200,.5)';cx.fillRect(mx-1,ay-60,2,58-th);cx.fillStyle='#fff6c8';cx.beginPath();cx.moveTo(mx-5,y-th/2-9);cx.lineTo(mx+5,y-th/2-9);cx.lineTo(mx,y-th/2-3);cx.fill()}}
  else if(f.res){const img=swBig(f.res,null,null,'bl');let px=ax,py=ay-6,rot=PI/2,hot=0;
   if(s==='flash'){const k=clamp(f.t/.5,0,1),c=swCenter(img,img.fb,px,py,P,rot);spr(tintOf(img,'#ff9a3a'),c.x,c.y,P,{ax:.5,ay:.5,rot,sy:lerp(.4,1,eBack(k))});return}
   if(s==='quench'){const t=f.t;if(t<.45){const k=eOut(t/.45);px=lerp(ax,L.BX,k);py=lerp(ay-6,L.fb-100,k)-Math.sin(k*PI)*60;rot=lerp(PI/2,PI,k);hot=1}else if(t<.8){px=L.BX;py=L.fb-100+eIn(clamp((t-.45)/.15,0,1))*44;rot=PI;hot=1-clamp((t-.6)/.2,0,1)}else{const k=eOut(clamp((t-.8)/.3,0,1));px=lerp(L.BX,GS.x+14,k);py=lerp(L.fb-70,GS.y-GS.r-6,k)-Math.sin(k*PI)*40;rot=lerp(PI,PI/2+.1,k)}}
   else if(s==='grind'||s==='grindEnd'){px=GS.x+14;py=GS.y-GS.r-6+(MG.on?Math.sin(NOW*50):0);rot=PI/2+.1}
   else if(s==='place'){const k=eOut(clamp(f.t/.45,0,1));px=lerp(GS.x+14,ax,k);py=lerp(GS.y-GS.r-6,ay-6,k)-Math.sin(k*PI)*30}
   else if(s==='reveal'){const c=GR[f.step].c;glowAt(ax,ay-6,c,70+f.step*15,.25+f.step*.08)}
   else if(s==='rise'||s==='named'||s==='card')return;
   const c=swCenter(img,img.fb,px,py,P,rot);cx.save();if(s==='quench'&&f.t>.45&&f.t<.8){cx.beginPath();cx.rect(-500,-500,W+1000,L.fb-74+500);cx.clip()}spr(img,c.x,c.y,P,{ax:.5,ay:.5,rot});if(hot>0)spr(tintOf(img,'#ff8a2a'),c.x,c.y,P,{ax:.5,ay:.5,rot,a:hot*.85});if(s==='reveal')spr(tintOf(img,GR[f.step].c),c.x,c.y,P,{ax:.5,ay:.5,rot,a:clamp(.5-(NOW-f.slamT),0,.5)});cx.restore()}}
 else if(m==='guard'){if((s==='reveal1'||s==='card'||s==='set')&&f.res){const img=swBig(null,f.res,null,'gd'),c=swCenter(img,img.fg,ax+6,ay-6,4,0);spr(img,c.x,c.y,4,{ax:.5,ay:.5});if(s!=='set')glowAt(ax+6,ay-6,matCol('g',f.res.t),40,.3)}}
 else if(m==='handle'&&f.res){const hd=Object.assign({},f.res,{p:0}),img=swBig(null,null,hd,'hd'),c=swCenter(img,img.fh,ax+6,ay-8,4,PI/2);spr(img,c.x,c.y,4,{ax:.5,ay:.5,rot:PI/2})}
 else if(m.startsWith('enh')){if(!S.bl&&!S.gd&&!S.hd)return;const img=swBig(S.bl,S.gd,S.hd),p=SWP();let sh=0;if(s==='hold'&&MG.on)sh=Math.sin(NOW*60)*MG.v*2;if(s==='roll'&&f.kind==='bl')sh=Math.sin(NOW*50)*1.5;const c=swCenter(img,.5,p.x+sh,p.y,P,PI/2);
  if(S.bl&&S.bl.p>=15)glowAt(p.x,p.y,GR[S.bl.g].c,90,.18+.08*Math.sin(NOW*3));spr(img,c.x,c.y,P,{ax:.5,ay:.5,rot:PI/2,fl:f.out==='ok'?clamp(.6-f.t,0,.6):0,flc:'#fff6d0'});
  if(m==='enhbl'&&S.bl&&(s==='strike'||s==='roll')){const b=bladeBar();tapBar(b.x0,b.x1,b.y-12,MG.mk,MG.hot)}}}
function drawRays(x,y,c,a,rot){cx.save();cx.translate(x,y);cx.rotate(rot);cx.globalCompositeOperation='lighter';for(let i=0;i<12;i++){cx.rotate(TAU/12);const g=cx.createLinearGradient(0,0,0,-200);g.addColorStop(0,rgba(c,a));g.addColorStop(1,rgba(c,0));cx.fillStyle=g;cx.beginPath();cx.moveTo(0,0);cx.lineTo(-14,-200);cx.lineTo(14,-200);cx.fill()}cx.restore()}
function glowAt(x,y,c,r,a){if(!(r>0))return;cx.save();cx.globalCompositeOperation='lighter';const g=cx.createRadialGradient(x,y,0,x,y,r);g.addColorStop(0,rgba(c,clamp(a,0,1)));g.addColorStop(1,rgba(c,0));cx.fillStyle=g;cx.fillRect(x-r,y-r,r*2,r*2);cx.restore()}
const revY=()=>L.hud+(L.fb-L.hud)*.42;
function drawForge(dt){const fb=L.fb,top=L.hud,bg=SPR.fbg,s=3;cx.save();cx.beginPath();cx.rect(0,top,W,fb-top);cx.clip();cx.fillStyle='#24161a';cx.fillRect(0,top,W,fb-top);
 cx.save();cx.translate((R()-.5)*SHK*2,(R()-.5)*SHK*2);cx.translate(F.fx,F.fy);cx.scale(F.z,F.z);cx.translate(-F.fx,-F.fy);
 const by=fb-bg.height*s;cx.drawImage(bg,0,by,bg.width*s,bg.height*s);for(let y=by;y>top-200;y-=16*s)cx.drawImage(bg,0,0,120,16,0,y-16*s,360,16*s);
 drawFire();const fl=.8+.12*Math.sin(NOW*7)+.08*Math.sin(NOW*17.3);glowAt(L.FX,fb-70,'#ff7828',220*fl,.32);if(F.mode==='blade'&&(F.st==='strike'||F.st==='heat'))glowAt(L.AX,L.AY,'#ff8c32',170,.22);
 drawSmith(dt);spr(SPR.anvil,L.AX,fb-8,3);drawMold();drawGrindstone();drawForgePiece();spr(SPR.barrel,L.BX,fb-6,3);drawParts(PS);cx.restore();
 const vg=cx.createLinearGradient(0,fb-40,0,fb);vg.addColorStop(0,'rgba(0,0,0,0)');vg.addColorStop(1,'rgba(0,0,0,.35)');cx.fillStyle=vg;cx.fillRect(0,fb-40,W,40);
 if(F.st==='reveal'){const a=.15+F.step*.08+F.dark*.6,g=cx.createRadialGradient(F.fx,F.fy,40,F.fx,F.fy,320);g.addColorStop(0,'rgba(8,5,4,0)');g.addColorStop(1,'rgba(8,5,4,'+Math.min(.92,a+.3)+')');cx.fillStyle=g;cx.fillRect(0,top,W,fb-top)}
 if(F.mode==='blade'&&(F.st==='rise'||F.st==='named'||F.st==='card')&&F.res){const st=F.res.g,c=GR[st].c,y=revY(),k=F.st==='rise'?eOut(clamp(F.t/(F.fast?.35:.7),0,1)):1;cx.fillStyle='rgba(8,5,4,'+(.62*k)+')';cx.fillRect(0,top,W,fb-top);drawRays(W/2,y,c,(.12+st*.07)*k,NOW*.35);
  const img=swBig(F.res,S.gd,S.hd),bob=Math.sin(NOW*2)*4,yy=lerp(L.AY-6,y,k)+bob*k;glowAt(W/2,yy,c,60+st*12,.35+st*.08);spr(img,lerp(L.AX+6,W/2,k),yy,lerp(3,4,k),{ax:.5,ay:.5,rot:lerp(PI/2,PI/4,k)+Math.sin(NOW*1.3)*.06*k});
  text(GR[st].n,W/2,y+92,{size:14,bold:1,color:c,wave:st>=2?2:0});if(F.named)text(F.res.name,W/2,y-96,{size:F.res.g===4?24:18,bold:1,title:1,color:c,wave:F.res.g>=2?1.6:0,pop:F.res.g<4,drop:F.res.g===4,t0:F.nameT,gap:F.res.g===4?.12:.04,ol:1})}
 if(F.st==='ring'||F.st==='wrap'||(F.st==='roll'&&F.mode==='enhgd')){const r=60,x=F.fx,y=F.fy,win=MG.win||{perf:.1,ok:.3};cx.strokeStyle='rgba(20,14,16,.85)';cx.lineWidth=12;cx.beginPath();cx.arc(x,y,r,0,TAU);cx.stroke();cx.strokeStyle='#4a3a33';cx.lineWidth=6;cx.stroke();
  cx.strokeStyle='#d8c8b4';cx.lineWidth=6;cx.beginPath();cx.arc(x,y,r,MG.tg-win.ok,MG.tg+win.ok);cx.stroke();cx.strokeStyle='#ffd257';cx.beginPath();cx.arc(x,y,r,MG.tg-win.perf,MG.tg+win.perf);cx.stroke();
  cx.fillStyle='#fff6c8';cx.beginPath();cx.moveTo(x-7,y-r-16);cx.lineTo(x+7,y-r-16);cx.lineTo(x,y-r-7);cx.fill();
  let mp=ringPos();if(F.st==='roll'&&F.gp){const k=eOut(clamp(F.t/.3,0,1));mp={x:lerp(F.gp.x,x,k),y:lerp(F.gp.y,y-r+4,k)}}
  if(F.mode==='enhgd'){const gt=S.gd?S.gd.t:0;spr(SPR.ico.gem[Math.min(gt,11)],mp.x,mp.y,3,{ay:.5});glowAt(mp.x,mp.y,matCol('g',gt),18,.5)}else{cx.fillStyle='#140e10';cx.fillRect(mp.x-6,mp.y-6,12,12);cx.fillStyle=matCol('h',F.tier);cx.fillRect(mp.x-5,mp.y-5,10,10)}}
 if(MG.k==='hold'&&(F.st==='grind'||F.st==='pour'||F.st==='hold'||((F.st==='grindEnd'||F.st==='set'||F.st==='roll'||F.st==='done')&&F.t<.6))){const x=W-30,y0=top+34,y1=fb-24,hh=y1-y0,zc=MG.zc,zw=MG.zw;prect(x-12,y0-6,24,hh+12,'#0e0a09');cx.fillStyle='#3a2e2a';cx.fillRect(x-7,y0,14,hh);
  cx.fillStyle='#4f8a3c';cx.fillRect(x-7,y1-(zc+zw/2)*hh,14,zw*hh);cx.fillStyle='#8bd67f';cx.fillRect(x-7,y1-(zc+zw*.22)*hh,14,zw*.44*hh);
  const v=clamp(MG.v,0,1.05),fc=F.st==='grind'||F.st==='grindEnd'?'#d8e0e8':F.st==='pour'||F.st==='set'?'#ff9a3a':'#c89a6a';cx.fillStyle=fc;cx.fillRect(x-4,y1-v*hh,8,v*hh);cx.fillStyle='#fff';cx.fillRect(x-9,y1-v*hh-1,18,2);
  text(F.mode==='blade'?'날':F.mode==='guard'?'붓기':'조임',x,y0-16,{size:11,bold:1,color:'#f3e3c4'})}
 drawTexts(TX);cx.restore()}
function drawSwordCard(){const y=L.card,b=S.bl;panel(12,y,W-24,74);const img=swBig(S.bl,S.gd,S.hd);if(b&&b.g>=3)glowAt(84,y+37,GR[b.g].c,60,.18);
 if(b||S.gd||S.hd)spr(img,84,y+37+Math.sin(NOW*2)*1.5,2,{ax:.5,ay:.5,rot:PI/2});else spr(SPR.ham,84,y+37,3,{ax:.5,ay:.5,rot:PI/2});
 const st=stats();text(b?b.name+(b.p?' +'+b.p:''):'대장 망치',158,y+17,{size:14,bold:1,align:'left',color:b?GR[b.g].c:'#cfc7bb',wave:b&&b.g>=3?1:0});
 text('공격 '+fmt(st.atk)+' · 치명 '+Math.round(st.cc*100)+'% · 속도 '+st.aps.toFixed(1),158,y+38,{size:11,align:'left',color:'#b8a89a'});
 text('전투력 '+fmt(power())+'  ·  Lv '+S.lv,158,y+57,{size:11,bold:1,align:'left',color:'#ffe2a0'});if(S.charm)spr(SPR.ico.coal,W-30,y+56,2,{ay:.5})}
function costTxt(k,t){if(t<0)t=S.stage;if(k==='bl')return oreName(t)+' '+matN('o',t)+'/'+K.bCost;if(k==='gd')return matName('g',t)+' '+matN('g',t)+'/'+K.gCost+' · '+oreName(t)+' '+matN('o',t)+'/'+K.gOre;return matName('h',t)+' '+matN('h',t)+'/'+K.hCost+' · 금화 '+fmt(hGold(t))}
function drawForgeUI(){const m=F.mode,s=F.st;
 const tapSt=(m==='blade'&&s==='strike')||(m==='handle'&&s==='wrap')||(m==='enhbl'&&s==='strike')||(m==='enhgd'&&s==='ring')||s==='grind'||s==='pour'||s==='hold'||s==='reveal'||s==='rise'||s==='named';
 if(tapSt)hit('fArea',0,L.hud,W,H-L.hud,null,forgeTap);
 if(m==='idle'){drawSwordCard();[['bl','날'],['gd','가드'],['hd','손잡이']].forEach((a,i)=>{const k=a[0],p=S[k],x=12+i*114;button('tab'+k,x,L.tabs,108,46,a[1]+(p&&p.p?' +'+p.p:''),{sub:p?(k==='bl'?GR[p.g].n+' '+SHP[p.sh].n:qWord(p.q)+' '+matName(k==='gd'?'g':'h',p.t)):'없음',col:UI.tab===k?'#5a4030':'#2a211d',size:12,sel:UI.tab===k,cb:()=>{UI.tab=k},tc:UI.tab===k?'#fff4e6':'#bfb0a0',sc:p&&k==='bl'?GR[p.g].c:'#b8a89a'})});
  const k=UI.tab,ct=craftTier(k),p=S[k];button('craft',12,L.row,164,52,PNAME[k]+' 만들기',{sub:costTxt(k,ct),dis:ct<0,cb:()=>startCraft(k),dcb:()=>toast('재료가 모자라다. 사냥에서 모아 오자','#ffb0a0'),col:'#5a3e2e',glow:!p&&ct>=0,size:13});
  button('enh',184,L.row,164,52,ENHN[k],{sub:p?(p.p>=MAXP[k]?'최고 단계':'+'+p.p+' → +'+(p.p+1)):PNAME[k]+' 없음',dis:!p||p.p>=MAXP[k],cb:()=>enhOpen(k),col:'#3e4a56',size:13});
  button('hunt',12,L.go,W-24,56,'사냥 가기',{sub:stageDef(S.stage).n,cb:()=>go('hunt'),col:'#a8532a',size:16,glow:!!S.bl&&!S.tut.h1});
  if(!M){if(!S.bl&&craftTier('bl')>=0){if(UI.tab!=='bl')hintB('날부터 만들자',68,L.tabs-16,'t0');else hintB('광석으로 날을 만들자',94,L.row-16,'f1')}
   else if(S.bl&&!S.gd&&craftTier('gd')>=0&&!S.tut.cgd){if(UI.tab!=='gd')hintB('보석으로 가드를 만들자',180,L.tabs-16,'t1');else hintB('가드를 만들자',94,L.row-16,'f2')}
   else if(S.bl&&!S.hd&&craftTier('hd')>=0&&!S.tut.chd){if(UI.tab!=='hd')hintB('가죽으로 손잡이를',W-66,L.tabs-16,'t2');else hintB('손잡이를 만들자',94,L.row-16,'f3')}
   else if(S.bl&&!S.tut.h1)hintB('새 검으로 사냥하러 가자',W/2,L.go-4,'h1')}}
 else if(m==='blade'&&s==='pick'){const y=L.card;panel(12,y,W-24,L.go-y-8,{c:'#221a17'});text('날 모양을 고르자',W/2,y+18,{size:14,bold:1,color:'#ffe2a0',wave:1,pop:1,t0:F.t0p||(F.t0p=NOW)});
  SHPK.forEach((k,i)=>{const sh=SHP[k],bx=22+(i%2)*162,byy=y+36+Math.floor(i/2)*60;button('sh'+k,bx,byy,154,54,sh.n,{sub:sh.d,cb:()=>pickShape(k),col:'#4a3a33',size:13})});
  text(oreName(F.tier)+' '+K.bCost+'개를 쓴다',W/2,y+164,{size:11,color:'#b8a89a'});button('cancel',12,L.go,W-24,56,'취소',{cb:cancelCraft,col:'#3a2e2a'})}
 else if(m==='blade'&&(s==='heat'||s==='strike'||s==='flash')){const y=L.card;panel(12,y,W-24,L.go-y+50,{c:'#221a17'});text(s==='strike'?'빛나는 곳에 망치가 오면 탭':'달구는 중',W/2,y+18,{size:14,bold:1,color:'#ffe2a0',wave:1});
  for(let i=0;i<5;i++){const q=F.acc[i],x=W/2-64+i*32,yy=y+50;prect(x-11,yy-11,22,22,'#0e0a09');prect(x-9,yy-9,18,18,q===undefined?(i===F.n?'#4a3a33':'#2e2420'):q>=1?'#ffd257':q>.5?'#d8c8b4':'#5a4a44');if(q>=1)glowAt(x,yy,'#ffd257',18,.4)}
  const od=gradeOdds(effQ(F.acc),perfN(F.acc),S.charm>0),tg=[od[2]+od[3]+od[4],od[3]+od[4],od[4]],nm=['희귀 이상','영웅 이상','전설'];
  for(let i=0;i<3;i++){const gap=tg[i]-F.od[i];if(gap>.002&&!F['ok'+i+F.n]){F['ok'+i+F.n]=1;kick('od'+i,3)}F.od[i]+=gap*.15;const x=W/2-104+i*104,sc=spring('od'+i,0);text(nm[i],x,y+84,{size:11,color:'#b8a89a'});cx.save();cx.translate(x,y+106);cx.scale(sc,sc);text((F.od[i]*100).toFixed(F.od[i]<.1?1:0)+'%',0,0,{size:18,bold:1,color:GR[i+2].c,ol:1});cx.restore()}
  const pf=perfN(F.acc);text(pf>=5?'완벽 5번: 희귀 이상 보장 · 전설 1.5배':pf>=4?'완벽 4번: 고급 이상 보장 (5번이면 희귀 이상)':'완벽 '+pf+'번 · 4번이면 고급 이상 보장',W/2,y+136,{size:11,color:pf>=4?'#ffd257':'#9a8a80',wave:pf>=4?1:0});
  if(S.pity>=K.pity)text('이번엔 전설이 확실하다',W/2,y+156,{size:11,bold:1,color:'#ffd257',wave:1})}
 else if(m==='blade'||m==='guard'||m==='handle'){const y=L.card;panel(12,y,W-24,L.go-y+50,{c:'#221a17'});let t1='';
  if(s==='grind')t1='꾹 눌러 갈고, 초록 칸에서 떼기';else if(s==='grindEnd'||s==='place')t1='날을 세웠다';else if(s==='quench')t1='식히는 중';else if(s==='reveal'||s==='reveal1')t1='마무리 두드리기';else if(s==='rise'||s==='named')t1='완성!';
  else if(s==='heat')t1='쇳물을 녹이는 중';else if(s==='pour')t1='꾹 눌러 붓고, 초록 칸에서 떼기';else if(s==='set')t1='보석을 박는다';else if(s==='wrap')t1='가죽 끝이 위 홈에 올 때 탭 ('+F.n+'/3)';
  text(t1,W/2,y+22,{size:14,bold:1,color:'#ffe2a0',wave:1});if(F.msg)text(F.msg,W/2,y+48,{size:12,bold:1,color:'#f3e9dc',pop:1,t0:F.msgT||0});
  if(m==='blade'&&F.res)text(oreName(F.tier)+' '+SHP[F.sh].n,W/2,y+72,{size:11,color:'#b8a89a'});
  if(s==='reveal'||s==='rise'||s==='named')text('탭하면 빨리',W/2,y+100,{size:11,color:'#7a6a62'});
  if(s==='grind'&&!S.tut.g1)hintB('꾹 누르고 있다가 떼자',W/2,L.fb-30,'g1');if(s==='grindEnd')S.tut.g1=1}
 else drawEnhPanel()}
function drawEnhPanel(){const k=F.kind,p=S[k],s=F.st,y=L.card;panel(12,y,W-24,L.go-y-8,{c:'#221a17'});
 if(!p){text('날이 부서졌다',W/2,y+30,{size:16,bold:1,color:'#ffb0a0',wave:1});text('망치를 들고 새로 만들자',W/2,y+56,{size:11,color:'#b8a89a'});button('enhX',12,L.go,W-24,56,'닫기',{cb:()=>fReset('idle'),col:'#3a2e2a'});return}
 text(ENHN[k]+' · '+(k==='bl'?p.name:PNAME[k]),W/2,y+18,{size:13,bold:1,color:'#ffe2a0'});
 if(p.p>=MAXP[k])text('최고 단계 +'+p.p,W/2,y+52,{size:18,bold:1,color:'#ffd257',wave:1.5});
 else{text('+'+p.p+'  →  +'+(p.p+1),W/2,y+46,{size:20,bold:1,color:'#fff4e6'});
  if(k==='bl'){text('성공 '+EP[p.p]+'%  ·  완벽하게 치면 +15%',W/2,y+74,{size:11,bold:1,color:'#8bd67f'});text(EB(p.p)?'실패하면 '+EB(p.p)+'% 확률로 부서짐 (완벽이면 절반)':'+8 전까지는 부서지지 않는다',W/2,y+94,{size:11,color:EB(p.p)?'#ff8a7a':'#8a7a70'})}
  else if(k==='gd'){text('보석이 위 홈에 맞으면 성공',W/2,y+74,{size:11,bold:1,color:'#8bd67f'});text('빗나가면 보석이 깨진다 · 홈은 점점 좁아진다',W/2,y+94,{size:11,color:'#ff8a7a'})}
  else{text('초록 칸에서 떼면 성공',W/2,y+74,{size:11,bold:1,color:'#8bd67f'});text('덜 조이거나 지나치면 가죽이 끊어진다',W/2,y+94,{size:11,color:'#ff8a7a'})}
  const c=enhCost(k);text('금화 '+fmt(c.g)+(c.m?' · '+matName(c.m,c.t)+' '+c.n+' (가진 것 '+matN(c.m,c.t)+')':''),W/2,y+116,{size:11,color:S.gold>=c.g&&(!c.m||matN(c.m,c.t)>=c.n)?'#ffe2a0':'#ff8a7a'})}
 const verb={bl:'두드리기',gd:'박기',hd:'조이기'}[k];
 if(s==='ready'||s==='done'){if(p.p<MAXP[k])button('enhGo',12,L.go-62,W-24,54,s==='done'?'한 번 더 '+verb:verb,{cb:enhGo,col:'#a8532a',dis:!canEnh(k),dcb:()=>toast(enhLack(k),'#ff8a6a'),size:15});button('enhX',12,L.go,W-24,56,'닫기',{cb:()=>fReset('idle'),col:'#3a2e2a'})}
 else text(s==='strike'?'빛나는 곳에서 탭!':s==='ring'?'보석이 위 홈에 올 때 탭!':s==='hold'?'꾹 눌러 조이고, 초록 칸에서 떼기':'',W/2,L.go-20,{size:14,bold:1,color:'#ffe2a0',wave:1.2})}
function drawHUD(){panel(6,4,W-12,40,{c:'#221a17'});const t=S.stage,b=UI.bump;
 const item=(img,x,v,bk,col)=>{const k=b[bk]||0;spr(img,x,24,2,{ay:.5,sx:1+k*.5,sy:1+k*.5});text(fmt(v),x+10,24,{size:12,bold:1,align:'left',color:col})};
 item(SPR.ico.coin,HUDX.g,UI.dg,'g','#ffe2a0');item(SPR.ico.ore[Math.min(t,11)],HUDX.o,UI.dm[0],'o',mix(oreCol(t),'#ffffff',.35));item(SPR.ico.gem[Math.min(t,11)],HUDX.gm,UI.dm[1],'gm',mix(matCol('g',t),'#ffffff',.3));item(SPR.ico.hide[Math.min(t,11)],HUDX.h,UI.dm[2],'h',mix(matCol('h',t),'#ffffff',.45));
 if(SCN==='forge'){button('quest',268,7,40,34,'',{cb:()=>{ensureDaily();openM('quest')},col:'#33281f'});spr(SPR.ico.scroll,288,24,2,{ay:.5});if(qReady()){cx.fillStyle='#ff5a4a';cx.fillRect(300,9,6,6)}}
 else button('spd',268,7,40,34,'×'+S.set.spd,{cb:()=>{S.set.spd=S.set.spd>=3?1:S.set.spd+1;save()},col:S.set.spd>1?'#6a4a2a':'#33281f',size:13,tc:S.set.spd>1?'#ffd257':'#f3e9dc'});
 button('set',312,7,40,34,'',{cb:()=>openM('set'),col:'#33281f'});spr(SPR.ico.gear,332,24,2,{ay:.5})}

// ===== Drawing: hunt =====
function layer(img,f,bottom){const s=3,w=img.width*s,h=img.height*s;let ox=-((Hn.cam*f)%w);if(ox>0)ox-=w;for(let x=ox;x<W;x+=w)cx.drawImage(img,Math.round(x),Math.round(bottom-h),w,h)}
const STARS=[];for(let i=0;i<60;i++)STARS.push([Math.random(),Math.random(),Math.random()]);
function drawHuntBG(){const d=Hn.def,bg=SPR.bg[Math.min(Hn.t,6)],top=L.hud,gy=L.gy,hz=gy-50,n=9;for(let i=0;i<n;i++){const t=i/(n-1),c=t<.5?mix(d.sky[0],d.sky[1],t*2):mix(d.sky[1],d.sky[2],(t-.5)*2);cx.fillStyle=c;const y0=top+(hz-top)*i/n;cx.fillRect(0,Math.floor(y0),W,Math.ceil((hz-top)/n)+1)}cx.fillStyle=d.sky[2];cx.fillRect(0,hz,W,60);
 if(d.deco==='star'||d.deco==='tree'||d.deco==='rock'||d.deco==='ice'){for(const s of STARS){const tw2=.5+.5*Math.sin(NOW*2+s[2]*20);cx.globalAlpha=(d.deco==='star'?.9:.4)*tw2;cx.fillStyle='#e8ecff';const sz=s[2]>.85?2:1;cx.fillRect(Math.floor((s[0]*W-Hn.cam*.03%W+W)%W),top+s[1]*(hz-top)*.8,sz,sz)}cx.globalAlpha=1}
 if(d.deco==='grass'||d.deco==='bone'){glowAt(W*.72,top+(hz-top)*.55,d.sky[2],70,.35);cx.fillStyle=shade(d.sky[2],.3);cx.beginPath();cx.arc(W*.72,top+(hz-top)*.55,22,0,TAU);cx.fill()}
 layer(bg.far,.12,gy-30);layer(bg.mid,.4,gy+4);layer(bg.near,1,gy-12+bg.near.height*3)}
function drawEnemy(e){const s=e.spr,f=Math.floor(e.ph*(e.fly?8:5))%2,img=s.fr[f],P=PH,base=L.gy;let y=base,sx=1,sy=1,rot=0,ox=0;
 if(e.fly)y=base-60-Math.sin(e.ph*3)*7-(e.boss?26:0);
 const sp=Math.sin(e.ph*(e.d.a==='slime'?6:4));if(e.d.a==='slime'){sx=1+sp*.05;sy=1-sp*.05}else if(!e.fly)y+=Math.abs(sp)*-1.5;
 const mv=e.mv,tele=!!(mv&&mv.ph==='tele'&&TELE[mv.k]);
 if(e.wind>0){rot=.12;ox=6}if(e.lunge>0)ox=-e.lunge*10;if(e.fl>0){sx*=1+e.fl*.12;sy*=1-e.fl*.12}if(e.stun>0)rot=Math.sin(NOW*30)*.05;
 if(mv&&mv.k==='charge'){if(mv.ph==='tele2'){rot=.18;sy*=.9;ox=Math.sin(NOW*40)*2}else if(mv.ph==='dash'){rot=-.12;sx*=1.12}}
 if(tele){sy*=1+.06*Math.sin(NOW*14);if(mv.k==='breath')sx*=1+.05*(mv.t/K.tele)}
 let a=1;if(e.dead){const k=clamp(e.dead/.3,0,1);sx*=1+k*.4;sy*=1+k*.4;a=1-k}if(e.flee)a=clamp(e.fa,0,1);
 cx.globalAlpha=.3*a;cx.fillStyle='#000';cx.beginPath();cx.ellipse(e.x,base+2,img.width*P*.32,4,0,0,TAU);cx.fill();cx.globalAlpha=1;
 const cyy=y-img.height*P*.5;
 if(e.boss){glowAt(e.x,cyy,e.d.glow||e.d.c,img.width*P*.7,.18+(tele?.25+.2*Math.sin(NOW*20):0));if(e.d.a==='orb'){cx.save();cx.translate(e.x,cyy);cx.rotate(NOW*.6);cx.scale(1,.32);cx.strokeStyle=rgba(e.d.glow,.7);cx.lineWidth=4;cx.beginPath();cx.arc(0,0,img.width*P*.62,0,TAU);cx.stroke();cx.restore()}
  if(e.heat>0)glowAt(e.x,cyy,'#ff6a2a',img.width*P*.6,e.heat*.6)}
 const tc=tele?(mv.k==='freeze'?'#9fe0ff':mv.k==='laser'?'#c8b0ff':'#ff5a3a'):e.heat>.5?'#ff6a2a':'#ffffff';
 spr(img,e.x+ox,y,P,{flip:true,sx,sy,rot,a,fl:e.dead?1:Math.max(e.fl,tele?(.35+.3*Math.sin(NOW*24)):e.heat>.5?(e.heat-.5)*1.2:0),flc:(tele||e.heat>.5)&&!e.fl?tc:'#ffffff'});
 if(tele||(mv&&mv.ph==='tele2')){const yy=y-img.height*P-14+Math.sin(NOW*12)*2;text('!',e.x,yy,{size:22,bold:1,color:tele?tc:'#ffe08a',ol:1})}
 if(e.shield>0){cx.save();cx.globalAlpha=.55+.2*Math.sin(NOW*10);cx.strokeStyle='#9fc8ff';cx.lineWidth=5;cx.beginPath();cx.arc(e.x,cyy,img.width*P*.5,PI*.65,PI*1.35);cx.stroke();cx.restore()}
 if(e.shards>0)for(let i=0;i<e.shards;i++){const an=NOW*1.8+i/e.shards*TAU,r=img.width*P*.55,px=e.x+Math.cos(an)*r,py=cyy+Math.sin(an)*r*.5;cx.fillStyle='#140e10';cx.fillRect(px-6,py-8,12,16);cx.fillStyle='#c8b0ff';cx.beginPath();cx.moveTo(px,py-7);cx.lineTo(px+5,py);cx.lineTo(px,py+7);cx.lineTo(px-5,py);cx.fill();glowAt(px,py,'#c8b0ff',14,.4)}
 if(e.heat>0&&!e.dead){const w=70;bar(e.x-w/2,y-img.height*P-24,w,5,e.heat,e.heat>.7?'#ff4a2a':'#ffb03a')}
 if(mv&&mv.k==='laser'&&(tele||mv.ph==='beam')){cx.save();cx.strokeStyle=mv.ph==='beam'?'#ffffff':'rgba(200,176,255,'+(.3+.4*(mv.t/K.tele))+')';cx.lineWidth=mv.ph==='beam'?10:2;cx.beginPath();cx.moveTo(e.x-10,cyy);cx.lineTo(Hn.x+10,L.gy-60);cx.stroke();if(mv.ph==='beam'){cx.strokeStyle='#c8b0ff';cx.lineWidth=18;cx.globalAlpha=.4;cx.stroke()}cx.restore()}
 if(!e.boss&&!e.dead&&e.hp<e.mhp){const w=30;bar(e.x-w/2,y-img.height*P-6,w,3,e.hp/e.mhp,'#e8584a')}}
function drawHeroHunt(dt){const P=PH,tier=Math.min(S.stage,6),moved=Hn.x-HV.lx;HV.lx=Hn.x;if(moved>.01&&!Hn.dash){HV.ft+=moved/9;HV.f=Math.floor(HV.ft)%4}else{HV.f=1;HV.ft=0}
 const img=SPR.hero[tier][HV.f],sw=Hn.sw,pk=sw?sw.pose:shapeOf(S.bl).pose,x=Hn.x,b=(HV.f===1||HV.f===3)?-1:0;let y=L.gy,rotB=0,a=1;
 if(Hn.dead){const k=clamp(Hn.deadT/.6,0,1);rotB=-k*1.3;a=1-clamp((Hn.deadT-1)/.6,0,1)*.4}
 const pose=swPose(sw,pk);y+=pose.oy||0;const bx=x+(pose.ox||0)*.3,br=sw?0:Math.sin(NOW*3)*.02;
 if(!HV.sc)HV.sc=[];scarfUpd(HV.sc,bx-4,y-(16-b)*P,dt,moved>.01?1.4:.4);
 cx.save();cx.translate(bx,y);cx.rotate(rotB);cx.translate(-bx,-y);cx.globalAlpha=a;scarfDraw(HV.sc,SCARF[tier],P*.8);
 spr(img,bx,y,P,{sy:1+br,sx:1-br,fl:Hn.hfl,flc:'#ff6a5a'});
 const shX=bx,shY=y-(14-b)*P,wimg=swSmall(S.bl,S.gd,S.hd),aa=pose.rot*.5+.75,hand={x:shX+Math.sin(aa)*P*3.5+(pose.ox||0)*.7,y:shY-Math.cos(aa)*P*3.5+P*1.5};
 if(pose.sm&&HV.pr!==undefined){const len=wimg.pv[1]*wimg.height*P,a0=Math.min(HV.pr,pose.rot)-PI/2,a1=Math.max(HV.pr,pose.rot)-PI/2;cx.save();cx.globalCompositeOperation='lighter';cx.fillStyle=rgba(S.bl?GR[S.bl.g].c:'#ffffff',.45);cx.beginPath();cx.arc(hand.x,hand.y,len,a0,a1);cx.arc(hand.x,hand.y,len*.45,a1,a0,true);cx.fill();cx.restore()}
 HV.pr=pose.rot;
 cx.strokeStyle='#140e10';cx.lineWidth=P*2+2;cx.lineCap='square';cx.beginPath();cx.moveTo(shX,shY);cx.lineTo(hand.x,hand.y);cx.stroke();cx.strokeStyle=HPAL[tier].body;cx.lineWidth=P*2;cx.beginPath();cx.moveTo(shX,shY);cx.lineTo(hand.x,hand.y);cx.stroke();
 if(S.bl&&(S.bl.g>=3||S.bl.p>=10))glowAt(hand.x+Math.sin(pose.rot)*24,hand.y-Math.cos(pose.rot)*24,GR[S.bl.g].c,28,.25+.1*Math.sin(NOW*4));
 spr(wimg,hand.x,hand.y,P,{ax:wimg.pv[0],ay:wimg.pv[1],rot:pose.rot});cx.fillStyle='#efc39a';cx.fillRect(hand.x-P,hand.y-P,P*2,P*2);cx.restore();
 if(Hn.frz>0){const w=72,h=118,x0=bx-w/2,y0=L.gy-h;cx.save();cx.globalAlpha=.55;cx.fillStyle='#bfefff';cx.fillRect(x0,y0,w,h);cx.globalAlpha=.9;cx.strokeStyle='#effcff';cx.lineWidth=3;cx.strokeRect(x0,y0,w,h);cx.strokeStyle='#7fc8e0';cx.lineWidth=1.5;cx.beginPath();cx.moveTo(x0+10,y0+20);cx.lineTo(x0+30,y0+50);cx.lineTo(x0+20,y0+80);cx.moveTo(x0+50,y0+10);cx.lineTo(x0+44,y0+44);cx.stroke();cx.restore();hit('ice',bx-Hn.cam-60,L.gy-150,120,160,null,breakIce)}
 if(S.bl&&S.bl.g===4&&R()<.25)part(PW,{x:hand.x+rr(-10,10),y:hand.y-rr(0,40),vx:rr(-10,10),vy:rr(-40,-10),l:rr(.4,.9),s:2,c:'#ffe08a',k:'sq',sh:1})}
function swPose(s,ty){const REST={sword:.55,axe:.5,dagger:1.25,spear:1.5,hammer:.5}[ty];if(!s)return{rot:REST+Math.sin(NOW*3)*.04,ox:0,oy:0};const k=s.t/s.d;
 if(s.k==='sweep'){const a=eOut(clamp(k/.85,0,1));return{rot:REST-1+a*TAU*1.25,ox:0,oy:-Math.sin(clamp(k,0,1)*PI)*10,sm:k<.85}}
 if(ty==='dagger'||ty==='spear'){if(s.k==='heavy'&&ty==='dagger'){const n=5,ph=(k*n)%1,j=Math.floor(k*n)%2;return{rot:1.57+(j?.3:-.25),ox:Math.sin(ph*PI)*16,oy:j?-4:3}}
  const pk=s.k==='heavy'?30:14,p=k<.4?-eOut(k/.4)*5:k<.6?lerp(-5,pk,eOut((k-.4)/.2)):lerp(pk,0,eOut((k-.6)/.4));return{rot:1.57+(s.alt?.12:-.1),ox:p,oy:0}}
 const hv=ty==='axe'||ty==='hammer',W0=s.k==='heavy'?(hv?-2.6:-2.3):(hv?-1.9:-1.4),W1=s.k==='heavy'?2.6:2.2;let rot,sm=false,oy=0;
 if(k<.42)rot=lerp(REST,W0,eOut(k/.42));else if(k<.6){rot=lerp(W0,W1,(k-.42)/.18);sm=true}else rot=lerp(W1,REST,eOut((k-.6)/.4));
 if(s.k==='heavy'&&hv)oy=-Math.sin(clamp(k/.6,0,1)*PI)*24;return{rot,ox:s.k==='heavy'?Math.sin(clamp(k,0,1)*PI)*8:0,oy,sm}}
function drawVein(v){const img=SPR.vein[Math.min(Hn.t,6)],sh=v.sh>0?Math.sin(NOW*60)*v.sh*3:0,k=v.dead?clamp(v.dead/.3,0,1):0;spr(img,v.x+sh,L.gy+2,PH,{a:1-k,sx:1+k*.3,sy:1-k*.5});
 if(!v.dead){cx.strokeStyle='#140e10';cx.lineWidth=2;for(let i=0;i<v.hits;i++){cx.beginPath();cx.moveTo(v.x-12+i*10,L.gy-40);cx.lineTo(v.x-6+i*9,L.gy-24);cx.lineTo(v.x-12+i*10,L.gy-10);cx.stroke()}
  const pu=(NOW*1.4+v.id*.3)%1;cx.globalAlpha=(1-pu)*.6;cx.strokeStyle='#fff6c8';cx.lineWidth=2;cx.beginPath();cx.arc(v.x,L.gy-24,24+pu*18,0,TAU);cx.stroke();cx.globalAlpha=1;
  hit('vein'+v.id,v.x-Hn.cam-44,L.gy-72,88,84,null,()=>tapVein(v))}}
function ambient(dt){const d=Hn.def,a=d.amb,top=L.hud,gy=L.gy;if(R()>dt*14)return;
 if(a==='pollen')part(PS,{x:rr(0,W),y:rr(top+40,gy),vx:rr(-8,8),vy:rr(-6,6),l:rr(2,4),s:2,c:'#f6e08a',k:'sq',a:.7});
 else if(a==='leaf')part(PS,{x:rr(0,W+40),y:top+rr(0,40),vx:rr(-40,-10),vy:rr(20,40),l:rr(4,7),s:3,c:pick(['#7fae4a','#c9a24a','#5f8a3c']),k:'sq'});
 else if(a==='dust')part(PS,{x:W+10,y:rr(top+60,gy),vx:rr(-60,-30),vy:rr(-4,4),l:rr(4,7),s:2,c:'#b8bccc',k:'sq',a:.5});
 else if(a==='ember')part(PS,{x:rr(0,W),y:gy+10,vx:rr(-10,10),vy:rr(-50,-20),l:rr(3,6),s:2,c:pick(['#ff8a3a','#ffd257']),k:'sq'});
 else if(a==='snow')part(PS,{x:rr(0,W+60),y:top,vx:rr(-30,-10),vy:rr(25,45),l:rr(6,10),s:pick([2,3]),c:'#effcff',k:'sq'});
 else if(a==='ash')part(PS,{x:rr(0,W),y:top,vx:rr(-10,10),vy:rr(15,30),l:rr(6,9),s:2,c:'#8a7a78',k:'sq',a:.6});
 else if(a==='star'&&R()<.04)part(PS,{x:rr(W*.3,W+40),y:top+rr(0,60),vx:-260,vy:90,l:.7,s:2,c:'#e8ecff',k:'spk'})}
function drawHunt(dt){const top=L.hud,vb=L.vb;cx.save();cx.beginPath();cx.rect(0,top,W,vb-top);cx.clip();cx.save();cx.translate((R()-.5)*SHK*2,(R()-.5)*SHK*2);
 drawHuntBG();drawParts(PS);cx.save();cx.translate(-Math.round(Hn.cam*10)/10,0);for(const v of Hn.veins)drawVein(v);
 const es=Hn.en.slice().sort((a,b)=>(b.boss?1:0)-(a.boss?1:0));for(const e of es)drawEnemy(e);drawHeroHunt(dt);drawParts(PW);drawTexts(TW);cx.restore();cx.restore();
 if(Hn.intro>0){cx.fillStyle='rgba(0,0,0,'+(Math.min(1,Hn.intro)*.45)+')';cx.fillRect(0,top,W,vb-top)}
 const b=Hn.boss;if(b&&Hn.bm&&!b.dead&&Hn.intro<1.2){panel(20,top+8,W-40,36,{c:'#221a17'});text(b.d.n+(b.shield>0?' · 방패':b.shards>0?' · 파편 '+b.shards:b.stun>0?' · 기절':''),W/2,top+20,{size:11,bold:1,color:'#ffb0a0',wave:.8});bar(32,top+30,W-64,7,b.hp/b.mhp,b.vul>0?'#ffd257':'#e8584a')}
 const hy=b&&Hn.bm?top+52:top+10;text('Lv'+S.lv,12,hy+6,{size:11,bold:1,align:'left',color:'#ffe08a'});bar(46,hy+2,140,8,Hn.hp/Hn.mhp,Hn.hp/Hn.mhp<.3?'#ff5a4a':'#e8584a');text(fmt(Hn.hp)+' / '+fmt(Hn.mhp),192,hy+6,{size:11,align:'left',color:'#e8d8cc'});
 if(!Hn.bm)text(Hn.def.n,W-12,hy+6,{size:11,bold:1,align:'right',color:'#f3e3c4',wave:.6});
 if(Hn.frz>0)text('연타해서 얼음을 깨자!',W/2,(top+vb)/2-60,{size:16,bold:1,color:'#dff6ff',wave:2,shake:1,ol:1});
 if(Hn.dead&&Hn.deadT>.5)text('쓰러졌다',W/2,(top+vb)/2-40,{size:24,bold:1,title:1,color:'#ff8a7a',pop:1,t0:0,wave:1,ol:1});
 drawTexts(TX);cx.restore();
 panel(0,vb,W,H-vb+4,{c:'#1e1715'});const need=needN(),ready=S.prog>=need;
 if(Hn.bm)text(Hn.boss&&Hn.boss.dead?'보스 처치!':'보스와 싸우는 중',W/2,vb+30,{size:14,bold:1,color:'#ffb0a0',wave:1});
 else if(ready)button('boss',16,vb+12,W-32,40,'보스에게 도전',{cb:bossStart,col:'#9a3a2a',glow:1,size:14});
 else{text('보스까지',20,vb+24,{size:11,align:'left',color:'#b8a89a'});text(S.prog+' / '+need,W-20,vb+24,{size:11,bold:1,align:'right',color:'#f3e3c4'});bar(20,vb+36,W-40,8,S.prog/need,'#c8763a')}
 const cy=vb+108;button('back',14,cy-24,92,48,'대장간',{cb:leaveHunt,col:'#3a2e2a',size:12});
 skillBtn(0,W-152,cy,40,'강타');skillBtn(1,W-56,cy+6,32,'휩쓸기');
 if(Hn.mh&&!(Hn.mh.k==='freeze'&&Hn.frz>0)){const h=MVH[Hn.mh.k];hintB(h[0],h[1]?W-56:W-152,cy-(h[1]?54:60),'mv'+Hn.mh.k)}
 const fv=Hn.veins.find(v=>!v.dead&&v.x-Hn.cam<W-30);if(fv&&!S.tut.v1&&!Hn.bm)hintB('광맥을 탭해서 캐자',fv.x-Hn.cam,L.gy-104,'v1')}
function skillBtn(i,x,y,r,label){const cd=Hn.cd[i],mx=i?K.cd1:K.cd0,id='sk'+i,pr=PT.down&&PT.id===id,sc=spring(id,pr),rd=cd<=0&&Hn.frz<=0;if(rd&&HV['r'+i]===0){kick(id,6);HV['r'+i]=1}if(!rd)HV['r'+i]=0;
 const b=Hn.boss,mv=b&&b.mv,callH=i===0&&b&&!b.dead&&((mv&&mv.ph==='tele'&&TELE[mv.k])||(mv&&mv.k==='charge'&&mv.ph==='dash')||b.shield>0),callS=i===1&&b&&!b.dead&&(b.heat>.5||b.shards>0||Hn.en.some(e=>e.mini&&!e.dead));
 cx.save();cx.translate(x,y);cx.scale(sc,sc);cx.fillStyle='#0b0807';cx.beginPath();cx.arc(0,4,r,0,TAU);cx.fill();cx.fillStyle=rd?(i?'#3e5a6a':'#a8532a'):'#3a302c';cx.beginPath();cx.arc(0,0,r,0,TAU);cx.fill();cx.fillStyle=rd?shade(i?'#3e5a6a':'#a8532a',.2):'#4a3e38';cx.beginPath();cx.arc(0,0,r-4,PI*1.1,PI*1.9);cx.arc(0,0,r-8,PI*1.9,PI*1.1,true);cx.fill();
 if(!rd){cx.fillStyle='rgba(0,0,0,.45)';cx.beginPath();cx.moveTo(0,0);cx.arc(0,0,r,-PI/2,-PI/2+TAU*(cd/mx));cx.fill();text(Hn.frz>0?'얼음':Math.ceil(cd),0,0,{size:Hn.frz>0?11:16,bold:1,color:'#cfc7bb'})}else{if(callH||callS){cx.strokeStyle='#fff6c8';cx.lineWidth=3;cx.beginPath();cx.arc(0,0,r+4+Math.sin(NOW*20)*2,0,TAU);cx.stroke()}text(label,0,0,{size:i?11:14,bold:1})}
 cx.restore();hit(id,x-r,y-r,r*2,r*2,null,()=>skill(i))}

// ===== Modals =====
function drawModal(){if(!M)return;const pin=clamp((NOW-M.t0)/.25,0,1),pout=M.cl?clamp((NOW-M.cl)/.18,0,1):0,vis=eOut(pin)*(1-pout);
 cx.fillStyle='rgba(6,4,3,'+(.55*vis)+')';cx.fillRect(0,0,W,H);hit('blk',0,0,W,H,null);cx.save();const sc=lerp(.86,1,eBack(pin))*(1-pout*.08);cx.globalAlpha=vis;cx.translate(W/2,H/2);cx.scale(sc,sc);cx.translate(-W/2,-H/2);
 ({card:mCard,quest:mQuest,set:mSet,sum:mSum,stage:mStage}[M.k])();cx.restore()}
const mY=()=>H/2-170;
function mCard(){const k=F.kind,p=F.res;if(!p)return;const y=H-336,ph=306;panel(14,y,W-28,ph);const cur=S[k];
 const np=power(k==='bl'?p:S.bl,k==='gd'?p:S.gd,k==='hd'?p:S.hd),op=power(),df=np-op;
 if(k==='bl'){const c=GR[p.g].c;text('새 날 · '+GR[p.g].n,W/2,y+20,{size:11,bold:1,color:c,wave:1});text(p.name,W/2,y+42,{size:18,bold:1,title:1,color:c,wave:p.g>=2?1.2:0});text(oreName(p.t)+' '+SHP[p.sh].n+' · '+SHP[p.sh].d,W/2,y+64,{size:11,color:'#b8a89a'});
  text('공격 '+fmt(bladeAtk(p))+'  ·  날카로움 '+Math.round(p.q*100)+'%',W/2,y+84,{size:11,bold:1,color:'#f3e9dc'});let ty=y+102;for(const t of p.tr){const T=TR.find(x=>x.k===t.k);text(T.n+' · '+T.d(t.v),W/2,ty,{size:11,color:'#9fd0ff'});ty+=16}}
 else{const c=matCol(k==='gd'?'g':'h',p.t);text('새 '+PNAME[k]+' · 품질 '+qWord(p.q),W/2,y+20,{size:11,bold:1,color:c,wave:1});text(p.name,W/2,y+42,{size:18,bold:1,title:1,color:mix(c,'#ffffff',.3)});text('품질 '+Math.round(p.q*100)+'%',W/2,y+64,{size:11,color:'#b8a89a'});
  if(k==='gd'){const g=gdStat(p);text('치명 확률 +'+Math.round(g.cc*100)+'%  ·  치명 피해 +'+Math.round(g.cm*100)+'%',W/2,y+86,{size:12,bold:1,color:'#f3e9dc'})}else text('공격 속도 +'+Math.round(hdStat(p)*100)+'%',W/2,y+86,{size:12,bold:1,color:'#f3e9dc'})}
 text('검 전투력',W/2-46,y+140,{size:11,color:'#b8a89a'});text(fmt(np),W/2+14,y+140,{size:16,bold:1,color:'#ffe2a0',align:'left'});
 text((df>=0?'▲ ':'▼ ')+fmt(Math.abs(df)),W/2,y+162,{size:12,bold:1,color:df>=0?'#8bd67f':'#ff8a7a',wave:df>0?1:0});text('지금: '+(cur?cur.name+(cur.p?' +'+cur.p:''):'없음')+' ('+fmt(op)+')',W/2,y+182,{size:11,color:'#8a7a70'});
 if(cur&&cur.p)text('지금 것은 강화 +'+cur.p+' · 바꾸면 녹는다',W/2,y+200,{size:11,color:'#ff9a7a'});
 button('eq',26,y+ph-70,(W-60)/2,54,'장착',{cb:equipNew,col:df>=0?'#a8532a':'#5a3e2e',sub:cur?'쓰던 것은 녹임':'',glow:df>0});button('melt',34+(W-60)/2,y+ph-70,(W-60)/2,54,'녹이기',{cb:meltNew,col:'#3e4a56',sub:'금화 +'+fmt(meltV(k,p))})}
function mQuest(){const y=mY(),ph=340;panel(20,y,W-40,ph);text('오늘의 의뢰',W/2,y+24,{size:16,bold:1,title:1,color:'#ffe2a0',wave:1.2,pop:1,t0:M.t0});const r=qReward();
 S.daily.q.forEach((q,i)=>{const yy=y+56+i*72,def=QP.find(x=>x.k===q.k);panel(32,yy,W-64,62,{c:'#241c19'});text(def.d(q.t),44,yy+16,{size:11,bold:1,align:'left',color:q.c?'#8a7a70':'#f3e9dc'});bar(44,yy+32,150,6,q.p/q.t,q.p>=q.t?'#8bd67f':'#c8763a');text(q.p+'/'+q.t,200,yy+35,{size:11,align:'left',color:'#b8a89a'});text('금화 '+fmt(r.g)+' · 광석 '+r.o+' · 보석 '+r.gm,44,yy+50,{size:11,align:'left',color:'#a89070'});
  if(q.c)text('받음',W-62,yy+31,{size:11,bold:1,color:'#8a7a70'});else button('q'+i,W-110,yy+14,62,34,'받기',{dis:q.p<q.t,cb:()=>claimQ(i),col:'#a8532a',size:12,glow:q.p>=q.t})});
 const yy=y+56+3*72;spr(SPR.ico.coal,46,yy+12,2,{ay:.5});text(S.daily.b?'행운 숯을 받았다':'셋 다 끝내면 행운 숯 (다음 날 제작 전설 3배)',62,yy+12,{size:11,align:'left',color:'#ffd257'});text('가진 숯 '+S.charm+'개 · 내일 새 의뢰',W/2,yy+34,{size:11,color:'#8a7a70'});
 button('qX',W-62,y+8,34,30,'×',{cb:closeM,col:'#3a2e2a',size:14})}
let RST=0;
function mSet(){const y=mY()-20,ph=380;panel(30,y,W-60,ph);text('설정',W/2,y+24,{size:16,bold:1,color:'#ffe2a0'});const rows=[['효과음','sfx'],['음악','mus'],['화면 흔들림','shk']];
 rows.forEach((r,i)=>{const yy=y+50+i*48;text(r[0],52,yy+18,{size:12,bold:1,align:'left'});button('t'+r[1],W-132,yy,90,38,S.set[r[1]]?'켬':'끔',{cb:()=>{S.set[r[1]]=S.set[r[1]]?0:1;AU.vol();save()},col:S.set[r[1]]?'#5a7a3a':'#3a2e2a',size:12})});
 const yy=y+50+3*48;text('사냥 배속',52,yy+18,{size:12,bold:1,align:'left'});[1,2,3].forEach((v,i)=>button('spd'+v,W-180+i*48,yy,42,38,'×'+v,{cb:()=>{S.set.spd=v;save()},col:S.set.spd===v?'#6a4a2a':'#3a2e2a',size:12,tc:S.set.spd===v?'#ffd257':'#f3e9dc'}));
 button('fs',46,y+254,W-92,40,'전체화면',{cb:toggleFS,col:'#3e4a56',size:12});button('rst',46,y+302,W-92,40,NOW-RST<2?'한 번 더 누르면 지워진다':'처음부터 다시',{cb:()=>{if(NOW-RST<2){S=newState();ensureDaily();save();RST=0;closeM();TRN=null;enterScene('forge')}else RST=NOW},col:NOW-RST<2?'#9a3a2a':'#3a2e2a',size:12});
 button('sX',W-72,y+8,34,30,'×',{cb:closeM,col:'#3a2e2a',size:14})}
function mSum(){const y=mY()+10,ph=300,d=M.d,l=Hn.loot;panel(30,y,W-60,ph);text(d.dead?'쓰러졌다':'사냥 끝',W/2,y+28,{size:18,bold:1,title:1,color:d.dead?'#ff8a7a':'#ffe2a0',pop:1,t0:M.t0,wave:1});
 const rows=[['처치',l.k+'마리'],['금화','+'+fmt(l.g)],[oreName(Hn.t),'+'+fmt(l.o)],[matName('g',Hn.t),'+'+l.gm],[matName('h',Hn.t),'+'+l.h],['경험치','+'+fmt(l.x)]];rows.forEach((r,i)=>{const yy=y+64+i*26;text(r[0],70,yy,{size:12,align:'left',color:'#b8a89a'});text(r[1],W-70,yy,{size:13,bold:1,align:'right',color:'#f3e9dc',pop:1,t0:M.t0+.15+i*.1})});
 if(d.dead)text('더 좋은 검을 만들어 다시 오자',W/2,y+226,{size:11,color:'#b8a89a'});button('sumGo',50,y+ph-58,W-100,46,'대장간으로',{cb:toForge,col:'#a8532a'})}
function mStage(){const y=H/2-240,ph=460,d=M.d,sd=stageDef(d.t);panel(20,y,W-40,ph,{c:'#241a26'});glowAt(W/2,y+140,'#ffd257',120,.2);
 text('새 지역',W/2,y+26,{size:12,bold:1,color:'#ffe2a0',wave:1});text(sd.n,W/2,y+58,{size:26,bold:1,title:1,color:'#fff4e6',pop:1,t0:M.t0+.2,gap:.08,wave:1.5,ol:1});
 const tier=Math.min(d.t,6),img=SPR.hero[tier][1];glowAt(W/2,y+150,SCARF[tier],60,.3);spr(img,W/2,y+190,4,{sy:1+Math.sin(NOW*2.4)*.02});text('새 갑옷',W/2,y+206,{size:11,bold:1,color:'#d8c8b4'});
 [['o',12],['g',3],['h',4]].forEach((r,i)=>{const yy=y+238+i*26,ic=r[0]==='o'?SPR.ico.ore:r[0]==='g'?SPR.ico.gem:SPR.ico.hide;spr(ic[Math.min(d.t,11)],W/2-80,yy,2,{ay:.5});text(matName(r[0],d.t)+' ×'+r[1],W/2-64,yy,{size:13,bold:1,align:'left',color:mix(matCol(r[0],d.t),'#ffffff',.35),pop:1,t0:M.t0+.5+i*.15})});
 text('금화 +'+fmt(d.g),W/2,y+322,{size:12,color:'#ffe2a0',pop:1,t0:M.t0+.9});text('적도 더 세다. 새 재료로 검을 새로 맞추자',W/2,y+348,{size:11,color:'#b8a89a'});
 button('stGo',44,y+ph-70,W-88,52,'대장간으로',{cb:toForge,col:'#a8532a',glow:1})}
function toggleFS(){const d=document,el=d.documentElement;try{if(!(d.fullscreenElement||d.webkitFullscreenElement)){const f=el.requestFullscreen||el.webkitRequestFullscreen;if(f)f.call(el)}else{const f=d.exitFullscreen||d.webkitExitFullscreen;if(f)f.call(d)}}catch(e){}}

// ===== Title =====
const TT={hit:0,sq:0};
function drawTitle(dt){cx.fillStyle='#160f0d';cx.fillRect(0,0,W,H);glowAt(W/2,H*.6,'#ff7a2a',260,.25+.05*Math.sin(NOW*3));
 const ay=H*.62;TT.hit-=dt;if(TT.hit<=0){TT.hit=rr(1.1,1.8);burst(PS,W/2+10,ay-52,22,{an:-PI/2,sp:PI*.6,v0:120,v1:360,g:600,dr:1.2,c:['#fff6c8','#ffd257','#ff9a3a'],k:'spk',s:2});ring(PS,W/2+10,ay-52,'#ffd257',40,3);TT.sq=1}TT.sq=Math.max(0,TT.sq-dt*5);
 spr(SPR.anvil,W/2,ay,5);const l=70,t=9*(1-TT.sq*.3),x=W/2+10,y=ay-57;cx.fillStyle='#140e10';cx.fillRect(x-l/2-1,y-t/2-1,l+2,t+2);cx.fillStyle='#d9531e';cx.fillRect(x-l/2,y-t/2,l,t);cx.fillStyle='#ff9a3a';cx.fillRect(x-l/2,y-t/2,l,t*.55);glowAt(x,y,'#ff8a2a',60,.35);
 if(R()<.4)part(PS,{x:W/2+rr(-120,120),y:H,vx:rr(-10,10),vy:rr(-80,-30),l:rr(2,4),s:2,c:pick(['#ffd257','#ff8a3a']),k:'sq',sh:1});drawParts(PS);
 const cols=['#ffe08a','#ffb060','#ff8a4a'];text('모루',W/2,H*.27,{size:64,bold:1,title:1,wave:4,spd:3,colors:i=>cols[i%3],ol:1,pop:1,t0:.3,gap:.18});
 text('두드려 만들고, 들고 나가 벤다',W/2,H*.27+56,{size:11,color:'#d8b898',pop:1,t0:1,gap:.03});
 if(NOW>1.6){cx.globalAlpha=.6+.4*Math.sin(NOW*3.5);text('눌러서 시작',W/2,H*.82,{size:14,bold:1,wave:1.5,color:'#fff4e6'});cx.globalAlpha=1}
 text('v0.1.0',W-10,H-12,{size:11,align:'right',color:'#5a4a44',shadow:null});hit('title',0,0,W,H,()=>{AU.init();go('forge')})}

// ===== Loop =====
let last=0;
function update(rdt){for(const k in UI.spr){const s=UI.spr[k];s.vv+=(s.tg-s.v)*420*rdt;s.vv*=Math.exp(-20*rdt);s.v+=s.vv*rdt}for(const k in UI.bump)UI.bump[k]=Math.max(0,UI.bump[k]-rdt*4);
 const ap=(a,b)=>Math.abs(b-a)<1?b:lerp(a,b,1-Math.exp(-rdt*9));UI.dg=ap(UI.dg,S.gold);const t=S.stage;UI.dm[0]=ap(UI.dm[0],matN('o',t));UI.dm[1]=ap(UI.dm[1],matN('g',t));UI.dm[2]=ap(UI.dm[2],matN('h',t));
 SHK*=Math.exp(-rdt*9);if(SHK<.1)SHK=0;FLASH=Math.max(0,FLASH-rdt*2.5);
 if(TRN){TRN.t+=rdt;if(TRN.t>=.32&&!TRN.sw){TRN.sw=1;enterScene(TRN.to)}if(TRN.t>=.64)TRN=null}
 if(M&&M.cl&&NOW-M.cl>.18){M=null}
 let dt=rdt;if(STOP>0){STOP-=rdt;dt=0}if(SLOWT>0){SLOWT-=rdt;dt*=SLOW}else SLOW=1;
 if(SCN==='forge'){updForge(dt);updParts(PW,dt)}
 else if(SCN==='hunt'&&!(M&&M.k==='set')){const hd=dt*(S.set.spd||1),n=Math.max(1,Math.ceil(hd/.025));for(let i=0;i<n;i++)updHunt(hd/n);ambient(rdt);updParts(PW,hd)}
 updParts(PS,dt);updParts(PU,rdt);
 saveT+=rdt;if(saveT>8){saveT=0;save()}}
function draw(rdt){UI.hits=[];cx.setTransform(DPR,0,0,DPR,0,0);cx.fillStyle='#0c0908';cx.fillRect(0,0,cv.width,cv.height);cx.setTransform(SC*DPR,0,0,SC*DPR,OX*DPR,0);cx.imageSmoothingEnabled=false;
 cx.save();cx.beginPath();cx.rect(0,0,W,H);cx.clip();
 if(SCN==='title')drawTitle(rdt);
 else{cx.fillStyle='#160f0d';cx.fillRect(0,0,W,H);if(SCN==='forge'){drawForge(rdt);panel(0,L.fb,W,H-L.fb+4,{c:'#1e1715'});drawForgeUI()}else drawHunt(rdt);drawHUD()}
 drawParts(PU);drawModal();drawToasts();
 if(FLASH>0){cx.globalAlpha=Math.min(1,FLASH)*.8;cx.fillStyle=FLC;cx.fillRect(0,0,W,H);cx.globalAlpha=1}
 if(TRN){const t=TRN.t,R0=Math.hypot(W,H)*.6,r=t<.32?lerp(R0,0,eIn(t/.32)):lerp(0,R0,eOut((t-.32)/.32));cx.fillStyle='#0c0908';cx.beginPath();cx.rect(0,0,W,H);cx.arc(W/2,H*.55,Math.max(0,r),0,TAU,true);cx.fill()}
 cx.restore()}
function frame(ts){const t=ts/1000;const rdt=Math.min(.05,Math.max(0,t-last||0));last=t;NOW+=rdt;try{update(rdt);draw(rdt)}catch(e){console.error(e)}requestAnimationFrame(frame)}
function tp(e){return{x:(e.clientX-OX)/SC,y:e.clientY/SC}}
cv.addEventListener('pointerdown',e=>{e.preventDefault();AU.init();const p=tp(e);PT.down=true;PT.x=p.x;PT.y=p.y;const h=hitAt(p.x,p.y);PT.id=h?h.id:null;if(h&&h.dn)h.dn()});
window.addEventListener('pointerup',e=>{const p=tp(e);const h=hitAt(p.x,p.y);if(MG.k==='hold'&&MG.on)holdRelease();if(h&&PT.down&&h.id===PT.id&&h.cb){AU.tap();h.cb()}PT.down=false;PT.id=null});
window.addEventListener('pointercancel',()=>{if(MG.k==='hold'&&MG.on)holdRelease();PT.down=false;PT.id=null});
window.addEventListener('keydown',e=>{AU.init();if(e.repeat)return;const k=e.key;
 if(k===' '||k==='Enter'){e.preventDefault();if(SCN==='title')go('forge');else if(SCN==='forge'&&!M)forgeTap()}
 if(SCN==='hunt'&&!M){if(k==='1'||k==='q'||k==='Q')skill(0);if(k==='2'||k==='w'||k==='W')skill(1)}
 if(k==='Escape'&&M&&(M.k==='quest'||M.k==='set'))closeM()});
window.addEventListener('keyup',e=>{if((e.key===' '||e.key==='Enter')&&MG.k==='hold'&&MG.on)holdRelease()});
window.addEventListener('resize',resize);
document.addEventListener('visibilitychange',()=>{if(document.hidden)save()});
if(document.fonts&&document.fonts.addEventListener)document.fonts.addEventListener('loadingdone',()=>{CW={}});
if(document.fonts&&document.fonts.load){document.fonts.load('11px Galmuri11').then(()=>{CW={}}).catch(()=>{});document.fonts.load('bold 11px Galmuri11').catch(()=>{});document.fonts.load('14px Galmuri14').catch(()=>{})}
S=load();ensureDaily();buildSprites();resize();fReset('idle');F.fx=F.tfx;F.fy=F.tfy;UI.dg=S.gold;
window.__M={get S(){return S},set S(v){S=v},newState,F,MG,Hn,K,ST,startCraft,pickShape,forgeTap,holdRelease,enhOpen,enhGo,huntStart,updHunt,skill,tapVein,bossStart,breakIce,heroAtk,heroMaxHp,power,stats,needN,craftTier,payCraft,enterScene,update,draw,hitAt,UI,get M(){return M},get SCN(){return SCN},claimQ,ensureDaily,gapOf,front,hurtHero,rollGrade,gradeOdds,effQ,perfN,mkBlade,mkGuard,mkHandle,doEnh,canEnh,payEnh,enhCost,meltV,stageClear,fReset};
requestAnimationFrame(frame);
</script>
</body>
</html>
