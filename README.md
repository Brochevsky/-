<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>🏒 АЭРОХОККЕЙ | Играй кистями рук</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; user-select: none; }
        body {
            background: radial-gradient(circle at 30% 20%, #0a0a2a, #000);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: monospace;
            padding: 20px;
        }
        .game-container { text-align: center; }
        h1 {
            font-size: 1.8em;
            background: linear-gradient(135deg, #88aaff, #ff44aa);
            -webkit-background-clip: text;
            background-clip: text;
            color: transparent;
            margin-bottom: 5px;
        }
        .canvas-wrapper {
            position: relative;
            display: inline-block;
            border-radius: 24px;
            overflow: hidden;
            border: 3px solid #44aaff;
            box-shadow: 0 0 30px #44aaffaa;
        }
        canvas { display: block; cursor: none; }
        .score-board {
            margin-top: 15px;
            display: flex;
            gap: 50px;
            justify-content: center;
            background: #050520cc;
            backdrop-filter: blur(8px);
            padding: 10px 30px;
            border-radius: 60px;
            color: white;
            font-weight: bold;
            font-size: 1.4em;
        }
        .score-board span { color: #ffff88; font-size: 1.6em; }
        button {
            background: #2244aa;
            border: none;
            color: white;
            padding: 8px 25px;
            margin: 10px;
            border-radius: 40px;
            font-weight: bold;
            cursor: pointer;
        }
        button:hover { background: #ff44aa; transform: scale(1.02); }
        .info, .status {
            margin-top: 10px;
            color: #88aaff;
            font-size: 0.8em;
        }
        .status span { font-weight: bold; }
    </style>
</head>
<body>
<div class="game-container">
    <h1>🏒 АЭРОХОККЕЙ | КИСТИ РУК 🏒</h1>
    <div class="canvas-wrapper">
        <canvas id="gameCanvas" width="1200" height="700"></canvas>
    </div>
    <div class="score-board">
        <div>🔴 ЛЕВЫЙ: <span id="leftScore">0</span></div>
        <div>🔵 ПРАВЫЙ: <span id="rightScore">0</span></div>
    </div>
    <button id="resetBtn">🔄 НОВАЯ ИГРА</button>
    <div class="status">
        <div>🖐️ ЛЕВАЯ КИСТЬ: <span id="leftStatus">❌</span></div>
        <div>🖐️ ПРАВАЯ КИСТЬ: <span id="rightStatus">❌</span></div>
    </div>
    <div class="info">
        🥅 ПОКАЖИ КИСТИ РУК В КАМЕРУ — РАКЕТКИ ДВИГАЮТСЯ ЗА ТОБОЙ <br>
        🏒 ОТБИВАЙ ШАЙБУ КИСТЯМИ! ГОЛ ТОЛЬКО В КРАСНУЮ ЗОНУ <br>
        🔥 ЛУЧШЕ ВСЕГО РАБОТАЕТ В CHROME ИЛИ EDGE
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
    const W = 1200, H = 700;
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    let leftScore = 0, rightScore = 0;
    let lastGoalTime = 0;
    const GOAL_COOLDOWN = 600;
    
    let puck = { x: W/2, y: H/2, vx: 3, vy: 2, r: 13 };
    let leftPaddle = { x: 100, y: H/2, r: 38, active: false };
    let rightPaddle = { x: W-100, y: H/2, r: 38, active: false };
    
    let leftFilter = { x: 100, y: H/2 };
    let rightFilter = { x: W-100, y: H/2 };
    let lastLeftSeen = 0, lastRightSeen = 0;
    const LOST_TIMEOUT = 300;
    
    const videoElement = document.createElement('video');
    const hands = new Hands({ locateFile: f => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
    hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
    
    function getHandCenter(landmarks) {
        let wrist = landmarks[0];
        let indexBase = landmarks[5];
        let pinkyBase = landmarks[17];
        let avgX = (wrist.x + indexBase.x + pinkyBase.x) / 3;
        let avgY = (wrist.y + indexBase.y + pinkyBase.y) / 3;
        return { x: avgX, y: avgY };
    }
    
    const SMOOTH = 0.4;
    
    hands.onResults((results) => {
        const now = Date.now();
        let leftDetected = false, rightDetected = false;
        
        if (results.multiHandLandmarks) {
            for (let hand of results.multiHandLandmarks) {
                let center = getHandCenter(hand);
                let isLeft = center.x < 0.5;
                let rawX = W - center.x * W;
                let rawY = center.y * H;
                rawX = Math.min(W - 60, Math.max(60, rawX));
                rawY = Math.min(H - 70, Math.max(70, rawY));
                
                if (isLeft) {
                    leftDetected = true;
                    lastLeftSeen = now;
                    leftFilter.x = leftFilter.x * SMOOTH + rawX * (1 - SMOOTH);
                    leftFilter.y = leftFilter.y * SMOOTH + rawY * (1 - SMOOTH);
                    leftPaddle.x = leftFilter.x;
                    leftPaddle.y = leftFilter.y;
                    leftPaddle.active = true;
                } else {
                    rightDetected = true;
                    lastRightSeen = now;
                    rightFilter.x = rightFilter.x * SMOOTH + rawX * (1 - SMOOTH);
                    rightFilter.y = rightFilter.y * SMOOTH + rawY * (1 - SMOOTH);
                    rightPaddle.x = rightFilter.x;
                    rightPaddle.y = rightFilter.y;
                    rightPaddle.active = true;
                }
            }
        }
        
        if (!leftDetected && now - lastLeftSeen > LOST_TIMEOUT) leftPaddle.active = false;
        if (!rightDetected && now - lastRightSeen > LOST_TIMEOUT) rightPaddle.active = false;
        
        document.getElementById('leftStatus').innerHTML = leftPaddle.active ? '✅' : '❌';
        document.getElementById('rightStatus').innerHTML = rightPaddle.active ? '✅' : '❌';
    });
    
    const camera = new Camera(videoElement, { onFrame: async () => { await hands.send({ image: videoElement }); }, width: W, height: H });
    camera.start();
    
    function checkGoal() {
        const now = Date.now();
        if (now - lastGoalTime < GOAL_COOLDOWN) return false;
        const goalY = (puck.y > H/2 - 70 && puck.y < H/2 + 70);
        if (puck.x - puck.r < 20 && goalY) {
            rightScore++;
            document.getElementById('rightScore').innerText = rightScore;
            lastGoalTime = now;
            resetPuck('right');
            return true;
        }
        if (puck.x + puck.r > W - 20 && goalY) {
            leftScore++;
            document.getElementById('leftScore').innerText = leftScore;
            lastGoalTime = now;
            resetPuck('left');
            return true;
        }
        return false;
    }
    
    function resetPuck(scoredOn) {
        puck.x = W/2;
        puck.y = H/2;
        let angle = (Math.random() - 0.5) * Math.PI / 2.5;
        let speed = 4;
        let direction = scoredOn === 'left' ? 1 : -1;
        puck.vx = Math.cos(angle) * speed * direction;
        puck.vy = Math.sin(angle) * speed;
    }
    
    function updatePhysics() {
        puck.x += puck.vx;
        puck.y += puck.vy;
        
        if (puck.y - puck.r < 0) { puck.y = puck.r; puck.vy = -puck.vy; }
        if (puck.y + puck.r > H) { puck.y = H - puck.r; puck.vy = -puck.vy; }
        
        const goalScored = checkGoal();
        
        if (!goalScored && puck.x - puck.r < 0) { puck.x = puck.r; puck.vx = -puck.vx; }
        if (!goalScored && puck.x + puck.r > W) { puck.x = W - puck.r; puck.vx = -puck.vx; }
        
        let dxl = puck.x - leftPaddle.x;
        let dyl = puck.y - leftPaddle.y;
        let distL = Math.hypot(dxl, dyl);
        if (distL < puck.r + leftPaddle.r && leftPaddle.active) {
            let angle = Math.atan2(dyl, dxl);
            let speed = Math.hypot(puck.vx, puck.vy) + 1;
            puck.vx = Math.cos(angle) * speed;
            puck.vy = Math.sin(angle) * speed;
            puck.x = leftPaddle.x + (puck.r + leftPaddle.r) * Math.cos(angle);
            puck.y = leftPaddle.y + (puck.r + leftPaddle.r) * Math.sin(angle);
        }
        
        let dxr = puck.x - rightPaddle.x;
        let dyr = puck.y - rightPaddle.y;
        let distR = Math.hypot(dxr, dyr);
        if (distR < puck.r + rightPaddle.r && rightPaddle.active) {
            let angle = Math.atan2(dyr, dxr);
            let speed = Math.hypot(puck.vx, puck.vy) + 1;
            puck.vx = Math.cos(angle) * speed;
            puck.vy = Math.sin(angle) * speed;
            puck.x = rightPaddle.x + (puck.r + rightPaddle.r) * Math.cos(angle);
            puck.y = rightPaddle.y + (puck.r + rightPaddle.r) * Math.sin(angle);
        }
        
        let maxSpeed = 11;
        if (Math.abs(puck.vx) > maxSpeed) puck.vx = puck.vx > 0 ? maxSpeed : -maxSpeed;
        if (Math.abs(puck.vy) > maxSpeed) puck.vy = puck.vy > 0 ? maxSpeed : -maxSpeed;
        
        leftPaddle.y = Math.min(H - 50, Math.max(50, leftPaddle.y));
        rightPaddle.y = Math.min(H - 50, Math.max(50, rightPaddle.y));
    }
    
    function resetGame() {
        leftScore = 0;
        rightScore = 0;
        document.getElementById('leftScore').innerText = leftScore;
        document.getElementById('rightScore').innerText = rightScore;
        puck.x = W/2;
        puck.y = H/2;
        puck.vx = 3;
        puck.vy = 2;
        lastGoalTime = 0;
    }
    
    function draw() {
        ctx.clearRect(0, 0, W, H);
        let grad = ctx.createLinearGradient(0, 0, W, H);
        grad.addColorStop(0, "#88aaff");
        grad.addColorStop(1, "#aaccff");
        ctx.fillStyle = grad;
        ctx.fillRect(0, 0, W, H);
        
        ctx.strokeStyle = "#ffffffaa";
        ctx.lineWidth = 3;
        ctx.setLineDash([15, 25]);
        ctx.beginPath();
        ctx.moveTo(W/2, 0);
        ctx.lineTo(W/2, H);
        ctx.stroke();
        ctx.beginPath();
        ctx.arc(W/2, H/2, 70, 0, Math.PI*2);
        ctx.stroke();
        ctx.setLineDash([]);
        
        ctx.fillStyle = "#ff000044";
        ctx.fillRect(0, H/2-70, 20, 140);
        ctx.fillRect(W-20, H/2-70, 20, 140);
        ctx.strokeStyle = "#ff0000";
        ctx.lineWidth = 3;
        ctx.strokeRect(0, H/2-70, 20, 140);
        ctx.strokeRect(W-20, H/2-70, 20, 140);
        
        ctx.shadowBlur = 10;
        ctx.beginPath();
        ctx.arc(leftPaddle.x, leftPaddle.y, leftPaddle.r, 0, Math.PI*2);
        ctx.fillStyle = leftPaddle.active ? "#ff3366" : "#888";
        ctx.fill();
        ctx.beginPath();
        ctx.arc(rightPaddle.x, rightPaddle.y, rightPaddle.r, 0, Math.PI*2);
        ctx.fillStyle = rightPaddle.active ? "#33ff66" : "#888";
        ctx.fill();
        
        ctx.beginPath();
        ctx.arc(puck.x, puck.y, puck.r, 0, Math.PI*2);
        ctx.fillStyle = "#000";
        ctx.fill();
        ctx.beginPath();
        ctx.arc(puck.x, puck.y, puck.r-3, 0, Math.PI*2);
        ctx.fillStyle = "#ff6600";
        ctx.fill();
        ctx.shadowBlur = 0;
    }
    
    function animate() {
        updatePhysics();
        draw();
        requestAnimationFrame(animate);
    }
    
    document.getElementById('resetBtn').onclick = resetGame;
    resetGame();
    animate();
</script>
</body>
</html>
