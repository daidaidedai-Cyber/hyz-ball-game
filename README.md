<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆球冒险游戏 - 在线版</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            background-color: #f0f0f0;
            font-family: 'Arial', sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            overflow: hidden;
        }
        
        .game-wrapper {
            position: relative;
            background-color: white;
            border-radius: 10px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            overflow: hidden;
        }
        
        #gameCanvas {
            display: block;
            background-color: white;
        }
        
        #gameUI {
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            padding: 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: linear-gradient(to bottom, rgba(255,255,255,0.9), transparent);
            pointer-events: none;
        }
        
        #hearts {
            font-size: 28px;
            color: #ff4757;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
        }
        
        #score {
            font-size: 24px;
            font-weight: bold;
            color: #333;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
        }
        
        #gameOver {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: rgba(0, 0, 0, 0.9);
            color: white;
            padding: 40px;
            border-radius: 15px;
            text-align: center;
            display: none;
            backdrop-filter: blur(10px);
        }
        
        #gameOver h2 {
            margin-top: 0;
            font-size: 36px;
            margin-bottom: 20px;
            color: #ff4757;
        }
        
        #finalScore {
            color: #ffa502;
            font-size: 28px;
        }
        
        #restartBtn {
            padding: 15px 30px;
            font-size: 20px;
            background: linear-gradient(45deg, #4CAF50, #45a049);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            margin-top: 25px;
            transition: all 0.3s ease;
            box-shadow: 0 4px 15px rgba(76, 175, 80, 0.3);
        }
        
        #restartBtn:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(76, 175, 80, 0.4);
        }
        
        #instructions {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: rgba(0, 0, 0, 0.7);
            color: white;
            padding: 15px 25px;
            border-radius: 8px;
            font-size: 14px;
            text-align: center;
        }
        
        .share-buttons {
            margin-top: 20px;
            display: flex;
            gap: 10px;
            justify-content: center;
        }
        
        .share-btn {
            padding: 8px 16px;
            font-size: 14px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: all 0.3s ease;
        }
        
        .share-wechat {
            background-color: #07c160;
            color: white;
        }
        
        .share-weibo {
            background-color: #ff8200;
            color: white;
        }
        
        .share-link {
            background-color: #1da1f2;
            color: white;
        }
    </style>
