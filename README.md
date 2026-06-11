<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Toy Highway Racer</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #0b0f19;
            overflow: hidden;
            font-family: sans-serif;
            user-select: none;
            touch-action: none;
        }
        canvas {
            display: block;
            width: 100vw;
            height: 100vh;
        }
        #score-board {
            position: absolute;
            top: 15px;
            left: 50%;
            transform: translateX(-50%);
            font-size: 24px;
            font-weight: bold;
            color: #ffffff;
            background: rgba(0, 0, 0, 0.4);
            padding: 8px 20px;
            border-radius: 20px;
            border: 3px solid #ffffff;
            text-shadow: 2px 2px 0px #000;
            z-index: 10;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="score-board">SCORE: <span id="score-val">0</span></div>
    <canvas id="raceCanvas"></canvas>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        const canvas = document.getElementById('raceCanvas');
        const ctx = canvas.getContext('2d');

        // Адаптація під екран телефону
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        // Палітра кольорів (Toy Style)
        const COLORS = {
            sky: '#4facfe', skyDark: '#00f2fe', // Градієнт неба
            mountain: '#334155',
            city: '#1e293b',
            containerRed: '#ef4444', containerBlue: '#3b82f6',
            road: '#475569', roadLine: '#facc15',
            player: '#dc2626', enemy1: '#fbbf24', enemy2: '#a855f7',
            outline: '#000000'
        };

        // НАЛАШТУВАННЯ ШВИДКОСТІ ТА РУХУ
        let globalSpeed = 5;
        let score = 0;
        let gameActive = true;

        // Окремі змінні для Паралаксу (Логіка пункту 5)
        let bgOffset1 = 0; // Найдальший фон (гори)
        let bgOffset2 = 0; // Середній фон (місто/будівлі)
        let bgOffset3 = 0; // Ближній фон (рекламні щити, контейнери)
        let roadOffset = 0; // Дорога (рухається найшвидше)

        // Гравець (Іграшкова фура)
        let player = {
            x: canvas.width / 2,
            y: canvas.height - 120,
            width: 50,
            height: 85,
            targetX: canvas.width / 2,
            turnSpeed: 0.15,
            bounceTimer: 0
        };

        // Трафік (інші машинки)
        let traffic = [];
        function spawnEnemy() {
            if (!gameActive) return;
            const lanes = [canvas.width * 0.25, canvas.width * 0.4, canvas.width * 0.55, canvas.width * 0.7];
            const randomLane = lanes[Math.floor(Math.random() * lanes.length)];
            
            // Перевірка, щоб машини не спавнились одна на одній
            if (traffic.some(e => Math.abs(e.y) < 150 && Math.abs(e.x - randomLane) < 20)) return;

            traffic.push({
                x: randomLane,
                y: -100,
                width: 42,
                height: 70,
                color: Math.random() > 0.5 ? COLORS.enemy1 : COLORS.enemy2,
                speed: 2 + Math.random() * 2
            });
        }
        setInterval(spawnEnemy, 1200);

        // Об'єкти ближнього фону (Рекламні щити та контейнери для відчуття швидкості)
        let sideObjects = [];
        function spawnSideObject() {
            if (!gameActive) return;
            let isLeft = Math.random() > 0.5;
            sideObjects.push({
                x: isLeft ? canvas.width * 0.05 : canvas.width * 0.85,
                y: -120,
                width: 55,
                height: 45,
                type: Math.random() > 0.4 ? 'billboard' : 'container',
                color: Math.random() > 0.5 ? COLORS.containerRed : COLORS.containerBlue
            });
        }
        setInterval(spawnSideObject, 800);

        // КЕРУВАННЯ ТАПКАМИ НА ТЕЛЕФОНІ
        window.addEventListener('touchstart', (e) => {
            if (!gameActive) {
                // Перезапуск гри при тапі після аварії
                gameActive = true;
                score = 0;
                globalSpeed = 5;
                traffic = [];
                sideObjects = [];
                player.x = canvas.width / 2;
                player.targetX = canvas.width / 2;
                document.getElementById('score-board').style.display = 'block';
                return;
            }
            const touchX = e.touches[0].clientX;
            // Якщо тапнули зліва — зміщуємось ліворуч, якщо справа — праворуч
            if (touchX < canvas.width / 2) {
                player.targetX = Math.max(canvas.width * 0.25, player.x - canvas.width * 0.15);
            } else {
                player.targetX = Math.min(canvas.width * 0.7, player.x + canvas.width * 0.15);
            }
        });

        // МАЛЮВАННЯ ОБ'ЄКТІВ З ТОВСТИМ КОНТУРОМ (Пункт 2 - Мультяшний стиль)
        function drawCardboardObj(x, y, w, h, color, radius = 8) {
            ctx.save();
            ctx.lineWidth = 4;
            ctx.strokeStyle = COLORS.outline;
            ctx.fillStyle = color;
            
            // Малюємо закруглений пластиковий кубик
            ctx.beginPath();
            ctx.roundRect(x, y, w, h, radius);
            ctx.fill();
            ctx.stroke();
            ctx.restore();
        }

        // ГОЛОВНИЙ ЦИКЛ ГРИ
        function updateAndRender() {
            // 1. ЛОГІКА РУХУ ТА ПАРАЛАКСУ
            if (gameActive) {
                globalSpeed += 0.002; // Поступове прискорення
                score += 1;
                document.getElementById('score-val').innerText = Math.floor(score / 10);

                // Нарахування зміщень за твоїми коефіцієнтами (Пункт 5)
                bgOffset1 += globalSpeed * 0.05; // Гори (повільно)
                bgOffset2 += globalSpeed * 0.15; // Місто (середнє)
                roadOffset += globalSpeed;       // Дорога (максимальна швидкість = 1.0)

                // Плавний рух гравця до цілі
                player.x += (player.targetX - player.x) * player.turnSpeed;
                player.bounceTimer += 0.15; // Для ефекту Arcade Bounce
            }

            // --- МАЛЮВАННЯ ---
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // ШАР 1: Небо + Гори (Паралакс 0.05)
            let skyGrad = ctx.createLinearGradient(0, 0, 0, canvas.height * 0.3);
            skyGrad.addColorStop(0, COLORS.sky);
            skyGrad.addColorStop(1, COLORS.skyDark);
            ctx.fillStyle = skyGrad;
            ctx.fillRect(0, 0, canvas.width, canvas.height * 0.35);

            ctx.fillStyle = COLORS.mountain;
            for (let i = -1; i < 3; i++) {
                let xPos = (i * canvas.width) - (bgOffset1 % canvas.width);
                ctx.beginPath();
                ctx.moveTo(xPos, canvas.height * 0.35);
                ctx.lineTo(xPos + canvas.width * 0.5, canvas.height * 0.22);
                ctx.lineTo(xPos + canvas.width, canvas.height * 0.35);
                ctx.fill();
            }

            // ШАР 2: Прості міські кубики (Паралакс 0.15)
            ctx.fillStyle = COLORS.city;
            for (let i = -1; i < 5; i++) {
                let xPos = (i * 120) - (bgOffset2 % 120);
                // Будівлі без зайвого шуму (Пункт 2 і 3)
                ctx.fillRect(xPos, canvas.height * 0.28, 90, canvas.height * 0.07);
                ctx.lineWidth = 3;
                ctx.strokeStyle = COLORS.outline;
                ctx.strokeRect(xPos, canvas.height * 0.28, 90, canvas.height * 0.07);
            }

            // ШАР 4: Дорога (2D перспектива з градієнтом, без шуму)
            ctx.fillStyle = COLORS.road;
            ctx.fillRect(canvas.width * 0.15, canvas.height * 0.35, canvas.width * 0.7, canvas.height * 0.65);
            
            // Чорні відбійники/бордюри по краях дороги
            ctx.fillStyle = COLORS.outline;
            ctx.fillRect(canvas.width * 0.14, canvas.height * 0.35, 10, canvas.height * 0.65);
            ctx.fillRect(canvas.width * 0.85, canvas.height * 0.35, 10, canvas.height * 0.65);

            // Розмітка дороги (рух текстури вниз для імітації швидкості)
            ctx.fillStyle = COLORS.roadLine;
            let lineY = (roadOffset % 80) - 80;
            while (lineY < canvas.height) {
                if (lineY > canvas.height * 0.35) {
                    ctx.fillRect(canvas.width * 0.49, lineY, 8, 40);
                    ctx.fillRect(canvas.width * 0.33, lineY, 4, 40);
                    ctx.fillRect(canvas.width * 0.66, lineY, 4, 40);
                }
                lineY += 80;
            }

            // ШАР 3: Ближній фон (Рекламні щити та контейнери збоку траси)
            sideObjects.forEach((obj, idx) => {
                if (gameActive) obj.y += globalSpeed;
                
                if (obj.type === 'billboard') {
                    // Ніжка щита
                    ctx.fillRect(obj.x + 25, obj.y + 20, 6, 40);
                    // Сам щит (яскравий неоновий пластик)
                    drawCardboardObj(obj.x, obj.y, obj.width, obj.height, '#f43f5e', 4);
                } else {
                    // Іграшковий морський контейнер біля дороги
                    drawCardboardObj(obj.x, obj.y, obj.width, obj.height, obj.color, 6);
                }

                if (obj.y > canvas.height) sideObjects.splice(idx, 1);
            });

            // РУХ ТА МАЛЮВАННЯ ТРАФІКУ
            traffic.forEach((enemy, idx) => {
                if (gameActive) enemy.y += (globalSpeed - enemy.speed);

                // Малюємо машинку суперника в стилі 2-3 тони
                drawCardboardObj(enemy.x - enemy.width/2, enemy.y, enemy.width, enemy.height, enemy.color);
                
                // Спрощене велике скло кабіни
                ctx.fillStyle = '#ffffff';
                ctx.fillRect(enemy.x - enemy.width/2 + 6, enemy.y + 10, enemy.width - 12, 15);
                ctx.strokeStyle = COLORS.outline;
                ctx.strokeRect(enemy.x - enemy.width/2 + 6, enemy.y + 10, enemy.width - 12, 15);

                // Перевірка зіткнення (Аварія)
                if (gameActive && 
                    player.x - player.width/2 < enemy.x + enemy.width/2 &&
                    player.x + player.width/2 > enemy.x - enemy.width/2 &&
                    player.y < enemy.y + enemy.height &&
                    player.y + player.height > enemy.y) {
                    
                    // КРАШ ГРИ
                    gameActive = false;
                    if (tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('error');
                    tg.showAlert(`💥 CRASH!\nYour Score: ${Math.floor(score / 10)} points.\nTap screen to restart!`);
                }

                if (enemy.y > canvas.height + 100) traffic.splice(idx, 1);
            });

            // МАЛЮВАННЯ НАШОЇ ІГРАШКОВОЇ ФУРИ (Toy Arcade Bounce)
            let bounceY = gameActive ? Math.sin(player.bounceTimer) * 3 : 0; // Пружний рух вгору-вниз (Пункт 6)
            
            // Основний корпус великої кабіни фури
            drawCardboardObj(player.x - player.width/2, player.y + bounceY, player.width, player.height, COLORS.player, 12);
            
            // Величезне мультяшне лобове скло (Пункт 6 - перебільшені форми)
            ctx.fillStyle = '#93c5fd';
            ctx.fillRect(player.x - player.width/2 + 6, player.y + 15 + bounceY, player.width - 12, 22);
            ctx.strokeRect(player.x - player.width/2 + 6, player.y + 15 + bounceY, player.width - 12, 22);
            
            // Жовті пластикові фари
            ctx.fillStyle = '#facc15';
            ctx.fillRect(player.x - player.width/2 + 6, player.y + 70 + bounceY, 10, 8);
            ctx.fillRect(player.x + player.width/2 - 16, player.y + 70 + bounceY, 10, 8);

            // Текст-інструкція, якщо розбився
            if (!gameActive) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
                ctx.fillRect(0, canvas.height/2 - 40, canvas.width, 80);
                ctx.fillStyle = '#ffffff';
                ctx.font = 'bold 20px sans-serif';
                ctx.textAlign = 'center';
                ctx.fillText('GAME OVER', canvas.width/2, canvas.height/2 - 5);
                ctx.font = '16px sans-serif';
                ctx.fillText('Tap screen to try again', canvas.width/2, canvas.height/2 + 22);
            }

            requestAnimationFrame(updateAndRender);
        }

        // Запуск
        requestAnimationFrame(updateAndRender);
    </script>
</body>
</html>
