<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cyber Jump - Endless Runner Game</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
        }
        body {
            background-color: #121212;
            color: #ffffff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
        }
        #game-container {
            position: relative;
            box-shadow: 0 10px 30px rgba(0, 255, 204, 0.2);
            border: 2px solid #00ffcc;
            border-radius: 8px;
            overflow: hidden;
            background: linear-gradient(180deg, #1a1a2e 0%, #16213e 100%);
        }
        canvas {
            display: block;
        }
        #ui-layer {
            position: absolute;
            top: 15px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            font-size: 18px;
            font-weight: bold;
            letter-spacing: 1px;
            pointer-events: none;
        }
        .score-box span {
            color: #00ffcc;
        }
        #overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(18, 18, 18, 0.8);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            text-align: center;
            transition: opacity 0.2s;
        }
        #overlay h1 {
            font-size: 36px;
            color: #00ffcc;
            margin-bottom: 10px;
            text-transform: uppercase;
            letter-spacing: 3px;
        }
        #overlay p {
            font-size: 16px;
            color: #cccccc;
            margin-bottom: 25px;
        }
        #start-btn {
            padding: 12px 30px;
            font-size: 18px;
            font-weight: bold;
            color: #121212;
            background-color: #00ffcc;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: transform 0.1s, background-color 0.2s;
        }
        #start-btn:hover {
            background-color: #00cca3;
            transform: scale(1.05);
        }
        #start-btn:active {
            transform: scale(0.95);
        }
        .hidden {
            display: none !important;
        }
        .controls-hint {
            margin-top: 15px;
            font-size: 13px;
            color: #888;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas" width="800" height="350"></canvas>
        
        <div id="ui-layer">
            <div class="score-box">SKOR: <span id="score">0</span></div>
            <div class="score-box">TERTINGGI: <span id="high-score">0</span></div>
        </div>

        <div id="overlay">
            <h1 id="overlay-title">CYBER JUMP</h1>
            <p id="overlay-desc">Tekan SPASI, PANAH ATAS, atau KLIK LAYAR untuk melompat!</p>
            <button id="start-btn">MULAI MAIN</button>
        </div>
    </div>
    <div class="controls-hint">Tips: Tahan tombol lompat sedikit lebih lama untuk lompatan lebih tinggi.</div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const highScoreEl = document.getElementById('high-score');
        const overlay = document.getElementById('overlay');
        const overlayTitle = document.getElementById('overlay-title');
        const overlayDesc = document.getElementById('overlay-desc');
        const startBtn = document.getElementById('start-btn');

        // Pengaturan Game
        const GRAVITY = 0.65;
        const GROUND_Y = 290;
        let gameSpeed = 6;
        let score = 0;
        let highScore = localStorage.getItem('cyberJumpHighScore') || 0;
        let isGameOver = false;
        let isPlaying = false;
        let animationId;
        let frameCount = 0;

        highScoreEl.textContent = Math.floor(highScore);

        // Objek Pemain
        const player = {
            x: 80,
            y: GROUND_Y - 40,
            width: 35,
            height: 40,
            dy: 0,
            jumpForce: -13,
            grounded: false,
            color: '#00ffcc',

            draw() {
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 15;
                ctx.shadowColor = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                
                // Mata karakter (detail visual)
                ctx.fillStyle = '#121212';
                ctx.shadowBlur = 0;
                ctx.fillRect(this.x + 22, this.y + 8, 8, 8);
            },

            update() {
                // Fisika lompatan & gravitasi
                this.y += this.dy;

                if (this.y + this.height < GROUND_Y) {
                    this.dy += GRAVITY;
                    this.grounded = false;
                } else {
                    this.dy = 0;
                    this.grounded = true;
                    this.y = GROUND_Y - this.height;
                }

                this.draw();
            },

            jump() {
                if (this.grounded) {
                    this.dy = this.jumpForce;
                    this.grounded = false;
                }
            }
        };

        // Array Rintangan
        let obstacles = [];

        class Obstacle {
            constructor() {
                this.width = 25 + Math.random() * 20;
                this.height = 35 + Math.random() * 35;
                this.x = canvas.width;
                this.y = GROUND_Y - this.height;
                this.color = '#ff2a6d';
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 12;
                ctx.shadowColor = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                ctx.shadowBlur = 0;
            }

            update() {
                this.x -= gameSpeed;
                this.draw();
            }
        }

        // Latar Belakang (Partikel Bintang)
        const stars = Array.from({ length: 40 }, () => ({
            x: Math.random() * canvas.width,
            y: Math.random() * (GROUND_Y - 20),
            size: Math.random() * 2 + 1,
            speed: Math.random() * 1.5 + 0.5
        }));

        function drawBackground() {
            // Bintang bergerak
            ctx.fillStyle = 'rgba(255, 255, 255, 0.5)';
            stars.forEach(star => {
                if (isPlaying) {
                    star.x -= star.speed;
                    if (star.x < 0) star.x = canvas.width;
                }
                ctx.fillRect(star.x, star.y, star.size, star.size);
            });

            // Garis Tanah
            ctx.strokeStyle = '#00ffcc';
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(0, GROUND_Y);
            ctx.lineTo(canvas.width, GROUND_Y);
            ctx.stroke();
        }

        // Kontrol Input
        function handleJump(e) {
            if (e.type === 'keydown' && (e.code === 'Space' || e.code === 'ArrowUp')) {
                e.preventDefault();
                if (isPlaying) player.jump();
            } else if (e.type === 'mousedown' || e.type === 'touchstart') {
                if (isPlaying && e.target !== startBtn) {
                    player.jump();
                }
            }
        }

        window.addEventListener('keydown', handleJump);
        canvas.addEventListener('mousedown', handleJump);
        canvas.addEventListener('touchstart', handleJump, { passive: true });

        // Logika Utama Game
        function spawnObstacles() {
            // Jarak antar rintangan acak namun tetap adil
            let minSpawnRate = Math.max(55, 110 - Math.floor(gameSpeed * 3));
            if (frameCount % minSpawnRate === 0 && Math.random() > 0.25) {
                obstacles.push(new Obstacle());
            }
        }

        function checkCollision(rect1, rect2) {
            // Sedikit toleransi hitbox (4px) agar terasa lebih adil bagi pemain
            const padding = 4;
            return (
                rect1.x + padding < rect2.x + rect2.width - padding &&
                rect1.x + rect1.width - padding > rect2.x + padding &&
                rect1.y + padding < rect2.y + rect2.height - padding &&
                rect1.y + rect1.height - padding > rect2.y + padding
            );
        }

        function gameOver() {
            isPlaying = false;
            isGameOver = true;
            cancelAnimationFrame(animationId);

            if (score > highScore) {
                highScore = Math.floor(score);
                localStorage.setItem('cyberJumpHighScore', highScore);
                highScoreEl.textContent = highScore;
            }

            overlayTitle.textContent = 'GAME OVER!';
            overlayTitle.style.color = '#ff2a6d';
            overlayDesc.textContent = `Skor Akhir Kamu: ${Math.floor(score)}`;
            startBtn.textContent = 'MAIN LAGI';
            overlay.classList.remove('hidden');
        }

        function resetGame() {
            obstacles = [];
            score = 0;
            gameSpeed = 6;
            frameCount = 0;
            player.y = GROUND_Y - player.height;
            player.dy = 0;
            isGameOver = false;
            isPlaying = true;
            scoreEl.textContent = '0';
            overlay.classList.add('hidden');
            animate();
        }

        function animate() {
            if (!isPlaying) return;

            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawBackground();

            player.update();
            spawnObstacles();

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i];
                obs.update();

                // Deteksi Tabrakan
                if (checkCollision(player, obs)) {
                    gameOver();
                    return;
                }

                // Hapus rintangan yang sudah lewat layar
                if (obs.x + obs.width < 0) {
                    obstacles.splice(i, 1);
                }
            }

            // Tambah skor & tingkatkan kecepatan secara bertahap
            score += 0.15;
            scoreEl.textContent = Math.floor(score);
            gameSpeed += 0.0015;
            frameCount++;

            animationId = requestAnimationFrame(