</head>
<body>
    <div class="game-wrapper">
        <div id="gameUI">
            <div id="hearts">❤️❤️❤️❤️❤️❤️❤️❤️❤️❤️</div>
            <div id="score">积分: 0</div>
        </div>
        
        <canvas id="gameCanvas" width="800" height="600"></canvas>
        
        <div id="instructions">
            使用 WASD 键移动 | 触碰怪物击杀它们 | 避开子弹！
        </div>
        
        <div id="gameOver">
            <h2>游戏结束！</h2>
            <p>最终积分: <span id="finalScore">0</span></p>
            <button id="restartBtn">重新开始游戏</button>
            <div class="share-buttons">
                <button class="share-btn share-link" onclick="shareGame()">分享游戏链接</button>
                <button class="share-btn share-wechat" onclick="shareToSocial('wechat')">分享到微信</button>
            </div>
        </div>
    </div>

    <script>
        // 游戏核心代码
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const heartsDisplay = document.getElementById('hearts');
        const scoreDisplay = document.getElementById('score');
        const gameOverDiv = document.getElementById('gameOver');
        const finalScoreSpan = document.getElementById('finalScore');
        const restartBtn = document.getElementById('restartBtn');
        const instructionsDiv = document.getElementById('instructions');

        // 游戏状态
        let score = 0;
        let gameRunning = true;
        let gameStarted = false;

        // 玩家对象
        let player = {
            x: canvas.width / 2,
            y: canvas.height / 2,
            radius: 15,
            speed: 5,
            health: 10,
            color: '#0066cc'
        };

        // 按键状态
        const keys = {
            w: false, a: false, s: false, d: false, r: false
        };

        // 游戏对象数组
        let monsters = [];
        let bullets = [];
        let particles = [];

        // 隐藏说明
        setTimeout(() => {
            instructionsDiv.style.opacity = '0';
            setTimeout(() => {
                instructionsDiv.style.display = 'none';
            }, 500);
        }, 5000);

        // 怪物类
        class Monster {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.size = 25;
                this.speed = 1.5;
                this.color = '#ff8c00';
                this.shootCooldown = 0;
                this.isDying = false;
                this.dyingAnimation = 0;
            }

            update() {
                if (this.isDying) {
                    this.dyingAnimation++;
                    if (this.dyingAnimation > 20) return false;
                    return true;
                }

                const dx = player.x - this.x;
                const dy = player.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance > 0) {
                    this.x += (dx / distance) * this.speed;
                    this.y += (dy / distance) * this.speed;
                }

                if (this.shootCooldown > 0) this.shootCooldown--;
                if (this.shootCooldown === 0 && Math.random() < 0.02) {
                    this.shoot();
                    this.shootCooldown = 120;
                }
                return true;
            }

            shoot() {
                const dx = player.x - this.x;
                const dy = player.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                if (distance > 0) {
                    bullets.push(new Bullet(this.x, this.y, (dx / distance) * 2, (dy / distance) * 2));
                }
            }

            draw() {
                if (this.isDying) {
                    ctx.save();
                    ctx.globalAlpha = 1 - (this.dyingAnimation / 20);
                    const scale = 1 + (this.dyingAnimation / 20);
                    ctx.translate(this.x, this.y);
                    ctx.scale(scale, scale);
                    ctx.translate(-this.x, -this.y);
                }

                ctx.fillStyle = this.color;
                ctx.fillRect(this.x - this.size/2, this.y - this.size/2, this.size, this.size);

                if (this.isDying) ctx.restore();
            }

            startDying() {
                this.isDying = true;
                for (let i = 0; i < 8; i++) {
                    particles.push(new Particle(this.x, this.y));
                }
            }
        }

        // 子弹类
        class Bullet {
            constructor(x, y, vx, vy) {
                this.x = x; this.y = y; this.vx = vx; this.vy = vy;
                this.radius = 4; this.color = '#ff4500';
            }

            update() { this.x += this.vx; this.y += this.vy; }

            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2); ctx.fill();
            }

            isOffScreen() {
                return this.x < 0 || this.x > canvas.width || this.y < 0 || this.y > canvas.height;
            }
        }

        // 粒子类
        class Particle {
            constructor(x, y) {
                this.x = x; this.y = y;
                this.vx = (Math.random() - 0.5) * 8;
                this.vy = (Math.random() - 0.5) * 8;
                this.size = Math.random() * 4 + 2;
                this.life = 30; this.maxLife = 30;
                this.color = '#ff8c00';
            }

            update() {
                this.x += this.vx; this.y += this.vy;
                this.vx *= 0.98; this.vy *= 0.98;
                this.life--;
                return this.life > 0;
            }

            draw() {
                ctx.save();
                ctx.globalAlpha = this.life / this.maxLife;
                ctx.fillStyle = this.color;
                ctx.beginPath(); ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2); ctx.fill();
                ctx.restore();
            }
        }

        // 事件监听
        document.addEventListener('keydown', (e) => {
            if (keys.hasOwnProperty(e.key.toLowerCase())) {
                keys[e.key.toLowerCase()] = true;
            }
            if (e.key.toLowerCase() === 'r' && !gameRunning) restartGame();
        });

        document.addEventListener('keyup', (e) => {
            if (keys.hasOwnProperty(e.key.toLowerCase())) {
                keys[e.key.toLowerCase()] = false;
            }
        });

        restartBtn.addEventListener('click', restartGame);

        // 游戏逻辑
        function updatePlayer() {
            if (keys.w && player.y - player.radius > 0) player.y -= player.speed;
            if (keys.s && player.y + player.radius < canvas.height) player.y += player.speed;
            if (keys.a && player.x - player.radius > 0) player.x -= player.speed;
            if (keys.d && player.x + player.radius < canvas.width) player.x += player.speed;
        }

        function spawnMonster() {
            if (Math.random() < 0.02) {
                const angle = Math.random() * Math.PI * 2;
                const distance = 200 + Math.random() * 100;
                const x = player.x + Math.cos(angle) * distance;
                const y = player.y + Math.sin(angle) * distance;
                if (x > 0 && x < canvas.width && y > 0 && y < canvas.height) {
                    monsters.push(new Monster(x, y));
                }
            }
        }

        function checkCollisions() {
            for (let i = monsters.length - 1; i >= 0; i--) {
                const monster = monsters[i];
                if (monster.isDying) continue;
                const dx = player.x - monster.x;
                const dy = player.y - monster.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                if (distance < player.radius + monster.size / 2) {
                    monster.startDying();
                    score++;
                    updateScore();
                }
            }

            for (let i = bullets.length - 1; i >= 0; i--) {
                const bullet = bullets[i];
                const dx = player.x - bullet.x;
                const dy = player.y - bullet.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                if (distance < player.radius + bullet.radius) {
                    bullets.splice(i, 1);
                    player.health--;
                    updateHearts();
                    if (player.health <= 0) gameOver();
                }
            }
        }

        function gameOver() {
            gameRunning = false;
            finalScoreSpan.textContent = score;
            gameOverDiv.style.display = 'block';
        }

        function restartGame() {
            score = 0; gameRunning = true;
            player = { x: canvas.width / 2, y: canvas.height / 2, radius: 15, speed: 5, health: 10, color: '#0066cc' };
            monsters = []; bullets = []; particles = [];
            updateScore(); updateHearts();
            gameOverDiv.style.display = 'none';
            gameLoop();
        }

        function updateHearts() {
            let hearts = '';
            for (let i = 0; i < player.health; i++) hearts += '❤️';
            heartsDisplay.textContent = hearts;
        }

        function updateScore() {
            scoreDisplay.textContent = '积分: ' + score;
        }

        // 分享功能
        function shareGame() {
            const gameUrl = window.location.href;
            navigator.clipboard.writeText(gameUrl).then(() => {
                alert('游戏链接已复制到剪贴板！可以分享给朋友了！');
            }).catch(() => {
                prompt('请复制以下链接分享给朋友：', gameUrl);
            });
        }

        function shareToSocial(platform) {
            const gameUrl = window.location.href;
            const shareText = `我在圆球冒险游戏中获得了${score}分！快来挑战吧！`;
            
            if (platform === 'wechat') {
                alert('请使用微信扫一扫功能，或复制链接分享给微信好友');
                navigator.clipboard.writeText(gameUrl);
            }
        }

        // 游戏循环
        function gameLoop() {
            if (!gameRunning) return;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            updatePlayer();
            spawnMonster();

            for (let i = monsters.length - 1; i >= 0; i--) {
                if (!monsters[i].update()) monsters.splice(i, 1);
            }

            for (let i = bullets.length - 1; i >= 0; i--) {
                bullets[i].update();
                if (bullets[i].isOffScreen()) bullets.splice(i, 1);
            }

            for (let i = particles.length - 1; i >= 0; i--) {
                if (!particles[i].update()) particles.splice(i, 1);
            }

            checkCollisions();

            // 绘制
            ctx.fillStyle = player.color;
            ctx.beginPath(); ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2); ctx.fill();
            monsters.forEach(monster => monster.draw());
            bullets.forEach(bullet => bullet.draw());
            particles.forEach(particle => particle.draw());

            requestAnimationFrame(gameLoop);
        }

        // 开始游戏
        gameLoop();
    </script>
</body>
</html>
