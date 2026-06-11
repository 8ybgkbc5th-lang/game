<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Zombie Survivor 2D</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #111827;
            color: #ffffff;
            font-family: sans-serif;
            overflow: hidden;
            user-select: none;
            touch-action: none;
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            font-weight: bold;
            font-size: 14px;
            z-index: 10;
            text-shadow: 1px 1px 3px black;
            pointer-events: none;
        }
        canvas {
            display: block;
            background-color: #1f2937;
        }
        /* Віртуальний джойстик */
        #joystick-zone {
            position: absolute;
            bottom: 40px;
            left: 40px;
            width: 120px;
            height: 120px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 50%;
            border: 2px solid rgba(255, 255, 255, 0.3);
            z-index: 10;
        }
        #joystick-stick {
            position: absolute;
            width: 50px;
            height: 50px;
            background: rgba(59, 130, 246, 0.7);
            border-radius: 50%;
            top: 35px;
            left: 35px;
        }
    </style>
</head>
<body>

    <div id="ui">
        <div>HP: <span id="hp-val">100</span> / <span id="max-hp-val">100</span></div>
        <div>LEVEL: <span id="lvl-val">1</span> (EXP: <span id="exp-val">0</span>/10)</div>
        <div>ZOMBIES KILLED: <span id="kills-val">0</span></div>
    </div>

    <div id="joystick-zone">
        <div id="joystick-stick"></div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Підганяємо розмір екрана
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        // Ігрові змінні
        let player = {
            x: canvas.width / 2,
            y: canvas.height / 2,
            radius: 18,
            speed: 3.5,
            hp: 100,
            maxHp: 100,
            level: 1,
            exp: 0,
            nextLevelExp: 10,
            damage: 25
        };

        let zombies = [];
        let bullets = [];
        let gems = [];
        let kills = 0;
        let lastShotTime = 0;
        let shotInterval = 400; // Швидкість стрільби (мс)

        // Джойстик керування
        const joystickZone = document.getElementById('joystick-zone');
        const joystickStick = document.getElementById('joystick-stick');
        let joystickActive = false;
        let joystickStart = { x: 0, y: 0 };
        let moveDir = { x: 0, y: 0 };

        // Обробка дотиків для джойстика
        window.addEventListener('touchstart', (e) => {
            const touch = e.touches[0];
            const rect = joystickZone.getBoundingClientRect();
            // Перевіряємо, чи натиснули в зоні джойстика
            if (touch.clientX >= rect.left && touch.clientX <= rect.right &&
                touch.clientY >= rect.top && touch.clientY <= rect.bottom) {
                joystickActive = true;
                joystickStart = { x: rect.left + 60, y: rect.top + 60 };
            }
        });

        window.addEventListener('touchmove', (e) => {
            if (!joystickActive) return;
            const touch = e.touches[0];
            
            let dx = touch.clientX - joystickStart.x;
            let dy = touch.clientY - joystickStart.y;
            let distance = Math.sqrt(dx * dx + dy * dy);
            
            // Обмежуємо рух стіка межами джойстика
            if (distance > 40) {
                dx = (dx / distance) * 40;
                dy = (dy / distance) * 40;
                distance = 40;
            }

            joystickStick.style.transform = `translate(${dx}px, ${dy}px)`;
            
            // Напрямок руху
            moveDir.x = dx / 40;
            moveDir.y = dy / 40;
        });

        window.addEventListener('touchend', () => {
            joystickActive = false;
            joystickStick.style.transform = `translate(0px, 0px)`;
            moveDir = { x: 0, y: 0 };
        });

        // Спавн зомбі
        function spawnZombie() {
            if (zombies.length > 25) return; // Обмеження кількості
            
            let x, y;
            // З'являються за межами екрана
            if (Math.random() < 0.5) {
                x = Math.random() < 0.5 ? -30 : canvas.width + 30;
                y = Math.random() * canvas.height;
            } else {
                x = Math.random() * canvas.width;
                y = Math.random() < 0.5 ? -30 : canvas.height + 30;
            }

            zombies.push({
                x: x,
                y: y,
                radius: 15,
                hp: 40 + (player.level * 10),
                speed: 1 + Math.random() * 0.8
            });
        }
        setInterval(spawnZombie, 1500);

        // Головний ігровий цикл (Оновлення та Малювання)
        function gameLoop(currentTime) {
            // 1. ОНОВЛЕННЯ ЛОГІКИ
            
            // Рух гравця
            player.x += moveDir.x * player.speed;
            player.y += moveDir.y * player.speed;

            // Обмеження екрана для гравця
            player.x = Math.max(player.radius, Math.min(canvas.width - player.radius, player.x));
            player.y = Math.max(player.radius, Math.min(canvas.height - player.radius, player.y));

            // Автоматична стрільба в найближчого зомбі
            if (zombies.length > 0 && currentTime - lastShotTime > shotInterval) {
                // Шукаємо найближчого
                let nearestZombie = zombies[0];
                let minDist = Infinity;
                
                zombies.forEach(z => {
                    let dist = Math.hypot(z.x - player.x, z.y - player.y);
                    if (dist < minDist) {
                        minDist = dist;
                        nearestZombie = z;
                    }
                });

                // Стріляємо в нього
                let angle = Math.atan2(nearestZombie.y - player.y, nearestZombie.x - player.x);
                bullets.push({
                    x: player.x,
                    y: player.y,
                    dx: Math.cos(angle) * 7,
                    dy: Math.sin(angle) * 7,
                    radius: 5
                });
                lastShotTime = currentTime;
                if (tg.HapticFeedback) tg.HapticFeedback.impactOccurred('light');
            }

            // Рух куль
            bullets.forEach((b, bIdx) => {
                b.x += b.dx;
                b.y += b.dy;
                // Видаляємо кулі за екраном
                if (b.x < 0 || b.x > canvas.width || b.y < 0 || b.y > canvas.height) {
                    bullets.splice(bIdx, 1);
                }
            });

            // Рух зомбі до гравця
            zombies.forEach((z, zIdx) => {
                let angle = Math.atan2(player.y - z.y, player.x - z.x);
                z.x += Math.cos(angle) * z.speed;
                z.y += Math.sin(angle) * z.speed;

                // Зіткнення зомбі з гравцем (нанесення шкоди)
                let distToPlayer = Math.hypot(player.x - z.x, player.y - z.y);
                if (distToPlayer < player.radius + z.radius) {
                    player.hp -= 0.3; // Зменшуємо HP
                    document.getElementById('hp-val').innerText = Math.max(0, Math.floor(player.hp));
                    if (tg.HapticFeedback && Math.random() < 0.1) tg.HapticFeedback.impactOccurred('medium');
                    
                    if (player.hp <= 0) {
                        if (tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('error');
                        tg.showAlert(`Game Over! 💀\nYou survived until level ${player.level} and killed ${kills} zombies!`);
                        // Перезапуск
                        player.hp = player.maxHp;
                        zombies = [];
                        bullets = [];
                        gems = [];
                        kills = 0;
                        player.level = 1;
                        player.exp = 0;
                        player.nextLevelExp = 10;
                        player.damage = 25;
                        document.getElementById('lvl-val').innerText = player.level;
                        document.getElementById('kills-val').innerText = kills;
                        document.getElementById('hp-val').innerText = player.hp;
                    }
                }

                // Зіткнення куль із зомбі
                bullets.forEach((b, bIdx) => {
                    let distToBullet = Math.hypot(b.x - z.x, b.y - z.y);
                    if (distToBullet < z.radius + b.radius) {
                        z.hp -= player.damage;
                        bullets.splice(bIdx, 1);

                        // Якщо зомбі помер
                        if (z.hp <= 0) {
                            kills++;
                            document.getElementById('kills-val').innerText = kills;
                            // Створюємо кристал досвіду на місці зомбі
                            gems.push({ x: z.x, y: z.y, radius: 6 });
                            zombies.splice(zIdx, 1);
                        }
                    }
                });
            });

            // Збір кристалів досвіду (EXP)
            gems.forEach((g, gIdx) => {
                let distToGem = Math.hypot(player.x - g.x, player.y - g.y);
                if (distToGem < player.radius + g.radius) {
                    gems.splice(gIdx, 1);
                    player.exp++;
                    
                    // Перевірка на новий рівень (Level Up)
                    if (player.exp >= player.nextLevelExp) {
                        player.level++;
                        player.exp = 0;
                        player.nextLevelExp = Math.floor(player.nextLevelExp * 1.5);
                        player.maxHp += 15;
                        player.hp = player.maxHp; // Повністю лікуємо при Level Up
                        player.damage += 10; // Збільшуємо шкоду
                        
                        document.getElementById('max-hp-val').innerText = player.maxHp;
                        document.getElementById('hp-val').innerText = player.hp;
                        document.getElementById('lvl-val').innerText = player.level;
                        
                        if (tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('success');
                        tg.showAlert(`🎉 LEVEL UP!\nYou reached Level ${player.level}!\nMax HP and Damage increased!`);
                    }
                    document.getElementById('exp-val').innerText = player.exp;
                }
            });


            // 2. МАЛЮВАННЯ НА CANVAS
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Малюємо кристали досвіду (жовті ромбики)
            gems.forEach(g => {
                ctx.fillStyle = '#f59e0b';
                ctx.beginPath();
                ctx.arc(g.x, g.y, g.radius, 0, Math.PI * 2);
                ctx.fill();
            });

            // Малюємо кулі (жовті маленькі кола)
            bullets.forEach(b => {
                ctx.fillStyle = '#fbbf24';
                ctx.beginPath();
                ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
                ctx.fill();
            });

            // Малюємо зомбі (зелені кола з очима)
            zombies.forEach(z => {
                ctx.fillStyle = '#10b981';
                ctx.beginPath();
                ctx.arc(z.x, z.y, z.radius, 0, Math.PI * 2);
                ctx.fill();
                
                // Мультяшні очі зомбі
                ctx.fillStyle = '#ffffff';
                ctx.beginPath();
                ctx.arc(z.x - 5, z.y - 3, 3, 0, Math.PI * 2);
                ctx.arc(z.x + 5, z.y - 3, 3, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#red';
                ctx.beginPath();
                ctx.arc(z.x - 5, z.y - 3, 1, 0, Math.PI * 2);
                ctx.arc(z.x + 5, z.y - 3, 1, 0, Math.PI * 2);
                ctx.fill();
            });

            // Малюємо нашого крутого героя (синій мультяшний персонаж)
            ctx.fillStyle = '#3b82f6';
            ctx.beginPath();
            ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
            ctx.fill();

            // Біла кепка/пов'язка для стилю
            ctx.fillStyle = '#ffffff';
            ctx.beginPath();
            ctx.arc(player.x, player.y - 8, 10, 0, Math.PI, true);
            ctx.fill();

            // Очі героя
            ctx.fillStyle = '#ffffff';
            ctx.beginPath();
            ctx.arc(player.x - 5, player.y - 1, 4, 0, Math.PI * 2);
            ctx.arc(player.x + 5, player.y - 1, 4, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = '#000000';
            ctx.beginPath();
            ctx.arc(player.x - 5, player.y - 1, 1.5, 0, Math.PI * 2);
            ctx.arc(player.x + 5, player.y - 1, 1.5, 0, Math.PI * 2);
            ctx.fill();

            requestAnimationFrame(gameLoop);
        }

        // Запуск гри
        requestAnimationFrame(gameLoop);
    </script>
</body>
</html>
