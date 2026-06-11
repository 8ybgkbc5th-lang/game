<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Boot Clicker</title>

<script src="https://telegram.org/js/telegram-web-app.js"></script>

<style>
body {
    margin: 0;
    font-family: Arial;
    background: linear-gradient(#1e3c72, #2a5298);
    color: white;
    text-align: center;
}

h1 {
    margin-top: 20px;
}

#boot {
    width: 180px;
    margin-top: 40px;
    cursor: pointer;
    transition: transform 0.08s;
}

#boot:active {
    transform: scale(0.9);
}

.panel {
    margin-top: 20px;
}

button {
    margin: 5px;
    padding: 10px;
    cursor: pointer;
    border: none;
    border-radius: 8px;
}

.upgrade {
    background: #ffcc00;
}

.reset {
    background: #ff4444;
    color: white;
}

.stats {
    margin-top: 10px;
}
</style>
</head>

<body>

<h1>BOOT CLICKER</h1>

<img id="boot" src="https://i.imgur.com/7yUvePI.png">

<div class="stats">
    Coins: <span id="coins">0</span><br>
    Power: <span id="power">1</span><br>
    Crit: <span id="crit">10</span>%<br>
    Auto: <span id="auto">0</span><br>
    Prestige: <span id="prestige">0</span>
</div>

<div class="panel">
    <button class="upgrade" onclick="upgradePower()">Upgrade Power</button>
    <button class="upgrade" onclick="upgradeCrit()">Upgrade Crit</button>
    <button class="upgrade" onclick="upgradeAuto()">Upgrade Auto</button>
</div>

<div class="panel">
    <button class="reset" onclick="prestigeReset()">Prestige Reset</button>
</div>

<script>
let tg = window.Telegram.WebApp;
tg.expand();

// GAME STATE
let coins = 0;
let power = 1;
let crit = 0.1;
let auto = 0;
let prestige = 0;

// LOAD SAVE
function load() {
    let save = JSON.parse(localStorage.getItem("boot_save"));
    if (save) {
        coins = save.coins;
        power = save.power;
        crit = save.crit;
        auto = save.auto;
        prestige = save.prestige;
    }
}

// SAVE
function save() {
    localStorage.setItem("boot_save", JSON.stringify({
        coins, power, crit, auto, prestige
    }));
}

// TAP SYSTEM
document.getElementById("boot").onclick = () => {
    let dmg = power;

    if (Math.random() < crit) {
        dmg *= 2;
    }

    coins += dmg * (1 + prestige * 0.5);
    update();
};

// UPGRADES
function upgradePower() {
    let cost = Math.floor(10 * Math.pow(1.15, power));

    if (coins >= cost) {
        coins -= cost;
        power++;
        update();
    }
}

function upgradeCrit() {
    let cost = Math.floor(50 * (crit * 10));

    if (coins >= cost) {
        coins -= cost;
        crit += 0.02;
        update();
    }
}

function upgradeAuto() {
    let cost = Math.floor(100 * (auto + 1));

    if (coins >= cost) {
        coins -= cost;
        auto++;
        update();
    }
}

// PRESTIGE SYSTEM
function prestigeReset() {
    if (coins < 10000) return;

    prestige++;
    coins = 0;
    power = 1;
    crit = 0.1;
    auto = 0;

    update();
}

// AUTO INCOME
setInterval(() => {
    coins += auto * (1 + prestige);
    update();
}, 1000);

// UI UPDATE
function update() {
    document.getElementById("coins").innerText = Math.floor(coins);
    document.getElementById("power").innerText = power;
    document.getElementById("crit").innerText = Math.floor(crit * 100);
    document.getElementById("auto").innerText = auto;
    document.getElementById("prestige").innerText = prestige;

    save();
}

// INIT
load();
update();
</script>

</body>
</html>
