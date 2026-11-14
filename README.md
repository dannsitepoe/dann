<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Neon Jumper - Mini Game</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        body {
            background-color: #121212;
            color: #ffffff;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
            user-select: none;
        }
        h1 {
            margin-bottom: 10px;
            font-size: 28px;
            letter-spacing: 2px;
            color: #00ffcc;
            text-shadow: 0 0 10px rgba(0, 255, 204, 0.5);
        }
        #game-container {
            position: relative;
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.8);
            border: 2px solid #333;
            border-radius: 8px;
            overflow: hidden;
        }
        canvas {
            display: block;
            background: linear-gradient(180deg, #1a1a2e 0%, #16213e 100%);
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
            pointer-events: none;
        }
        #overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.75);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            text-align: center;
        }
        #overlay h2 {
            font-size: 36px;
            color: #ff0055;
            margin-bottom: 10px;
        }
        #overlay p {
            font-size: 16px;
            color: #cccccc;
            margin-bottom: 20px;
        }
        button {
            padding: 12px 28px;
            font-size: 18px;
            font-weight: bold;
            color: #121212;
            background-color: #00ffcc;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: transform 0.1s, background-color 0.2s;
        }
        button:hover {
            background-color: #00cca3;
            transform: scale(1.05);
        }
        .hidden {
            display: none !important;
        }
    </style>
</head>
<body>

    <h1>NEON JUMPER</h1>

    <div id="game-container">
        <canvas id="gameCanvas" width="800" height="350"></canvas>
        
        <div id="ui-layer">
            <div>Skor: <span id="score">0</span></div>
            <div>Terbaik: <span id="highScore">0</span></div>
        </div>

        <div id="overlay">
            <h2 id="overlay-title">SIAP BERMAIN?</h2>
            <p id="overlay-desc">Tekan <b>SPASI</b>, <b>PANAH ATAS</b>, atau <b>KLIK / SENTUH LAYAR</b> untuk melompat.</p>
            <button id="start-btn">MULAI GAME</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const highScoreEl = document.getElementById('highScore');
        const overlay = document.getElementById('overlay');
        const overlayTitle = document.getElementById('overlay-title');
        const overlayDesc = document.getElementById('overlay-desc');
        const startBtn = document.getElementById('start-btn');

        // Pengaturan Dasar
        let isPlaying = false;
        let score = 0;
        let highScore = localStorage.getItem('neonJumperHighScore') || 0;
        highScoreEl.textContent = highScore;
        let gameSpeed = 6;
        let animationId;

        const groundHeight = 50;

        // Objek Pemain
        const player = {
            x: 80,
            y: canvas.height - groundHeight - 40,
            width: 40,
            height: 40,
            dy: 0,
            jumpForce: 13,
            gravity: 0.65,
            grounded: false,
            color: '#00ffcc',

            draw() {
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 15;
                ctx.shadowColor = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                ctx.shadowBlur = 0;
            },

            update() {
                // Fisika Lompatan
                if (this.y + this.height < canvas.height - groundHeight) {
                    this.dy += this.gravity;
                    this.grounded = false;
                } else {
                    this.dy = 0;
                    this.grounded = true;
                    this.y = canvas.height - groundHeight - this.height;
                }

                this.y += this.dy;
                this.draw();
            },

            jump() {
                if (this.grounded) {
                    this.dy = -this.jumpForce;
                    this.grounded = false;
                }
            }
        };

        // Rintangan
        let obstacles = [];
        let spawnTimer = 0;

        class Obstacle {
            constructor() {
                this.width = Math.floor(Math.random() * 20) + 25;
                this.height = Math.floor(Math.random() * 35) + 35;
                this.x = canvas.width;
                this.y = canvas.height - groundHeight - this.height;
                this.color = '#ff0055';
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 10;
                ctx.shadowColor = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                ctx.shadowBlur = 0;
            }

            update() {
                this.x -= gameSpeed;
                this.draw();
            }
        }

        // Kontrol Input
        function handleJump(e) {
            if (e.type === 'keydown' && e.code !== 'Space' && e.code !== 'ArrowUp') return;
            if (e.type === 'keydown') e.preventDefault();
            
            if (isPlaying) {
                player.jump();
            }
        }

        window.addEventListener('keydown', handleJump);
        canvas.addEventListener('mousedown', handleJump);
        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault();
            handleJump(e);
        }, { passive: false });

        // Fungsi Utama Game Loop
        function updateGame() {
            if (!isPlaying) return;

            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Gambar Lantai
            ctx.fillStyle = '#0f3460';
            ctx.fillRect(0, canvas.height - groundHeight, canvas.width, groundHeight);
            ctx.strokeStyle = '#00ffcc';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(0, canvas.height - groundHeight);
            ctx.lineTo(canvas.width, canvas.height - groundHeight);
            ctx.stroke();

            // Update Pemain
            player.update();

            // Munculkan Rintangan
            spawnTimer--;
            if (spawnTimer <= 0) {
                obstacles.push(new Obstacle());
                spawnTimer = Math.floor(Math.random() * 50) + 55 - Math.min(gameSpeed * 2, 25);
            }

            // Update & Cek Tabrakan Rintangan
            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i];
                obs.update();

                // Deteksi Tabrakan (AABB)
                if (
                    player.x < obs.x + obs.width &&
                    player.x + player.width > obs.x &&
                    player.y < obs.y + obs.height &&
                    player.y + player.height > obs.y
                ) {
                    gameOver();
                    return;
                }

                // Hapus rintangan yang keluar layar & tambah skor
                if (obs.x + obs.width < 0) {
                    obstacles.splice(i, 1);
                    score++;
                    scoreEl.textContent = score;

                    // Tingkatkan kecepatan secara bertahap
                    if (score % 5 === 0) {
                        gameSpeed += 0.3;
                    }
                }
            }

            animationId = requestAnimationFrame(updateGame);
        }

        function startGame() {
            isPlaying = true;
            score = 0;
            gameSpeed = 6;
            obstacles = [];
            spawnTimer = 30;
            player.y = canvas.height - groundHeight - player.height;
            player.dy = 0;
            
            scoreEl.textContent = score;
            overlay.classList.add('hidden');
            updateGame();
        }

        function gameOver() {
            isPlaying = false;
            cancelAnimationFrame(animationId);

            if (score > highScore) {
                highScore = score;
                localStorage.setItem('neonJumperHighScore', highScore);
                highScoreEl.textContent = highScore;
            }

            overlayTitle.textContent = 'GAME OVER!';
            overlayDesc.innerHTML = `Skor Akhir Kamu: <b>${score}</b><br>Klik tombol di bawah untuk main lagi.`;
            startBtn.textContent = 'MAIN LAGI';
            overlay.classList.remove('hidden');
        }

        startBtn.addEventListener('click', startGame);
    </script>
</body>
</html>
