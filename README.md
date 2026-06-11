<!DOCTYPE html>
<html lang="uk">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Play and Win!</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>

    <style>
        body {
            background-color: var(--tg-theme-bg-color, #1a1a1a);
            color: var(--tg-theme-text-color, #ffffff);
            font-family: -apple-system, BlinkMacSystemFont, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            padding: 20px;
            box-sizing: border-box;
            text-align: center;
        }

        /* Стильний блок з балами */
        .score-box {
            background: rgba(255, 255, 255, 0.1);
            padding: 15px 30px;
            border-radius: 15px;
            font-size: 20px;
            font-weight: bold;
            margin-bottom: 20px;
            border: 1px solid rgba(255, 255, 255, 0.2);
        }

        #score {
            color: #f39c12;
            font-size: 28px;
        }

        p {
            font-size: 16px;
            opacity: 0.8;
            max-width: 280px;
            margin-bottom: 40px;
        }

        /* Велика ігрова кнопка PLAY */
        .play-btn {
            background-color: var(--tg-theme-button-color, #2481cc);
            color: var(--tg-theme-button-text-color, #ffffff);
            border: none;
            width: 150px;
            height: 150px;
            font-size: 24px;
            border-radius: 50%; /* Робимо її круглою */
            cursor: pointer;
            font-weight: bold;
            text-transform: uppercase;
            box-shadow: 0 8px 20px rgba(0,0,0,0.3);
            transition: transform 0.1s ease;
        }

        .play-btn:active {
            transform: scale(0.9); /* Анімація стискання при натисканні */
        }
    </style>
</head>
<body>

    <h2 id="welcome-text">Welcome! 👋</h2>
    
    <div class="score-box">
        Points: <span id="score">0</span>
    </div>

    <p>Tap PLAY to get points and unlock cool rewards!</p>
    
    <button class="play-btn" id="play-button">PLAY</button>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        // Підставляємо ім'я з Телеграм
        if (tg.initDataUnsafe && tg.initDataUnsafe.user) {
            document.getElementById('welcome-text').innerText = `Welcome, ${tg.initDataUnsafe.user.first_name}! 👋`;
        }

        let score = 0;
        const scoreDisplay = document.getElementById('score');
        const playButton = document.getElementById('play-button');

        // Список крутих призів, які можна випадково виграти
        const prizes = [
            "🔥 Secret Promo Code",
            "💎 +100 Bonus Coins",
            "🎁 Premium Sticker Pack",
            "⚡ Double Multiplier x2"
        ];

        playButton.addEventListener('click', () => {
            // Додаємо +1 бал при кожному кліку
            score++;
            scoreDisplay.innerText = score;

            // Робимо легку вібрацію при кожному кліку
            if (tg.HapticFeedback) {
                tg.HapticFeedback.impactOccurred('light');
            }

            // Коли користувач набиває 10 балів — він "виграє" приз!
            if (score === 10) {
                // Вибираємо випадковий приз із нашого списку
                const randomPrize = prizes[Math.floor(Math.random() * prizes.length)];
                
                // Вібруємо сильніше (успіх!)
                if (tg.HapticFeedback) {
                    tg.HapticFeedback.notificationOccurred('success');
                }

                // Показуємо вікно з виграшем через Телеграм
                tg.showAlert(`🎉 CONGRATULATIONS!\nYou unlocked: ${randomPrize}`);
                
                // Скидаємо лічильник, щоб грати заново
                score = 0;
                scoreDisplay.innerText = score;
            }
        });
    </script>
</body>
</html>
