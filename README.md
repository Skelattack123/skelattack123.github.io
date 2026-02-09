# skelattack123.github.io
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <title>NEON BREAKER ULTIMATE</title>
    <style>
        :root { --neon: #00ffff; }
        body { margin: 0; background: #000; overflow: hidden; font-family: 'Avenir Next', sans-serif; color: var(--neon); }
        canvas { display: block; touch-action: none; }
        
        #ui-layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 10; }
        .hud { position: absolute; top: 20px; width: 100%; text-align: center; font-weight: 800; text-shadow: 0 0 10px var(--neon); }
        
        .menu-overlay { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); 
                        background: rgba(0,0,0,0.9); border: 2px solid var(--neon); padding: 25px; 
                        text-align: center; pointer-events: auto; display: none; min-width: 250px; }
        
        .btn { background: rgba(0,255,255,0.1); border: 1px solid var(--neon); padding: 12px 24px; 
               color: var(--neon); cursor: pointer; margin: 10px; font-weight: bold; pointer-events: auto; }
        
        #settings-btn { position: absolute; bottom: 20px; right: 20px; font-size: 24px; }
        #share-btn { display: none; }
    </style>
</head>
<body>

    <div id="ui-layer">
        <div class="hud" id="stats">SCORE: 0 | SHARDS: 0 | BEST: 0</div>
        
        <div id="start-menu" class="menu-overlay" style="display: block;">
            <h1 style="margin-top:0">NEON BREAKER</h1>
            <p id="instr">EDITOR: TAP SCREEN TO PLACE BRICKS</p>
            <button id="start-btn" class="btn">START MISSION</button>
            <button id="share-btn" class="btn">📤 SHARE SCORE</button>
        </div>

        <div id="settings-menu" class="menu-overlay">
            <h2>ENGINE SETTINGS</h2>
            <label>SCREEN SHAKE: <input type="checkbox" id="shake-toggle" checked></label><br><br>
            <label>THEME COLOR: <input type="color" id="color-picker" value="#00ffff"></label><br><br>
            <button id="close-settings" class="btn">CLOSE</button>
        </div>

        <button id="settings-btn" class="btn">⚙️</button>
    </div>

    <canvas id="gameCanvas"></canvas>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    // --- State & Settings ---
    let gameState = 'EDITOR';
    let score = 0, shards = parseInt(localStorage.getItem('shards')) || 0;
    let highScore = parseInt(localStorage.getItem('highScore')) || 0;
    let totalBricks = 0, currentProgress = 0, shakeAmount = 0;
    
    let settings = { shakeEnabled: true, themeColor: '#00ffff' };
    
    let paddle = { x: 0, y: 0, w: 100, h: 15, isEnlarged: false };
    let balls = [], bricks = [], particles = [], activePowerUps = [], ghosts = [];

    // --- Initialization ---
    function init() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        paddle.y = canvas.height - 100;
        paddle.x = canvas.width / 2 - paddle.w / 2;
        updateUI();
    }

    // --- Systems ---
    function spawnBall(isExtra = false) {
        const b = {
            x: isExtra ? paddle.x + paddle.w/2 : canvas.width / 2,
            y: paddle.y - 20,
            dx: (Math.random() - 0.5) * 8,
            dy: -6,
            radius: 8
        };
        balls.push(b);
    }

    function createExplosion(x, y) {
        for(let i=0; i<12; i++) {
            particles.push({
                x, y, vx: (Math.random()-0.5)*10, vy: (Math.random()-0.5)*10,
                size: Math.random()*4+1, alpha: 1
            });
        }
    }

    function spawnPowerUp(x, y) {
        const rnd = Math.random();
        if (rnd > 0.85) {
            activePowerUps.push({
                x, y, w: 20, h: 20, dy: 3, 
                type: rnd > 0.93 ? 'GHOST' : 'LONG',
                color: rnd > 0.93 ? '#fff' : '#bf00ff'
            });
        }
    }

    // --- Main Loop ---
    function update() {
        if (gameState !== 'PLAYING') return;

        balls.forEach((b, i) => {
            b.x += b.dx; b.y += b.dy;
            if (b.x + b.radius > canvas.width || b.x - b.radius < 0) b.dx *= -1;
            if (b.y - b.radius < 0) b.dy *= -1;

            // Paddle Hit
            if (b.y + b.radius > paddle.y && b.x > paddle.x && b.x < paddle.x + paddle.w) {
                b.dy = -Math.abs(b.dy);
                b.dx = (b.x - (paddle.x + paddle.w/2)) * 0.2;
                if (settings.shakeEnabled) shakeAmount = 5;
            }

            // Brick Hit
            bricks = bricks.filter(br => {
                const hit = b.x > br.x && b.x < br.x + br.w && b.y > br.y && b.y < br.y + br.h;
                if (hit) {
                    b.dy *= -1; score += 10;
                    createExplosion(br.x + br.w/2, br.y + br.h/2);
                    spawnPowerUp(br.x + br.w/2, br.y + br.h/2);
                    if (settings.shakeEnabled) shakeAmount = 12;
                }
                return !hit;
            });

            // Ghost Trail
            ghosts.push({x: b.x, y: b.y, a: 0.4});
            if (b.y > canvas.height) balls.splice(i, 1);
        });

        // PowerUps
        activePowerUps.forEach((p, i) => {
            p.y += p.dy;
            if (p.y + p.h > paddle.y && p.x > paddle.x && p.x < paddle.x + paddle.w) {
                p.type === 'LONG' ? activateLong() : spawnBall(true);
                activePowerUps.splice(i, 1);
            }
        });

        if (balls.length === 0) endGame();
    }

    function draw() {
        ctx.save();
        if (shakeAmount > 0) {
            ctx.translate((Math.random()-0.5)*shakeAmount, (Math.random()-0.5)*shakeAmount);
            shakeAmount *= 0.9;
        }

        ctx.fillStyle = '#000';
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Progress Bar
        if (totalBricks > 0) {
            currentProgress += ((1 - bricks.length/totalBricks) - currentProgress) * 0.1;
            ctx.fillStyle = settings.themeColor + '33';
            ctx.fillRect(0, 0, canvas.width, 4);
            ctx.fillStyle = settings.themeColor;
            ctx.fillRect(0, 0, canvas.width * currentProgress, 4);
        }

        // Render Ghosts
        ghosts = ghosts.filter(g => {
            ctx.globalAlpha = g.a; ctx.fillStyle = '#fff';
            ctx.beginPath(); ctx.arc(g.x, g.y, 8, 0, Math.PI*2); ctx.fill();
            g.a -= 0.04; return g.a > 0;
        });
        ctx.globalAlpha = 1;

        // Render Bricks & Paddle
        ctx.shadowBlur = 10; ctx.shadowColor = settings.themeColor;
        ctx.fillStyle = settings.themeColor;
        bricks.forEach(b => ctx.fillRect(b.x, b.y, b.w, b.h));
        ctx.fillRect(paddle.x, paddle.y, paddle.w, paddle.h);

        // Render Balls
        ctx.fillStyle = '#fff'; ctx.shadowColor = '#fff';
        balls.forEach(b => { ctx.beginPath(); ctx.arc(b.x, b.y, b.radius, 0, Math.PI*2); ctx.fill(); });

        // Render Particles & PowerUps
        particles = particles.filter(p => {
            ctx.globalAlpha = p.alpha; ctx.fillStyle = settings.themeColor;
            ctx.fillRect(p.x, p.y, p.size, p.size);
            p.x += p.vx; p.y += p.vy; p.vy += 0.2; p.alpha -= 0.02;
            return p.alpha > 0;
        });
        activePowerUps.forEach(p => {
            ctx.fillStyle = p.color; ctx.shadowColor = p.color;
            ctx.fillRect(p.x, p.y, p.w, p.h);
        });

        ctx.restore();
        update();
        requestAnimationFrame(draw);
    }

    // --- Interactions ---
    function activateLong() {
        paddle.w = 180;
        setTimeout(() => paddle.w = 100, 8000);
    }

    function updateUI() {
        document.getElementById('stats').innerText = `SCORE: ${score} | SHARDS: ${shards} | BEST: ${highScore}`;
    }

    function endGame() {
        gameState = 'EDITOR';
        document.getElementById('start-menu').style.display = 'block';
        document.getElementById('share-btn').style.display = 'inline-block';
        if (score > highScore) { highScore = score; localStorage.setItem('highScore', score); }
        localStorage.setItem('shards', shards);
        updateUI();
    }

    // --- Listeners ---
    document.getElementById('start-btn').onclick = () => {
        gameState = 'PLAYING';
        document.getElementById('start-menu').style.display = 'none';
        totalBricks = bricks.length;
        score = 0; balls = []; spawnBall();
    };

    document.getElementById('share-btn').onclick = async () => {
        const data = { title: 'Neon Breaker', text: `Scored ${score} in Neon Breaker!`, url: window.location.href };
        if (navigator.share) await navigator.share(data);
    };

    document.getElementById('settings-btn').onclick = () => document.getElementById('settings-menu').style.display = 'block';
    document.getElementById('close-settings').onclick = () => document.getElementById('settings-menu').style.display = 'none';
    
    document.getElementById('color-picker').oninput = (e) => {
        settings.themeColor = e.target.value;
        document.documentElement.style.setProperty('--neon', settings.themeColor);
    };
    
    document.getElementById('shake-toggle').onchange = (e) => settings.shakeEnabled = e.target.checked;

    canvas.addEventListener('touchstart', e => {
        const t = e.touches[0];
        if (gameState === 'EDITOR') {
            const gw = 70, gh = 25;
            const sx = Math.floor(t.pageX / gw) * gw + 5;
            const sy = Math.floor(t.pageY / gh) * gh + 5;
            if (sy < paddle.y - 50) bricks.push({x: sx, y: sy, w: 60, h: 15});
        }
    });

    canvas.addEventListener('touchmove', e => {
        if (gameState === 'PLAYING') paddle.x = e.touches[0].pageX - paddle.w/2;
    });

    window.onresize = init;
    init(); draw();
</script>
</body>
</html>
