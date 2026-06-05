<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CYBER GLITCH: 40 PROTOCOLS</title>
    <style>
        * {
            box-sizing: border-box; margin: 0; padding: 0;
            user-select: none; -webkit-user-select: none; touch-action: none;
        }
        body {
            background-color: #020205; font-family: 'Courier New', Courier, monospace;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            height: 100vh; overflow: hidden; padding: 10px;
        }
        #gameContainer {
            position: relative; width: 100%; max-width: 800px; aspect-ratio: 2 / 1;
        }
        canvas {
            background: #04040c; border: 3px solid #00f0ff;
            box-shadow: 0 0 30px rgba(0, 240, 255, 0.25); border-radius: 16px;
            display: block; width: 100%; height: 100%;
        }
        #mobileControls {
            width: 100%; max-width: 800px; display: none; justify-content: space-between;
            margin-top: 12px; padding: 0 5px; gap: 20px;
        }
        .btn {
            flex: 1; height: 70px; border-radius: 14px; font-family: 'Courier New', monospace;
            font-size: 18px; font-weight: bold; display: flex; align-items: center; justify-content: center;
            background: rgba(0, 0, 0, 0.7); text-transform: uppercase; letter-spacing: 2px;
        }
        #btnJump {
            border: 3px solid #00f0ff; color: #00f0ff;
            box-shadow: 0 0 15px rgba(0, 240, 255, 0.2);
        }
        #btnJump:active { background: rgba(0, 240, 255, 0.3); box-shadow: 0 0 25px #00f0ff; }
        #btnDuck {
            border: 3px solid #ff007f; color: #ff007f;
            box-shadow: 0 0 15px rgba(255, 0, 127, 0.2);
        }
        #btnDuck:active { background: rgba(255, 0, 127, 0.3); box-shadow: 0 0 25px #ff007f; }
        
        .controls-info {
            color: #444466; margin-top: 8px; text-align: center; font-size: 11px; letter-spacing: 1px;
        }
    </style>
</head>
<body>

<div id="gameContainer">
    <canvas id="gameCanvas" width="800" height="400"></canvas>
</div>

<div id="mobileControls">
    <div id="btnJump" class="btn">ZIPLA (UP)</div>
    <div id="btnDuck" class="btn">EĞİL (DOWN)</div>
</div>

<div class="controls-info">
    KLAVYE: YUKARI (Zıpla) / AŞAĞI (Eğil) | MOBİL: Butonlar sadece koşu esnasında aktifleşir.
</div>

<script>
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");
const mobileControlsDiv = document.getElementById("mobileControls");

// Ses Motoru (Hatalardan Arındırılmış Web Audio API)
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
function playTone(freq, type, duration) {
    try {
        if (audioCtx.state === 'suspended') return;
        let osc = audioCtx.createOscillator(); let gain = audioCtx.createGain();
        osc.type = type; osc.frequency.value = freq;
        gain.gain.setValueAtTime(0.03, audioCtx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.00001, audioCtx.currentTime + duration);
        osc.connect(gain); gain.connect(audioCtx.destination);
        osc.start(); osc.stop(audioCtx.currentTime + duration);
    } catch(e){}
}

// Global Oyun Durumu
let gameState = "MENU";
let totalCoins = 500; // Başlangıç sermayesi sızdırıldı
let score = 0; let highScore = 0; let gameSpeed = 6; let spawnTimer = 0; let gameFrame = 0;

let upgrades = { magnet: 0, shieldTime: 0, glitchLuck: 0 };
let upgradePrices = { magnet: [200, 500, 1200], shieldTime: [250, 600, 1500], glitchLuck: [400, 900, 2000] };

let glitchActive = false; let glitchTimer = 0; let glitchType = ""; let screenShake = 0;
let activePowerUp = null; let powerUpTimer = 0; let powerUpsInWorld = [];

// Biyom Protokolleri
const biomes = [
    { name: "SİBER ŞEHİR GİRİŞİ", bg: "#040410", floor: "#220066", accent: "#00f0ff", bColor: "#0f0f26" },
    { name: "NEON ASİT ORMANI", bg: "#021006", floor: "#005511", accent: "#39ff14", bColor: "#081a0d" },
    { name: "RADYAKTİF YAĞMURLAR", bg: "#101002", floor: "#555500", accent: "#ffff00", bColor: "#1a1a08" },
    { name: "DEEP WEB VERİ MERKEZİ", bg: "#06010b", floor: "#3a0073", accent: "#bc13fe", bColor: "#120024" },
    { name: "SİBER CEHENNEM ÇEKİRDEĞİ", bg: "#140202", floor: "#770000", accent: "#ff3131", bColor: "#240808" },
    { name: "MATRIX KOD ODASI", bg: "#000000", floor: "#003300", accent: "#00ff66", bColor: "#001200" },
    { name: "PLAZMA BUZULLARI", bg: "#05121b", floor: "#003355", accent: "#00ffff", bColor: "#0b1e2b" },
    { name: "KROM TOZ ÇÖLÜ", bg: "#140c05", floor: "#552200", accent: "#ff5e00", bColor: "#24150a" },
    { name: "TOKSİK ATIK FABRİKASI", bg: "#0a1202", floor: "#335500", accent: "#adff2f", bColor: "#121f05" },
    { name: "SONSUZ SİBER UZAY (LİMİT)", bg: "#010105", floor: "#111122", accent: "#ffffff", bColor: "#060610" }
];

function getCurrentBiome() {
    let idx = Math.floor(score / 15);
    return idx >= biomes.length ? biomes[biomes.length - 1] : biomes[idx];
}

// Görev Matrisi
let quests = [];
function generateQuests() {
    quests = [
        { text: "TEK KOŞUDA " + (25 + Math.floor(Math.random()*15)) + " SCORE YAP", target: 25, check: function() { return score >= this.target; }, reward: 300, done: false },
        { text: "KUŞLARIN ALTINDAN 2 KEZ EĞİL", target: 2, current: 0, check: function() { return this.current >= this.target; }, reward: 200, done: false },
        { text: "SİBER MARKET PROTOKOLÜNÜ GENİŞLET", check: () => characters.filter(c => c.unlocked).length > 1, reward: 250, done: false }
    ];
}
generateQuests();

let boss = { x: 850, y: 100, width: 80, height: 120, speedY: 3.5, health: 120, maxHealth: 120 };
let bossProjectiles = []; let bossTimer = 0;

