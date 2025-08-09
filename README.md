index.html
# Grasshopper-game-v-2
<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const wrap = document.getElementById('gameWrap');
  const scoreLabel = document.getElementById('scoreLabel');
  const bestLabel = document.getElementById('bestLabel');
  const startScreen = document.getElementById('startScreen');
  const gameOverScreen = document.getElementById('gameOver');
  const finalScore = document.getElementById('finalScore');
  const startBtn = document.getElementById('startBtn');
  const restartBtn = document.getElementById('restartBtn');
  const howBtn = document.getElementById('howBtn');
  const fullscreenBtn = document.getElementById('fullscreenBtn');
  const muteBtn = document.getElementById('muteBtn');

  // Create settings button
  const settingsBtn = document.createElement('button');
  settingsBtn.textContent = 'Settings';
  settingsBtn.style.position = 'absolute';
  settingsBtn.style.top = '12px';
  settingsBtn.style.left = '12px';
  settingsBtn.style.zIndex = '20';
  settingsBtn.style.background = '#fff';
  settingsBtn.style.borderRadius = '999px';
  settingsBtn.style.padding = '10px 14px';
  settingsBtn.style.border = '0';
  settingsBtn.style.boxShadow = '0 6px 14px rgba(0,0,0,.12)';
  settingsBtn.style.fontWeight = '700';
  wrap.appendChild(settingsBtn);

  let dpr = Math.max(1, window.devicePixelRatio || 1);
  function resize() {
    canvas.width = wrap.clientWidth * dpr;
    canvas.height = wrap.clientHeight * dpr;
    canvas.style.width = wrap.clientWidth + 'px';
    canvas.style.height = wrap.clientHeight + 'px';
    ctx.setTransform(dpr,0,0,dpr,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  // Game vars
  let gravity = 0.6, baseJump = 18;
  let player, obstacles, spawnTimer, score, gameSpeed, running, highScore;
  let muted = false;
  let dayMode = true; // true=day, false=night

  // Background layers
  let buildings = [];
  let clouds = [];

  // Stars for night mode
  let stars = [];
  const maxStars = 100;

  // Load high score
  highScore = parseInt(localStorage.getItem('grasshopper_best') || '0', 10);
  bestLabel.innerText = 'Best: ' + highScore;

  // Audio setup (music + fx)
  let audioCtx, musicGain, musicOsc, jumpSound, hitSound;
  function initAudio(){
    try {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();

      // Background music: simple gentle oscillator loop
      musicGain = audioCtx.createGain();
      musicGain.gain.value = muted ? 0 : 0.05;
      musicGain.connect(audioCtx.destination);

      musicOsc = audioCtx.createOscillator();
      musicOsc.type = 'triangle';
      musicOsc.frequency.value = 220;
      musicOsc.connect(musicGain);
      musicOsc.start();

      // Jump sound
      jumpSound = (t=0.06, f=600) => {
        if(muted) return;
        const o = audioCtx.createOscillator();
        const g = audioCtx.createGain();
        o.type='sine'; o.frequency.value=f;
        g.gain.value = 0.08;
        o.connect(g); g.connect(audioCtx.destination);
        o.start();
        g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + t);
        setTimeout(()=>o.stop(), t*1000 + 20);
      };

      // Hit sound
      hitSound = () => {
        if(muted) return;
        const o = audioCtx.createOscillator();
        const g = audioCtx.createGain();
        o.type='triangle';
        o.frequency.value=120;
        g.gain.value=0.12;
        o.connect(g);
        g.connect(audioCtx.destination);
        o.start();
        g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.18);
        setTimeout(()=>o.stop(),200);
      };

    } catch(e){
      // No audio support
      jumpSound = () => {};
      hitSound = () => {};
    }
  }
  initAudio();

  // Helper to create stars
  function createStars(){
    stars = [];
    for(let i=0; i<maxStars; i++){
      stars.push({
        x: Math.random()*wrap.clientWidth,
        y: Math.random()*wrap.clientHeight*0.6,
        r: Math.random()*1.2 + 0.3,
        alpha: Math.random()*0.8 + 0.2
      });
    }
  }
  createStars();

  // Classes

  class Grasshopper {
    constructor(){
      this.w = 48; this.h = 36;
      this.x = 64;
      this.y = (wrap.clientHeight - this.h - 56);
      this.vy = 0;
      this.grounded = true;
      this.jumpForce = baseJump;
      this.legWiggle = 0; // animation frame
    }
    jump(){
      if(this.grounded){
        this.vy = -this.jumpForce;
        this.grounded = false;
        this.legWiggle = 10; // wiggle legs for 10 frames
        jumpSound();
      }
    }
    update(){
      this.vy += gravity;
      this.y += this.vy;
      const ground = wrap.clientHeight - this.h - 56;
      if(this.y >= ground){
        this.y = ground;
        this.vy = 0;
        if(!this.grounded){
          this.legWiggle = 10; // wiggle on landing too
        }
        this.grounded = true;
      }
      if(this.legWiggle > 0) this.legWiggle--;
    }
    draw(ctx){
      // Antennae
      ctx.strokeStyle = '#2e7d32';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(this.x + this.w*0.2, this.y + this.h*0.15);
      ctx.lineTo(this.x + this.w*0.1, this.y + this.h*0.05);
      ctx.moveTo(this.x + this.w*0.3, this.y + this.h*0.15);
      ctx.lineTo(this.x + this.w*0.4, this.y + this.h*0.05);
      ctx.stroke();

      // Segmented body: multiple ellipses
      ctx.fillStyle = '#388e3c';
      let segmentCount = 3;
      for(let i = 0; i < segmentCount; i++){
        let cx = this.x + this.w*0.3 + i * this.w*0.35;
        let cy = this.y + this.h*0.55;
        let rw = this.w*0.35;
        let rh = this.h*0.45;
        ctx.beginPath();
        ctx.ellipse(cx, cy, rw, rh, 0, 0, Math.PI*2);
        ctx.fill();
      }

      // Head / eye on front segment
      ctx.fillStyle = '#2e7d32';
      ctx.beginPath();
      ctx.arc(this.x + this.w*1.1, this.y + this.h*0.22, 6, 0, Math.PI*2);
      ctx.fill();

      ctx.fillStyle='#000';
      ctx.beginPath();
      ctx.arc(this.x + this.w*1.15, this.y + this.h*0.22, 2.4, 0, Math.PI*2);
      ctx.fill();

      // Legs (animated wiggle)
      ctx.strokeStyle = '#2e7d32';
      ctx.lineWidth = 2;
      ctx.beginPath();

      let wiggleOffset = (this.legWiggle > 0) ? Math.sin(this.legWiggle * 0.5) * 6 : 0;

      ctx.moveTo(this.x + 10, this.y + this.h - 2);
      ctx.bezierCurveTo(this.x - 2 + wiggleOffset, this.y + this.h + 8, this.x + 8 + wiggleOffset, this.y + this.h + 10, this.x - 6 + wiggleOffset, this.y + this.h + 12);
      ctx.moveTo(this.x + 22, this.y + this.h - 2);
      ctx.bezierCurveTo(this.x + 14 - wiggleOffset, this.y + this.h + 10, this.x + 24 - wiggleOffset, this.y + this.h + 12, this.x + 8 - wiggleOffset, this.y + this.h + 12);
      ctx.stroke();
    }
  }

  class Obstacle {
    constructor(speed){
      this.w = 16 + Math.random()*28;
      this.h = 22 + Math.random()*64;
      this.x = wrap.clientWidth + (Math.random()*120);
      this.y = wrap.clientHeight - this.h - 56;
      this.speed = speed;
      this.passed = false;
    }
    update(){ this.x -= this.speed; }
    draw(ctx){
      ctx.fillStyle = '#6d4c41';
      ctx.fillRect(this.x, this.y, this.w, this.h);
      // bark details
      ctx.fillStyle = '#5d3f36';
      ctx.fillRect(this.x+4, this.y+6, Math.min(10,this.w-8), 3);
    }
  }

  class Building {
    constructor(){
      this.w = 60 + Math.random() * 40;
      this.h = 100 + Math.random() * 100;
      this.x = wrap.clientWidth + Math.random() * 200;
      this.y = wrap.clientHeight - this.h - 56;
      this.speed = 0;
    }
    update(speed){
      this.speed = speed * 0.3;
      this.x -= this.speed;
    }
    draw(ctx){
      ctx.fillStyle = dayMode ? '#7b8a8b' : '#35454e';
      ctx.fillRect(this.x, this.y, this.w, this.h);

      // windows: lit in night mode, off in day mode
      let rows = Math.floor(this.h / 20);
      let cols = Math.floor(this.w / 15);
      for(let r=0; r<rows; r++){
        for(let c=0; c<cols; c++){
          if(Math.random() < 0.5){
            ctx.fillStyle = dayMode ? '#a2b1b2' : '#ffe066';
            ctx.fillRect(this.x + 3 + c*15, this.y + 4 + r*20, 10, 12);
          }
        }
      }
    }
  }

  class Cloud {
    constructor(){
      this.w = 80 + Math.random()*40;
      this.h = 40 + Math.random()*20;
      this.x = wrap.clientWidth + Math.random() * 300;
      this.y = 30 + Math.random() * 80;
      this.speed = 0;
    }
    update(speed){
      this.speed = speed * 0.1;
      this.x -= this.speed;
    }
    draw(ctx){
      ctx.fillStyle = dayMode ? 'rgba(255,255,255,0.8)' : 'rgba(200,200,200,0.4)';
      ctx.beginPath();
      ctx.ellipse(this.x, this.y, this.w*0.7, this.h*0.5, 0, 0, Math.PI*2);
      ctx.ellipse(this.x + this.w*0.4, this.y + 5, this.w*0.6, this.h*0.5, 0, 0, Math.PI*2);
      ctx.ellipse(this.x + this.w*0.8, this.y + 2, this.w*0.5, this.h*0.4, 0, 0, Math.PI*2);
      ctx.fill();
    }
  }

  // Initialize game state
  function reset(){
    player = new Grasshopper();
    obstacles = [];
    spawnTimer = 0;
    score = 0;
    gameSpeed = 3; // slower speed start
    running = false;
    scoreLabel.innerText = 'Score: 0';
    gameOverScreen.style.display = 'none';

    buildings = [];
    clouds = [];
    // spawn initial buildings & clouds for background
    for(let i=0; i<6; i++){
      buildings.push(new Building());
    }
    for(let i=0; i<8; i++){
      clouds.push(new Cloud());
    }
  }
  reset();

  // Spawn obstacles
  function spawn(){
    obstacles.push(new Obstacle(gameSpeed));
  }

  // User jump handler
  function userJump(){
    if(!running) return;
    player.jump();
  }

  // Draw sun or moon
  function drawSunOrMoon(){
    if(dayMode){
      // Sun
      const sunX = wrap.clientWidth - 80;
      const sunY = 80;
      const radius = 40;
      const gradient = ctx.createRadialGradient(sunX, sunY, 10, sunX, sunY, radius);
      gradient.addColorStop(0, '#fff59d');
      gradient.addColorStop(1, '#fbc02d');
      ctx.fillStyle = gradient;
      ctx.beginPath();
      ctx.arc(sunX, sunY, radius, 0, Math.PI * 2);
      ctx.fill();
    } else {
      // Moon
      const moonX = wrap.clientWidth - 80;
      const moonY = 80;
      ctx.fillStyle = '#f5f3ce';
      ctx.beginPath();
      ctx.arc(moonX, moonY, 30, 0, Math.PI * 2);
      ctx.fill();
      // crescent shape
      ctx.fillStyle = dayMode ? 'transparent' : '#2b2d42';
      ctx.beginPath();
      ctx.arc(moonX + 12, moonY - 6, 30, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  // Draw stars
  function drawStars(){
    if(!dayMode){
      for(let s of stars){
        ctx.fillStyle = `rgba(255, 255, 255, ${s.alpha})`;
        ctx.beginPath();
        ctx.arc(s.x, s.y, s.r, 0, Math.PI*2);
        ctx.fill();
      }
    }
  }

  // Game loop
  let last = performance.now();
  function loop(t){
    const dt = (t - last) / 1000;
    last = t;

    // Clear
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // Background
    if(dayMode){
      ctx.fillStyle = '#a8e6cf'; // sky day
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#dcedc1'; // gradient bottom day
      ctx.fillRect(0, wrap.clientHeight * 0.6 * dpr, canvas.width, canvas.height);
    } else {
      ctx.fillStyle = '#1a1a2e'; // dark night sky
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#16213e'; // darker bottom night
      ctx.fillRect(0, wrap.clientHeight * 0.6 * dpr, canvas.width, canvas.height);
    }

    // Stars & sun/moon
    drawStars();
    drawSunOrMoon();

    // Ground
    ctx.fillStyle = '#98d8a3';
    ctx.fillRect(0, wrap.clientHeight - 56, wrap.clientWidth, 56);

    // Update and draw background clouds/buildings
    for(let i = buildings.length - 1; i >= 0; i--){
      let b = buildings[i];
      b.update(gameSpeed);
      b.draw(ctx);
      if(b.x + b.w < 0){
        buildings.splice(i,1);
        buildings.push(new Building());
      }
    }
    for(let i = clouds.length - 1; i >= 0; i--){
      let c = clouds[i];
      c.update(gameSpeed);
      c.draw(ctx);
      if(c.x + c.w < 0){
        clouds.splice(i,1);
        clouds.push(new Cloud());
      }
    }

    // Update player
    player.update();

    // Spawn obstacles based on timer
    spawnTimer += dt * 60;
    const spawnGap = Math.max(48 - Math.floor(gameSpeed), 22);
    if(spawnTimer > spawnGap){
      spawn();
      spawnTimer = 0;
    }

    // Update obstacles, draw, collision detection
    for(let i = obstacles.length - 1; i >= 0; i--){
      const o = obstacles[i];
      o.speed = gameSpeed;
      o.update();
      o.draw(ctx);

      // Passed obstacles increase score
      if(!o.passed && o.x + o.w < player.x){
        o.passed = true;
        score++;
        scoreLabel.innerText = 'Score: ' + score;

        // Speed up every 6 points
        if(score % 6 === 0){
          gameSpeed += 0.5;
        }
      }

      // Remove offscreen
      if(o.x + o.w < -20){
        obstacles.splice(i,1);
      }

      // Collision detection (AABB)
      if(player.x < o.x + o.w &&
         player.x + player.w > o.x &&
         player.y < o.y + o.h &&
         player.y + player.h > o.y){
        running = false;
        hitSound();
        endGame();
      }
    }

    // Draw player last (front)
    player.draw(ctx);

    if(running) requestAnimationFrame(loop);
  }

  // Start game function
  function startGame(){
    startScreen.style.display = 'none';
    reset();
    running = true;
