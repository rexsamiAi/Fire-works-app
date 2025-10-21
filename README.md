<!-- Simple Fireworks App

Save this file as `simple-fireworks-app.html` and open in your browser.
Click or tap anywhere to launch a firework. Resize-safe and mobile-friendly.

Features:
- Click/tap to launch rockets that explode into particles
- Auto-launching fireworks every few seconds
- High-DPI canvas scaling
- Adjustable settings at top-right
--> 

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Simple Fireworks App</title>
  <style>
    html,body{height:100%;margin:0;background:#000}
    canvas{display:block}
    .ui {
      position:fixed;top:12px;right:12px;color:#fff;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial,sans-serif;font-size:13px;background:rgba(0,0,0,0.35);backdrop-filter:blur(6px);padding:8px 10px;border-radius:8px;box-shadow:0 6px 18px rgba(0,0,0,0.6);
    }
    .ui label{display:flex;align-items:center;gap:8px}
    .ui input[type=range]{width:120px}
    .hint{position:fixed;left:12px;bottom:12px;color:#ddd;font-family:system-ui,Segoe UI,Roboto;opacity:0.9}
    a.link{color:#66f;text-decoration:none}
  </style>
</head>
<body>
  <canvas id="c"></canvas>
  <div class="ui">
    <div style="font-weight:600;margin-bottom:6px">Fireworks</div>
    <label>Auto: <input id="auto" type="checkbox" checked></label>
    <label>Frequency <input id="freq" type="range" min="200" max="4000" step="100" value="900"></label>
    <label>Particles <input id="count" type="range" min="12" max="150" step="2" value="60"></label>
    <div style="margin-top:6px;font-size:11px;opacity:0.85">Click or tap anywhere to launch</div>
  </div>
  <div class="hint">Press <b>Space</b> to toggle auto. Made with &hearts;</div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  let DPR = Math.max(1, window.devicePixelRatio || 1);
  let w, h;

  function resize(){
    DPR = Math.max(1, window.devicePixelRatio || 1);
    w = Math.floor(window.innerWidth);
    h = Math.floor(window.innerHeight);
    canvas.width = Math.floor(w * DPR);
    canvas.height = Math.floor(h * DPR);
    canvas.style.width = w + 'px';
    canvas.style.height = h + 'px';
    ctx.setTransform(DPR,0,0,DPR,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  // Utilities
  const rand = (a,b)=> Math.random()*(b-a)+a;
  const TAU = Math.PI*2;

  // Entities
  const rockets = [];
  const shells = []; // after explosion particles

  class Rocket {
    constructor(x,y,tx,ty){
      this.x = x; this.y = y; this.tx = tx; this.ty = ty;
      this.vx = (tx - x) * 0.02 + rand(-0.6,0.6);
      this.vy = (ty - y) * 0.02 + rand(-6,-2);
      this.life = 0; this.maxLife = rand(45,75);
      this.hue = Math.floor(rand(0,360));
      this.tail = [];
    }
    update(){
      this.vy += 0.06; // gravity (negative upward initial)
      this.x += this.vx;
      this.y += this.vy;
      this.tail.push({x:this.x,y:this.y});
      if(this.tail.length>10) this.tail.shift();
      this.life++;
      // explode when life or near target
      if(this.life > this.maxLife || this.vy >= 0) {
        explode(this.x,this.y,this.hue);
        return false;
      }
      return true;
    }
    draw(ctx){
      ctx.beginPath();
      ctx.moveTo(this.x,this.y);
      if(this.tail.length>1){
        for(let i=0;i<this.tail.length-1;i++){
          const a=this.tail[i], b=this.tail[i+1];
          ctx.lineTo(b.x,b.y);
        }
        ctx.strokeStyle = 'hsl('+this.hue+',100%,'+ (Math.max(40,Math.min(70,60 - this.life*0.3))) + '%)';
        ctx.lineWidth = 2;
        ctx.stroke();
      }
    }
  }

  class Particle {
    constructor(x,y,angle,speed,hue,count){
      this.x = x; this.y = y;
      this.vx = Math.cos(angle) * speed;
      this.vy = Math.sin(angle) * speed;
      this.life = 0;
      this.ttl = Math.floor(rand(40,110));
      this.hue = (hue + rand(-20,20))%360;
      this.size = rand(1.2,3.2);
      this.alpha = 1;
      this.count = count || 60;
      this.decay = rand(0.008, 0.02);
    }
    update(){
      this.vy += 0.02; // gravity
      this.vx *= 0.995;
      this.vy *= 0.995;
      this.x += this.vx;
      this.y += this.vy;
      this.life++;
      this.alpha -= this.decay;
      if(this.alpha <= 0 || this.life > this.ttl) return false;
      return true;
    }
    draw(ctx){
      ctx.beginPath();
      ctx.globalAlpha = Math.max(0, Math.min(1, this.alpha));
      ctx.fillStyle = 'hsl('+this.hue+',100%,'+ (50 + Math.sin(this.life/6)*10) + '%)';
      ctx.arc(this.x, this.y, this.size, 0, TAU);
      ctx.fill();
      ctx.globalAlpha = 1;
    }
  }

  function explode(x,y,hue){
    const count = +document.getElementById('count').value || 60;
    const baseSpeed = rand(1.8,4.6);
    for(let i=0;i<count;i++){
      const angle = rand(0, TAU);
      const speed = baseSpeed * (0.4 + Math.random()*1.4) * (0.6 + Math.random()*0.8);
      shells.push(new Particle(x,y,angle,speed,hue,count));
    }
    // add a few bigger sparks
    for(let i=0;i<Math.floor(count*0.05);i++){
      const angle = rand(0,TAU);
      const p = new Particle(x,y,angle,rand(2.8,6),hue,count);
      p.size = rand(2.6,4.8); p.alpha = 1; p.decay = rand(0.01,0.03);
      shells.push(p);
    }
  }

  // Animation loop
  let last = performance.now();
  function loop(t){
    const dt = t - last; last = t;
    // fade background slightly for trails
    ctx.globalCompositeOperation = 'source-over';
    ctx.fillStyle = 'rgba(0,0,0,0.18)';
    ctx.fillRect(0,0,w,h);

    // draw fireworks blending
    ctx.globalCompositeOperation = 'lighter';

    // update rockets
    for(let i=rockets.length-1;i>=0;i--){
      const r = rockets[i];
      if(!r.update()) rockets.splice(i,1);
      else r.draw(ctx);
    }

    // update particles
    for(let i=shells.length-1;i>=0;i--){
      const p = shells[i];
      if(!p.update()) shells.splice(i,1);
      else p.draw(ctx);
    }

    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  // Interaction
  function launchAt(x,y){
    // launch rocket from bottom center or random bottom
    const fromX = rand(w*0.2,w*0.8);
    const fromY = h + 10;
    rockets.push(new Rocket(fromX, fromY, x, y));
  }

  // mouse / touch
  canvas.addEventListener('pointerdown', (e)=>{
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left);
    const y = (e.clientY - rect.top);
    launchAt(x,y);
  });

  // Auto-launch
  let autoTimer = null;
  const autoCheckbox = document.getElementById('auto');
  const freqInput = document.getElementById('freq');

  function startAuto(){
    stopAuto();
    const ms = +freqInput.value || 900;
    autoTimer = setInterval(()=>{
      const tx = rand(w*0.15, w*0.85);
      const ty = rand(h*0.12, h*0.5);
      launchAt(tx,ty);
    }, ms);
  }
  function stopAuto(){ if(autoTimer){ clearInterval(autoTimer); autoTimer=null; } }

  autoCheckbox.addEventListener('change', ()=>{
    if(autoCheckbox.checked) startAuto(); else stopAuto();
  });
  freqInput.addEventListener('input', ()=>{
    if(autoCheckbox.checked) startAuto();
  });

  // keyboard toggle
  window.addEventListener('keydown',(e)=>{
    if(e.code === 'Space'){
      e.preventDefault();
      autoCheckbox.checked = !autoCheckbox.checked;
      if(autoCheckbox.checked) startAuto(); else stopAuto();
    }
  });

  // start
  startAuto();

  // initial background
  ctx.fillStyle = '#000'; ctx.fillRect(0,0,w,h);

})();
</script>
</body>
</html>