// Çevresel Varlık Matrisleri
const BIRD1 = [[0,0,1,1,0,0,0,0],[0,1,1,1,1,0,0,0],[1,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,1],[0,0,0,1,1,1,0,0]];
const BIRD2 = [[0,0,0,0,0,0,0,0],[0,0,1,1,1,0,0,0],[1,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0]];
const M_BOSS = [[1,1,1,1,1,1,1,1],[1,0,0,1,1,0,0,1],[1,0,0,1,1,0,0,1],[1,1,1,1,1,1,1,1],[0,0,1,0,0,1,0,0],[0,1,1,1,1,1,1,0],[0,1,0,1,1,0,1,0],[1,1,1,1,1,1,1,1]];

// --- BENZERSİZ 40 ADET SİBER KOŞUCU MATRİS VE FİYATLANDIRMA VERİTABANI ---
const characters = [
    { id: 0, name: "CYBER DINO", color: "#00f0ff", mult: 1.0, price: 0, unlocked: true,
      r1: [[0,0,0,1,1,1,1,0],[0,0,0,1,1,1,1,1],[0,0,0,1,1,0,0,0],[1,0,1,1,1,1,1,0],[1,1,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,0,1,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0],[0,0,0,1,1,1,1,1],[1,0,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,1],[0,0,1,1,1,1,1,0],[0,0,1,0,0,1,0,0]] },
    
    { id: 1, name: "NEON NEKO", color: "#ff007f", mult: 1.2, price: 150, unlocked: false,
      r1: [[1,0,0,0,0,0,1,0],[1,1,1,1,1,1,1,0],[1,0,1,0,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,1,1,1,1,1],[0,1,0,1,0,1,0,0],[0,1,0,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,0,0,1,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,0,1,0,1,0,0],[0,1,0,0,0,1,0,0],[0,0,0,0,0,0,0,0]] },
    
    { id: 2, name: "ROBO-88", color: "#39ff14", mult: 1.5, price: 300, unlocked: false,
      r1: [[0,0,0,1,0,0,0,0],[0,1,1,1,1,1,0,0],[0,1,0,1,0,1,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,0,1,1,0,0],[0,1,0,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,0,1,1,1,1,0,0],[0,1,0,1,0,1,0,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,1,1,0,0],[0,1,0,0,0,1,0,0],[0,0,0,0,0,0,0,0]] },
    
    { id: 3, name: "XENOMORPH", color: "#ffff00", mult: 1.8, price: 450, unlocked: false,
      r1: [[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[0,0,1,1,0,0,0,0],[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[0,1,0,0,1,0,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,1,1],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[0,1,0,0,1,0,0,0],[0,0,0,0,0,0,0,0]] },
    
    { id: 4, name: "SHADOW V", color: "#ff3131", mult: 2.2, price: 600, unlocked: false,
      r1: [[0,0,1,1,1,0,0,0],[0,1,1,0,1,1,0,0],[0,0,1,1,1,0,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,0,1,0,0],[1,0,0,0,1,0,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,0,1,1,1,0,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,0,1,0,0],[1,0,0,0,1,0,0,0],[0,0,0,0,0,0,0,0]] },
    
    { id: 5, name: "MATRIX GOLEM", color: "#00ff66", mult: 2.6, price: 800, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,1,0,0,0,0,1,1],[1,1,0,1,1,0,1,1],[1,1,0,1,1,0,1,1],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[1,1,0,1,1,0,1,1],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 6, name: "CYBER PUNK", color: "#ea00d9", mult: 3.0, price: 1000, unlocked: false,
      r1: [[0,1,1,0,0,1,1,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 7, name: "GRID RUNNER", color: "#00bfff", mult: 3.5, price: 1300, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,0,0,0,0,1,0],[0,1,1,0,0,1,1,0],[0,0,1,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,0,1,0,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 8, name: "BIT HARVESTER", color: "#710094", mult: 4.0, price: 1600, unlocked: false,
      r1: [[0,1,0,1,0,1,0,0],[1,1,1,1,1,1,1,0],[1,0,0,0,0,0,1,0],[1,1,1,1,1,1,1,0],[1,1,0,1,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[1,1,0,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,0],[1,0,0,0,0,0,1,0],[1,1,0,1,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 9, name: "KINETIC RAY", color: "#ffaa00", mult: 4.5, price: 2000, unlocked: false,
      r1: [[0,0,0,1,1,0,0,0],[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 10, name: "DATA REAPER", color: "#32cd32", mult: 5.0, price: 2500, unlocked: false,
      r1: [[0,1,1,1,1,0,0,0],[1,1,1,1,1,1,0,0],[1,0,1,1,0,1,0,0],[1,1,1,1,1,1,0,0],[0,1,1,1,1,0,0,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[1,1,0,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,0,0],[1,0,1,1,0,1,0,0],[0,1,1,1,1,0,0,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 11, name: "PROXY GHOST", color: "#e6e6fa", mult: 5.5, price: 3000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,0,0,0,0,1,0],[1,0,1,0,0,1,0,1],[1,0,0,0,0,0,0,1],[1,0,1,1,1,1,0,1],[1,0,0,0,0,0,0,1],[0,1,1,1,1,1,1,0],[0,1,0,1,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,0,0,0,0,1,0],[1,0,1,0,0,1,0,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,1,0,1,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 12, name: "QUANTUM ENFORCER", color: "#ff4500", mult: 6.0, price: 3500, unlocked: false,
      r1: [[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[1,0,1,1,1,1,0,1],[1,0,1,0,0,1,0,1],[1,0,1,1,1,1,0,1],[1,1,0,0,0,0,1,1],[0,1,1,1,1,1,1,0],[0,0,1,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,0,0,0,0,1,1],[1,0,1,1,1,1,0,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,0,1,0,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 13, name: "KERNEL OVERLORD", color: "#9400d3", mult: 6.5, price: 4200, unlocked: false,
      r1: [[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,0,0,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 14, name: "GLITCH RESISTOR", color: "#00e5ff", mult: 7.0, price: 5000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,0,0,1,1,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 15, name: "VECTOR SYNDICATE", color: "#da70d6", mult: 7.5, price: 6000, unlocked: false,
      r1: [[1,0,1,1,1,1,0,1],[1,1,1,0,0,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,0,0,1,0,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,0,0,0,0,1,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,0,0,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 16, name: "PHANTOM LINK", color: "#40e0d0", mult: 8.0, price: 7200, unlocked: false,
      r1: [[0,0,0,1,1,0,0,0],[0,0,1,1,1,1,0,0],[0,1,0,1,1,0,1,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,0,1,0,0,1,0,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,0,1,1,1,1,0,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 17, name: "LASER SENTINEL", color: "#ff0000", mult: 8.5, price: 8500, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,0,0,0,0,0,0,1],[1,0,1,1,1,1,0,1],[1,0,1,1,1,1,0,1],[1,0,1,1,1,1,0,1],[1,0,0,0,0,0,0,1],[1,1,1,1,1,1,1,1],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,0,0,0,0,1],[1,0,1,1,1,1,0,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,0,0,0,0,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 18, name: "STATIC SHIVER", color: "#b0c4de", mult: 9.0, price: 10000, unlocked: false,
      r1: [[0,1,0,1,0,1,0,0],[0,1,1,1,1,1,0,0],[1,1,0,1,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,1,1,1,0,0],[0,1,0,0,0,1,0,0],[0,1,1,0,1,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 19, name: "NEO EXILE", color: "#ff69b4", mult: 10.0, price: 12000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,1,1,0,1,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,0,0,0,0,1,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,0,0,1,1,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 20, name: "VOID DEVIANT", color: "#4b0082", mult: 11.0, price: 15500, unlocked: false,
      r1: [[0,1,1,1,1,1,1,0],[1,0,0,1,1,0,0,1],[1,1,1,1,1,1,1,1],[0,1,0,1,1,0,1,0],[0,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,0,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,1,1,0,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 21, name: "SHELL SHREDDER", color: "#ffd700", mult: 12.0, price: 18000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,1,0,0,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,0,0,0,0,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 22, name: "LOGIC BOMB", color: "#ff6347", mult: 13.0, price: 21000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,0,1,1,0,1,0],[1,1,1,1,1,1,1,1],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,0,1,1,0,1,0],[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 23, name: "ROUTER CHASER", color: "#00efbc", mult: 14.0, price: 25000, unlocked: false,
      r1: [[0,1,1,0,0,1,1,0],[1,1,1,1,1,1,1,1],[1,0,0,1,1,0,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[1,0,0,1,1,0,0,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 24, name: "TITAN STRIKER", color: "#ff00ff", mult: 15.0, price: 30000, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1],[0,1,1,0,0,1,1,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 25, name: "CYBER DRAKE", color: "#adff2f", mult: 16.5, price: 36000, unlocked: false,
      r1: [[0,0,0,1,1,1,1,1],[1,0,0,1,1,1,1,1],[1,1,1,1,1,0,0,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,1,0],[0,0,1,1,0,1,1,0],[0,0,1,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,1,1,1,1,1],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,1,0],[0,0,1,1,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 26, name: "HEX DEFENDER", color: "#00ced1", mult: 18.0, price: 42000, unlocked: false,
      r1: [[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[1,0,0,0,0,1,1,0],[1,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[1,1,0,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,0],[1,0,0,0,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[1,1,0,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 27, name: "SOCKET VIPER", color: "#ff8c00", mult: 20.0, price: 50000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,0,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 28, name: "FIBER ASSASSIN", color: "#ba55d3", mult: 22.0, price: 60000, unlocked: false,
      r1: [[0,0,1,1,1,0,0,0],[0,1,1,1,1,1,0,0],[1,1,0,1,0,1,1,0],[1,1,1,1,1,1,1,0],[1,0,1,1,1,0,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[1,0,0,0,0,1,0,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,0,0],[1,1,1,1,1,1,1,0],[1,0,1,1,1,0,1,0],[0,1,1,1,1,1,0,0],[0,1,1,0,1,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 29, name: "MAC CRACKER", color: "#f4a460", mult: 25.0, price: 75000, unlocked: false,
      r1: [[1,1,1,1,1,1,1,0],[1,0,0,0,0,0,1,1],[1,0,1,1,1,0,1,1],[1,1,1,1,1,1,1,0],[1,1,0,0,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,0,1,1,0,0],[1,1,0,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,0,0,0,1,1],[1,1,1,1,1,1,1,0],[1,1,0,0,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,0,1,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 30, name: "CLOUD STRIDER", color: "#f0f8ff", mult: 30.0, price: 90000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,0,0,0,0,1,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,0,0,0,0,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 31, name: "DAEMON WRAITH", color: "#8b0000", mult: 35.0, price: 110000, unlocked: false,
      r1: [[1,0,0,1,1,0,0,1],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[1,0,0,0,0,0,0,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 32, name: "MALWARE ARCHON", color: "#9932cc", mult: 40.0, price: 135000, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,0,0,0,0,0,0,1],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[1,1,1,0,0,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,0,0,0,0,0,0,1],[1,1,1,1,1,1,1,1],[1,1,1,0,0,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 33, name: "SUBNET BLADE", color: "#00fa9a", mult: 45.0, price: 160000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[1,1,1,0,0,0,0,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,1,1,1,1,1,0],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 34, name: "CRITTICAL ROOT", color: "#ff1493", mult: 50.0, price: 200000, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,1,0,0,0,0,1,1],[1,1,0,1,1,0,1,1],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,0,0,0,0,1,1],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 35, name: "HYPER THREAD", color: "#00bcff", mult: 60.0, price: 250000, unlocked: false,
      r1: [[0,1,0,1,0,1,0,1],[1,1,1,1,1,1,1,1],[0,1,0,1,0,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,1,0,0,0,0,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[0,1,0,1,0,1,0,1],[1,1,1,1,1,1,1,1],[0,0,1,1,1,1,0,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 36, name: "NODE REBEL", color: "#ccff00", mult: 70.0, price: 320000, unlocked: false,
      r1: [[0,1,1,1,1,1,0,0],[1,1,0,0,0,1,1,0],[1,0,1,1,1,0,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[1,1,0,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,0,0,0,1,1,0],[1,1,1,1,1,1,1,0],[0,1,1,1,1,1,0,0],[0,1,1,1,1,1,1,0],[0,1,0,0,0,1,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 37, name: "OVERCLOCK BEAST", color: "#ff3300", mult: 85.0, price: 400000, unlocked: false,
      r1: [[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[1,1,0,0,0,0,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[1,0,1,1,1,1,0,1],[1,1,1,1,1,1,1,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 38, name: "SYNAPSE SPECTRE", color: "#df00ff", mult: 100.0, price: 500000, unlocked: false,
      r1: [[0,0,1,1,1,1,0,0],[0,1,0,0,0,0,1,0],[1,0,1,1,1,1,0,1],[1,0,1,0,0,1,0,1],[1,0,1,1,1,1,0,1],[1,0,0,0,0,0,0,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0]],
      duck: [[0,0,0,0,0,0,0,0],[0,1,0,0,0,0,1,0],[1,0,1,1,1,1,0,1],[1,0,1,1,1,1,0,1],[0,1,1,1,1,1,1,0],[0,1,1,0,0,1,1,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] },

    { id: 39, name: "MATRIX MASTER X", color: "#00ff66", mult: 250.0, price: 1000000, unlocked: false,
      r1: [[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1],[1,1,0,0,0,0,1,1],[1,1,0,1,1,0,1,1],[1,1,0,1,1,0,1,1],[1,1,0,0,0,0,1,1],[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1]],
      duck: [[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1],[1,1,0,0,0,0,1,1],[1,1,0,1,1,0,1,1],[1,1,1,1,1,1,1,1],[1,1,1,1,1,1,1,1],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0]] }
];

let selectedCharIndex = 0; let equippedCharIndex = 0;

const dino = { x: 80, y: 320, baseY: 320, width: 48, height: 48, velocityY: 0, gravity: 0.75, jumpForce: -13.5, isGrounded: true, isDucking: false };
let obstacles = [], particles = [], stars = [], buildings = [];

// Çevresel Yıldızlar ve Şehir Yapısı
for (let i = 0; i < 40; i++) stars.push({ x: Math.random() * canvas.width, y: Math.random() * 180, size: Math.random() * 2 + 1, speed: Math.random() * 0.2 + 0.1 });
for (let i = 0; i < 8; i++) buildings.push({ x: i * 115, width: Math.random() * 40 + 55, height: Math.random() * 110 + 40, speed: 0.4 });

// UI Buton Haritaları
let menuButtons = { 
    play: { x: 300, y: 135, w: 200, h: 36 }, 
    chars: { x: 300, y: 180, w: 200, h: 36 }, 
    upgrades: { x: 300, y: 225, w: 200, h: 36 },
    quests: { x: 300, y: 270, w: 200, h: 36 } 
};
let charMenuButtons = { prev: { x: 80, y: 220, w: 70, h: 42 }, next: { x: 650, y: 220, w: 70, h: 42 }, select: { x: 310, y: 320, w: 180, h: 45 }, back: { x: 30, y: 25, w: 90, h: 35 } };
let upgMenuButtons = { 
    b1: { x: 70, y: 165, w: 190, h: 50, type: "magnet" },
    b2: { x: 305, y: 165, w: 190, h: 50, type: "shieldTime" },
    b3: { x: 540, y: 165, w: 190, h: 50, type: "glitchLuck" },
    back: { x: 350, y: 320, w: 100, h: 40 }
};

// Akıllı Mobil Buton Tetikleyicisi (Hatalı Menü Görüntülenmesi Engellendi)
function updateButtonVisibility() {
    if (gameState === "PLAYING" || gameState === "BOSS_ALARM" || gameState === "BOSS_BATTLE") {
        mobileControlsDiv.style.display = "flex";
    } else {
        mobileControlsDiv.style.display = "none";
    }
}

// Canvas Koordinat Dönüştürücü (Tüm Cihazlarda Dokunmatik Senkronizasyonu)
function handleMenuClicks(clientX, clientY) {
    if (audioCtx.state === 'suspended') audioCtx.resume();
    let rect = canvas.getBoundingClientRect();
    let clickX = (clientX - rect.left) * (canvas.width / rect.width);
    let clickY = (clientY - rect.top) * (canvas.height / rect.height);

    if (gameState === "MENU") {
        if (clickX >= menuButtons.play.x && clickX <= menuButtons.play.x + menuButtons.play.w && clickY >= menuButtons.play.y && clickY <= menuButtons.play.y + menuButtons.play.h) resetGame();
        else if (clickX >= menuButtons.chars.x && clickX <= menuButtons.chars.x + menuButtons.chars.w && clickY >= menuButtons.chars.y && clickY <= menuButtons.chars.y + menuButtons.chars.h) { gameState = "CHAR_SELECT"; playTone(400, "square", 0.1); }
        else if (clickX >= menuButtons.upgrades.x && clickX <= menuButtons.upgrades.x + menuButtons.upgrades.w && clickY >= menuButtons.upgrades.y && clickY <= menuButtons.upgrades.y + menuButtons.upgrades.h) { gameState = "UPGRADE_MENU"; playTone(420, "square", 0.1); }
        else if (clickX >= menuButtons.quests.x && clickX <= menuButtons.quests.x + menuButtons.quests.w && clickY >= menuButtons.quests.y && clickY <= menuButtons.quests.y + menuButtons.quests.h) { gameState = "QUEST_PANEL"; playTone(450, "square", 0.1); }
    } 
    else if (gameState === "CHAR_SELECT") {
        let c = characters[selectedCharIndex];
        if (clickX >= charMenuButtons.back.x && clickX <= charMenuButtons.back.x + charMenuButtons.back.w && clickY >= charMenuButtons.back.y && clickY <= charMenuButtons.back.y + charMenuButtons.back.h) { gameState = "MENU"; playTone(300, "square", 0.1); }
        else if (clickX >= charMenuButtons.prev.x && clickX <= charMenuButtons.prev.x + charMenuButtons.prev.w && clickY >= charMenuButtons.prev.y && clickY <= charMenuButtons.prev.y + charMenuButtons.prev.h) { selectedCharIndex = (selectedCharIndex - 1 + characters.length) % characters.length; playTone(500, "sine", 0.04); }
        else if (clickX >= charMenuButtons.next.x && clickX <= charMenuButtons.next.x + charMenuButtons.next.w && clickY >= charMenuButtons.next.y && clickY <= charMenuButtons.next.y + charMenuButtons.next.h) { selectedCharIndex = (selectedCharIndex + 1) % characters.length; playTone(500, "sine", 0.04); }
        else if (clickX >= charMenuButtons.select.x && clickX <= charMenuButtons.select.x + charMenuButtons.select.w && clickY >= charMenuButtons.select.y && clickY <= charMenuButtons.select.y + charMenuButtons.select.h) {
            if (c.unlocked) { equippedCharIndex = selectedCharIndex; gameState = "MENU"; playTone(600, "triangle", 0.12); }
            else if (totalCoins >= c.price) { totalCoins -= c.price; c.unlocked = true; equippedCharIndex = selectedCharIndex; playTone(880, "square", 0.25); }
        }
    } 
    else if (gameState === "UPGRADE_MENU" || gameState === "QUEST_PANEL") {
        if (clickX >= upgMenuButtons.back.x && clickX <= upgMenuButtons.back.x + upgMenuButtons.back.w && clickY >= upgMenuButtons.back.y && clickY <= upgMenuButtons.back.y + upgMenuButtons.back.h) { gameState = "MENU"; playTone(300, "square", 0.1); }
        
        if (gameState === "UPGRADE_MENU") {
            ["magnet", "shieldTime", "glitchLuck"].forEach(type => {
                let btn = type === "magnet" ? upgMenuButtons.b1 : (type === "shieldTime" ? upgMenuButtons.b2 : upgMenuButtons.b3);
                if (clickX >= btn.x && clickX <= btn.x + btn.w && clickY >= btn.y && clickY <= btn.y + btn.h) {
                    let lvl = upgrades[type];
                    if (lvl < 3) {
                        let cost = upgradePrices[type][lvl];
                        if (totalCoins >= cost) { totalCoins -= cost; upgrades[type]++; playTone(700, "triangle", 0.18); }
                    }
                }
            });
        }
    }
    else if (gameState === "GAMEOVER") { generateQuests(); gameState = "MENU"; }
    updateButtonVisibility();
}

canvas.addEventListener("mousedown", (e) => { handleMenuClicks(e.clientX, e.clientY); });
canvas.addEventListener("touchstart", (e) => { e.preventDefault(); handleMenuClicks(e.touches[0].clientX, e.touches[0].clientY); }, { passive: false });

const btnJump = document.getElementById("btnJump");
const btnDuck = document.getElementById("btnDuck");

function triggerJump() {
    if ((gameState === "PLAYING" || gameState === "BOSS_BATTLE") && dino.isGrounded) {
        dino.velocityY = dino.jumpForce; dino.isGrounded = false; playTone(260, "square", 0.08);
    }
}
btnJump.addEventListener("touchstart", (e) => { e.preventDefault(); triggerJump(); });
btnJump.addEventListener("mousedown", (e) => { triggerJump(); });

btnDuck.addEventListener("touchstart", (e) => { e.preventDefault(); if (gameState === "PLAYING" || gameState === "BOSS_BATTLE") dino.isDucking = true; });
btnDuck.addEventListener("touchend", (e) => { e.preventDefault(); dino.isDucking = false; });
btnDuck.addEventListener("mousedown", (e) => { if (gameState === "PLAYING" || gameState === "BOSS_BATTLE") dino.isDucking = true; });
btnDuck.addEventListener("mouseup", () => { dino.isDucking = false; });
btnDuck.addEventListener("mouseleave", () => { dino.isDucking = false; });

window.addEventListener("keydown", (e) => {
    if ((gameState === "PLAYING" || gameState === "BOSS_BATTLE") && (e.code === "Space" || e.code === "ArrowUp") && dino.isGrounded) { dino.velocityY = dino.jumpForce; dino.isGrounded = false; playTone(260, "square", 0.08); }
    if (e.code === "ArrowDown") dino.isDucking = true;
    if (gameState === "GAMEOVER" && e.code === "Space") { generateQuests(); gameState = "MENU"; updateButtonVisibility(); }
});
window.addEventListener("keyup", (e) => { if(e.code === "ArrowDown") dino.isDucking = false; });

function drawPixelMatrix(matrix, x, y, width, height, color) {
    if(!matrix || !matrix[0]) return;
    let pX = width / matrix[0].length, pY = height / matrix.length; ctx.fillStyle = color;
    for (let r = 0; r < matrix.length; r++) {
        for (let c = 0; c < matrix[r].length; c++) {
            if (matrix[r][c] === 1) ctx.fillRect(x + c * pX, y + r * pY, pX + 0.4, pY + 0.4);
        }
    }
}

function createParticles(x, y, color) {
    for (let i = 0; i < 6; i++) particles.push({ x: x, y: y, vx: (Math.random() - 0.5) * 4, vy: (Math.random() - 0.5) * 4, size: Math.random() * 3 + 1, life: 14, color: color });
}

function spawnObstacle() {
    if (gameState === "BOSS_BATTLE") return;
    let type = Math.random() > 0.45 ? "cactus" : "bird"; if (score < 6) type = "cactus";
    if (type === "cactus") {
        let size = Math.random() * 10 + 22; obstacles.push({ type: "cactus", x: canvas.width, y: 368 - size, width: size, height: size, color: "#ff007f" });
    } else {
        obstacles.push({ type: "bird", x: canvas.width, y: Math.random() > 0.5 ? 255 : 295, width: 36, height: 24, color: "#cc00ff" });
    }
}

function spawnPowerUp() {
    if (Math.random() > 0.4) return;
    let pType = Math.random() > 0.5 ? "SHIELD" : "X2"; powerUpsInWorld.push({ type: pType, x: canvas.width, y: 235, width: 25, height: 25, color: pType === "SHIELD" ? "#39ff14" : "#ffff00" });
}

function resetGame() {
    obstacles = []; particles = []; powerUpsInWorld = []; bossProjectiles = [];
    score = 0; gameSpeed = 6; gameState = "PLAYING"; activePowerUp = null; glitchActive = false; screenShake = 0;
    dino.y = dino.baseY; dino.velocityY = 0; dino.isGrounded = true; dino.isDucking = false;
    boss.x = 850; boss.health = boss.maxHealth; bossTimer = 0; playTone(440, "triangle", 0.1);
    updateButtonVisibility();
}

let lastEarnedCoins = 0;
function endTheGame() {
    let activeChar = characters[equippedCharIndex];
    
    // --- 20'ŞER COIN EKONOMİ MOTORU: ARTIK GERÇEKTEN DEĞERLİSİNİZ ---
    let baseCoins = score * 20; 
    
    let finalCoins = baseCoins * activeChar.mult;
    if (activePowerUp === "X2") finalCoins *= 2;
    if (glitchActive && glitchType === "GOLD_RUSH") finalCoins *= 3;
    finalCoins = Math.floor(finalCoins); lastEarnedCoins = finalCoins; totalCoins += finalCoins;
    quests.forEach(q => { if (!q.done && q.check()) { totalCoins += q.reward; q.done = true; } });
    if (score > highScore) highScore = score; gameState = "GAMEOVER"; playTone(140, "sawtooth", 0.35);
    updateButtonVisibility();
}

function gameLoop() {
    requestAnimationFrame(gameLoop); ctx.save();
    if (screenShake > 0) { ctx.translate((Math.random() - 0.5) * screenShake, (Math.random() - 0.5) * screenShake); screenShake *= 0.88; }
    ctx.clearRect(0, 0, canvas.width, canvas.height); gameFrame++;

    let currentBiome = getCurrentBiome(); ctx.fillStyle = currentBiome.bg; ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = "#fff"; stars.forEach(s => { s.x -= s.speed; if (s.x < 0) s.x = canvas.width; ctx.fillRect(s.x, s.y, s.size, s.size); });
    ctx.fillStyle = currentBiome.bColor; buildings.forEach(b => { b.x -= b.speed; if (b.x + b.width < 0) b.x = canvas.width; ctx.fillRect(b.x, 368 - b.height, b.width, b.height); });
    ctx.strokeStyle = glitchActive ? "#bc13fe" : currentBiome.floor; ctx.lineWidth = 4; ctx.shadowBlur = 10; ctx.shadowColor = currentBiome.accent;
    ctx.beginPath(); ctx.moveTo(0, 368); ctx.lineTo(canvas.width, 368); ctx.stroke(); ctx.shadowBlur = 0;

    let activeChar = characters[equippedCharIndex];

    if (gameState === "MENU") {
        ctx.fillStyle = "#00f0ff"; ctx.font = "bold 32px 'Courier New'"; ctx.textAlign = "center"; ctx.shadowBlur = 15; ctx.shadowColor = "#00f0ff";
        ctx.fillText("CYBER GLITCH: 40 PROTOCOLS", canvas.width / 2, 50); ctx.shadowBlur = 0;
        ctx.fillStyle = "#ffff00"; ctx.font = "14px 'Courier New'"; ctx.fillText("CÜZDAN: ₿" + totalCoins + " | REKOR: " + highScore, canvas.width / 2, 85);

        let bLabels = ["SİBER BAŞLA", "PROTOKOL MAĞAZASI (40)", "GELİŞTİRME SEKTÖRÜ", "KONTRATLAR"];
        let bColors = ["#00f0ff", "#ff007f", "#39ff14", "#ffff00"]; let bTextColors = ["#000", "#fff", "#000", "#000"]; let idx = 0;
        for (let key in menuButtons) {
            ctx.fillStyle = bColors[idx]; ctx.fillRect(menuButtons[key].x, menuButtons[key].y, menuButtons[key].w, menuButtons[key].h);
            ctx.fillStyle = bTextColors[idx]; ctx.font = "bold 13px 'Courier New'"; ctx.fillText(bLabels[idx], menuButtons[key].x + 100, menuButtons[key].y + 22); idx++;
        }
        drawPixelMatrix(activeChar.r1, 70, 150, 60, 60, activeChar.color);
        ctx.fillStyle = activeChar.color; ctx.font = "11px 'Courier New'"; ctx.fillText("AKTİF KOŞUCU:", 100, 235); ctx.fillText(activeChar.name, 100, 250);
        ctx.fillStyle = "#ffff00"; ctx.fillText("SÜRÜCÜ: x" + activeChar.mult, 100, 265);
    } 
    else if (gameState === "CHAR_SELECT") {
        ctx.fillStyle = "#fff"; ctx.font = "bold 24px 'Courier New'"; ctx.textAlign = "center"; ctx.fillText("PROTOKOL MARKETİ (40 SÜRÜCÜ)", canvas.width / 2, 40);
        ctx.fillStyle = "#ffff00"; ctx.fillText("MEVCUT DATA BAKİYE: ₿" + totalCoins, canvas.width / 2, 70);
        ctx.fillStyle = "#444"; ctx.fillRect(charMenuButtons.back.x, charMenuButtons.back.y, charMenuButtons.back.w, charMenuButtons.back.h);
        ctx.fillStyle = "#fff"; ctx.font = "13px 'Courier New'"; ctx.fillText("<- PROTO", charMenuButtons.back.x + 45, charMenuButtons.back.y + 22);

        let c = characters[selectedCharIndex]; ctx.strokeStyle = c.color; ctx.strokeRect(250, 100, 300, 205);
        drawPixelMatrix(c.r1, 368, 110, 64, 64, c.color);
        ctx.fillStyle = c.color; ctx.font = "bold 15px 'Courier New'"; ctx.fillText(c.name, canvas.width / 2, 195);
        ctx.fillStyle = "#ffff00"; ctx.font = "bold 13px 'Courier New'"; ctx.fillText("DATA VERİMİ: x" + c.mult.toFixed(1), canvas.width / 2, 220);
        let status = c.unlocked ? (equippedCharIndex === selectedCharIndex ? "PROTOKOL ÇALIŞIYOR" : "SİSTEMDE KAYITLI") : "KİLİTLİ ALAN";
        ctx.fillStyle = "#fff"; ctx.font = "11px 'Courier New'"; ctx.fillText("INDEX [" + (selectedCharIndex+1) + "/40] | " + status, canvas.width / 2, 245);

        ctx.fillStyle = "#00f0ff"; ctx.fillRect(charMenuButtons.prev.x, charMenuButtons.prev.y, charMenuButtons.prev.w, charMenuButtons.prev.h);
        ctx.fillRect(charMenuButtons.next.x, charMenuButtons.next.y, charMenuButtons.next.w, charMenuButtons.next.h);
        ctx.fillStyle = "#000"; ctx.font = "bold 14px 'Courier New'"; ctx.fillText("<<", charMenuButtons.prev.x + 35, charMenuButtons.prev.y + 26); ctx.fillText(">>", charMenuButtons.next.x + 35, charMenuButtons.next.y + 26);
        ctx.fillStyle = c.unlocked ? (equippedCharIndex === selectedCharIndex ? "#555" : "#39ff14") : "#ff3131"; 
        ctx.fillRect(charMenuButtons.select.x, charMenuButtons.select.y, charMenuButtons.select.w, charMenuButtons.select.h);
        ctx.fillStyle = "#000"; ctx.fillText(c.unlocked ? (equippedCharIndex === selectedCharIndex ? "BAĞLI" : "SÜRÜCÜYÜ SEÇ") : "ENJEKTE ET: ₿" + c.price, charMenuButtons.select.x + 90, charMenuButtons.select.y + 27);
    } 
    else if (gameState === "UPGRADE_MENU") {
        ctx.fillStyle = "#fff"; ctx.font = "bold 24px 'Courier New'"; ctx.textAlign = "center"; ctx.fillText("SİBER GELİŞTİRME SEKTÖRÜ", canvas.width / 2, 50);
        ctx.fillStyle = "#ffff00"; ctx.fillText("PARAN: ₿" + totalCoins, canvas.width / 2, 85);

        let types = ["magnet", "shieldTime", "glitchLuck"]; let titles = ["COIN MIKNATISI", "KALKAN SÜRESİ", "GLITCH ŞANSI"]; let descs = ["Paraları Çeker", "Kalkanı Uzatır", "Glitch Puanı Artar"];
        for (let i = 0; i < 3; i++) {
            let type = types[i]; let lvl = upgrades[type]; let btn = i === 0 ? upgMenuButtons.b1 : (i === 1 ? upgMenuButtons.b2 : upgMenuButtons.b3);
            ctx.strokeStyle = "#39ff14"; ctx.strokeRect(btn.x - 10, btn.y - 40, btn.w + 20, 160);
            ctx.fillStyle = "#39ff14"; ctx.font = "bold 13px 'Courier New'"; ctx.fillText(titles[i], btn.x + btn.w/2, btn.y - 20);
            ctx.fillStyle = "#aaa"; ctx.font = "10px 'Courier New'"; ctx.fillText(descs[i], btn.x + btn.w/2, btn.y - 5);
            ctx.fillStyle = "#ffff00"; ctx.fillText("SEVİYE: " + lvl + "/3", btn.x + btn.w/2, btn.y + 12);
            ctx.fillStyle = lvl >= 3 ? "#555" : "#00f0ff"; ctx.fillRect(btn.x, btn.y + 25, btn.w, btn.h);
            ctx.fillStyle = "#000"; ctx.font = "bold 11px 'Courier New'"; ctx.fillText(lvl >= 3 ? "MAX ENJEKSİYON" : "YÜKLE: ₿" + upgradePrices[type][lvl], btn.x + btn.w/2, btn.y + 54);
        }
        ctx.fillStyle = "#ff007f"; ctx.fillRect(upgMenuButtons.back.x, upgMenuButtons.back.y, upgMenuButtons.back.w, upgMenuButtons.back.h);
        ctx.fillStyle = "#fff"; ctx.fillText("ÇIKIŞ", upgMenuButtons.back.x + 50, upgMenuButtons.back.y + 24);
    }
    else if (gameState === "QUEST_PANEL") {
        ctx.fillStyle = "rgba(0,0,0,0.85)"; ctx.fillRect(40, 40, 720, 320); ctx.strokeStyle = "#ffff00"; ctx.strokeRect(40, 40, 720, 320);
        ctx.fillStyle = "#ffff00"; ctx.font = "bold 22px 'Courier New'"; ctx.textAlign = "center"; ctx.fillText("SİBER ÖDÜL KONTRATLARIN", canvas.width / 2, 75);
        quests.forEach((q, idx) => {
            ctx.fillStyle = q.done ? "#39ff14" : "#aaa"; ctx.font = "12px 'Courier New'"; ctx.textAlign = "left";
            ctx.fillText((q.done ? "[TAMAMLANDI] " : "[AKTİF] ") + q.text, 80, 140 + idx * 55);
            ctx.fillStyle = "#ffff00"; ctx.textAlign = "right"; ctx.fillText("ÖDÜL: ₿" + q.reward, 720, 140 + idx * 55);
        });
        ctx.fillStyle = "#ff007f"; ctx.fillRect(upgMenuButtons.back.x, upgMenuButtons.back.y, upgMenuButtons.back.w, upgMenuButtons.back.h);
        ctx.fillStyle = "#fff"; ctx.textAlign = "center"; ctx.fillText("KAPAT", upgMenuButtons.back.x + 50, upgMenuButtons.back.y + 24);
    }
    else if (gameState === "PLAYING" || gameState === "BOSS_ALARM" || gameState === "BOSS_BATTLE") {
        let glitchTriggerChance = 0.0022 + (upgrades.glitchLuck * 0.0016);
        if (!glitchActive && gameState === "PLAYING" && Math.random() < glitchTriggerChance && gameFrame % 280 === 0) {
            glitchActive = true; glitchTimer = 240; screenShake = 14; glitchType = Math.random() > 0.5 ? "GOLD_RUSH" : "SPEED_OVERCLOCK"; playTone(820, "sawtooth", 0.25);
        }
        if (glitchActive) {
            glitchTimer--; ctx.fillStyle = "#bc13fe"; ctx.font = "bold 16px 'Courier New'"; ctx.textAlign = "center";
            ctx.fillText("!! GLITCH SIZINTISI: " + (glitchType === "GOLD_RUSH" ? "3 KAT VERİ HASILATI" : "OVERCLOCK HIZ MODU") + " !!", canvas.width/2, 90);
            if (glitchTimer <= 0) glitchActive = false;
        }

        if (score > 0 && score % 40 === 0 && gameState === "PLAYING") { gameState = "BOSS_ALARM"; bossTimer = 0; playTone(580, "sawtooth", 0.45); updateButtonVisibility(); }
        if (gameState === "BOSS_ALARM") {
            bossTimer++; if (gameFrame % 20 < 10) { ctx.fillStyle = "rgba(255,0,0,0.12)"; ctx.fillRect(0,0,canvas.width,canvas.height); }
            ctx.fillStyle = "#ff0000"; ctx.font = "bold 21px 'Courier New'"; ctx.textAlign = "center"; ctx.fillText("ANAMAL SİSTEM VİRÜSÜ ALGILANDI! BOSS SAVAŞI!", canvas.width / 2, 180);
            if (bossTimer > 120) { gameState = "BOSS_BATTLE"; obstacles = []; updateButtonVisibility(); }
        }

        dino.velocityY += dino.gravity; dino.y += dino.velocityY; let currentH = dino.height;
        if (dino.isDucking && dino.isGrounded) { dino.y = dino.baseY + 16; currentH = dino.height - 16; } 
        else { if (dino.y >= dino.baseY) { dino.y = dino.baseY; dino.velocityY = 0; dino.isGrounded = true; } }

        if (dino.isGrounded && gameFrame % 6 === 0) createParticles(dino.x, 365, activeChar.color);

        ctx.shadowBlur = 15; ctx.shadowColor = activeChar.color;
        if (dino.isDucking && dino.isGrounded) { 
            drawPixelMatrix(activeChar.duck, dino.x, dino.y, dino.width, currentH, activeChar.color); 
        } 
        else {
            let frame = activeChar.r1; 
            drawPixelMatrix(frame, dino.x, dino.y, dino.width, dino.height, activeChar.color);
        }
        if (activePowerUp === "SHIELD") { ctx.strokeStyle = "#39ff14"; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(dino.x + 24, dino.y + 24, 32, 0, Math.PI*2); ctx.stroke(); }
        ctx.shadowBlur = 0;

        if (activePowerUp) {
            powerUpTimer--; ctx.fillStyle = "#fff"; ctx.font = "12px 'Courier New'"; ctx.textAlign = "left";
            ctx.fillText(activePowerUp + ": " + Math.ceil(powerUpTimer/60) + "s", 20, 80); if (powerUpTimer <= 0) activePowerUp = null;
        }

        let speedMultiplier = (glitchActive && glitchType === "SPEED_OVERCLOCK") ? 1.5 : 1.0;
        if (gameFrame % 380 === 0 && gameState === "PLAYING") spawnPowerUp();
        for (let i = powerUpsInWorld.length - 1; i >= 0; i--) {
            let p = powerUpsInWorld[i]; p.x -= (gameSpeed * speedMultiplier);
            if (upgrades.magnet > 0) {
                let dist = p.x - dino.x; if (dist > 0 && dist < (100 + upgrades.magnet * 55)) { p.y += (dino.y + 10 - p.y) * 0.16; p.x -= 3; }
            }
            ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, p.width, p.height);
            if (dino.x < p.x + p.width && dino.x + dino.width > p.x && dino.y < p.y + p.height && dino.y + currentH > p.y) {
                activePowerUp = p.type; powerUpTimer = 320 + (upgrades.shieldTime * 100); powerUpsInWorld.splice(i, 1); playTone(820, "sine", 0.15);
            }
            if (p.x + p.width < 0) powerUpsInWorld.splice(i, 1);
        }

        if (gameState === "PLAYING") {
            spawnTimer++; if (spawnTimer > Math.max(45, 92 - gameSpeed * 2)) { spawnObstacle(); spawnTimer = 0; }
            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i]; obs.x -= (gameSpeed * speedMultiplier); ctx.fillStyle = obs.color;
                if (obs.type === "cactus") { ctx.fillRect(obs.x, obs.y, obs.width, obs.height); } 
                else { drawPixelMatrix(Math.floor(gameFrame / 8) % 2 === 0 ? BIRD1 : BIRD2, obs.x, obs.y, obs.width, obs.height, obs.color); }
                if (obs.type === "bird" && dino.isDucking && obs.x < dino.x + dino.width && obs.x > dino.x - 20 && !obs.duckCounted) { quests[1].current++; obs.duckCounted = true; }
                if (dino.x + 4 < obs.x + obs.width && dino.x + dino.width - 4 > obs.x && dino.y + 4 < obs.y + obs.height && dino.y + currentH - 4 > obs.y) {
                    if (activePowerUp === "SHIELD") { activePowerUp = null; obstacles.splice(i, 1); playTone(320, "sine", 0.1); screenShake = 6; } else { endTheGame(); return; }
                }
                if (obs.x + obs.width < 0) { obstacles.splice(i, 1); score++; if (score % 3 === 0) gameSpeed += 0.30; }
            }
        }

        if (gameState === "BOSS_BATTLE") {
            if (boss.x > 640) boss.x -= 2.5; boss.y += boss.speedY; if (boss.y < 40 || boss.y > 210) boss.speedY *= -1;
            drawPixelMatrix(M_BOSS, boss.x, boss.y, boss.width, boss.height, "#ff0000");
            ctx.fillStyle = "#333"; ctx.fillRect(boss.x, boss.y - 15, boss.width, 7);
            ctx.fillStyle = "#ff0000"; ctx.fillRect(boss.x, boss.y - 15, (boss.health/boss.maxHealth)*boss.width, 7);
            if (gameFrame % 52 === 0) { bossProjectiles.push({ x: boss.x, y: boss.y + 45, width: 22, height: 6, speed: 6.5, color: "#ffaa00" }); playTone(410, "sawtooth", 0.05); }
            for (let i = bossProjectiles.length - 1; i >= 0; i--) {
                let m = bossProjectiles[i]; m.x -= m.speed; ctx.fillStyle = m.color; ctx.fillRect(m.x, m.y, m.width, m.height);
                if (dino.x < m.x + m.width && dino.x + dino.width > m.x && dino.y < m.y + m.height && dino.y + currentH > m.y) {
                    if (activePowerUp === "SHIELD") { activePowerUp = null; bossProjectiles.splice(i, 1); } else { endTheGame(); return; }
                }
                if (m.x + m.width < 0) { bossProjectiles.splice(i, 1); boss.health -= 15; score += 2; }
            }
            if (boss.health <= 0) { gameState = "PLAYING"; score += 15; resetGameObjectsAfterBoss(); playTone(920, "square", 0.4); screenShake = 14; updateButtonVisibility(); }
        }

        for (let i = particles.length - 1; i >= 0; i--) {
            let p = particles[i]; p.x += p.vx; p.y += p.vy; p.life--; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, p.size, p.size); if (p.life <= 0) particles.splice(i, 1);
        }

        ctx.fillStyle = "#fff"; ctx.font = "bold 13px 'Courier New'"; ctx.textAlign = "left";
        ctx.fillText("SCORE: " + score + " (SÜRÜCÜ: x" + activeChar.mult.toFixed(1) + ")", 20, 35);
        ctx.fillStyle = currentBiome.accent; ctx.fillText("BİYOM [" + (Math.min(10, Math.floor(score/15)+1)) + "/10]: " + currentBiome.name, 20, 55);
        ctx.textAlign = "right"; ctx.fillStyle = "#fff"; ctx.fillText("REKOR: " + highScore, canvas.width - 20, 35);
    } 
    else if (gameState === "GAMEOVER") {
        ctx.fillStyle = "rgba(2, 2, 6, 0.96)"; ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "#ff3131"; ctx.font = "bold 34px 'Courier New'"; ctx.textAlign = "center"; ctx.shadowBlur = 20; ctx.shadowColor = "#ff3131";
        ctx.fillText("BAĞLANTI KESİLDİ", canvas.width / 2, canvas.height / 2 - 35); ctx.shadowBlur = 0;
        ctx.fillStyle = "#ffff00"; ctx.font = "15px 'Courier New'"; ctx.fillText("Kazanılan Toplam Siber Birim: +₿" + lastEarnedCoins, canvas.width / 2, canvas.height / 2 + 10);
        ctx.fillStyle = "#00f0ff"; ctx.font = "12px 'Courier New'"; ctx.fillText("Yeniden Bellek Yüklemek İçin Ekrana Bas", canvas.width / 2, canvas.height / 2 + 60);
    }
    ctx.restore();
}

function resetGameObjectsAfterBoss() { boss.x = 850; boss.health = boss.maxHealth; bossProjectiles = []; spawnTimer = 0; }

gameLoop();
</script>

</body>
</html>
