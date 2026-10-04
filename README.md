<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Red Alert Mini - Edition Finale</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
        }
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #111;
            font-family: 'Courier New', Courier, monospace;
            width: 100%;
            height: 100%;
        }
        #game-container {
            position: relative;
            width: 100vw;
            height: 100vh;
        }
        canvas {
            display: block;
            background-color: #2e3d24;
        }
        #ui-panel {
            position: absolute;
            top: 10px;
            right: 10px;
            width: 240px;
            background: rgba(0, 0, 0, 0.85);
            border: 2px solid #00ff00;
            color: #00ff00;
            padding: 15px;
            border-radius: 5px;
            z-index: 10;
        }
        .ui-stat {
            font-size: 16px;
            margin-bottom: 10px;
            font-weight: bold;
        }
        .build-btn {
            width: 100%;
            background: #222;
            border: 1px solid #00ff00;
            color: #00ff00;
            padding: 8px;
            margin: 5px 0;
            cursor: pointer;
            font-weight: bold;
            font-family: inherit;
            transition: 0.2s;
        }
        .build-btn:hover:not(:disabled) {
            background: #00ff00;
            color: #000;
        }
        .build-btn:disabled {
            border-color: #555;
            color: #555;
            cursor: not-allowed;
        }
        #mobile-controls {
            position: absolute;
            bottom: 20px;
            left: 20px;
            right: 20px;
            display: none;
            justify-content: space-between;
            align-items: flex-end;
            z-index: 10;
            pointer-events: none;
        }
        .touch-btn {
            width: 70px;
            height: 70px;
            background: rgba(0, 0, 0, 0.7);
            border: 3px solid #00ff00;
            color: #00ff00;
            border-radius: 50%;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 11px;
            pointer-events: auto;
            box-shadow: 0 0 10px rgba(0,255,0,0.5);
            text-align: center;
        }
        .touch-btn:active {
            background: #00ff00;
            color: #000;
        }
        #mobile-actions {
            display: flex;
            gap: 10px;
        }
        #game-overlay {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8);
            color: #fff;
            display: none;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 100;
        }
        #game-overlay h1 {
            font-size: 44px;
            margin-bottom: 20px;
            letter-spacing: 5px;
        }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="gameCanvas"></canvas>

    <div id="ui-panel">
        <div class="ui-stat">CREDITS: $<span id="credits-display">1200</span></div>
        <div style="font-size: 11px; margin-bottom: 5px; color: #888;">PRODUCTION :</div>
        <button class="build-btn" id="btn-tank" onclick="buyUnit('tank')">CHAR LOURD ($400)</button>
        <button class="build-btn" id="btn-harvester" onclick="buyUnit('harvester')">COLLECTEUR ($500)</button>
        <button class="build-btn" id="btn-turret" onclick="activateTurretPlacement()">TOURELLE DEF ($600)</button>
        <div id="placement-mode" style="display:none; font-size:11px; color:#ffaa00; margin-top:5px; text-align:center;">
            ▲ Cliquez sur la map pour poser la tourelle
        </div>
        <div style="margin-top: 15px; font-size: 11px; color: #00aa00; line-height: 1.3;">
            <b style="color:#fff">PC:</b> Clic G. Sélection | Clic D. Action.<br>
            <b style="color:#fff">Mobile:</b> Tap Sélection | Boutons d'action.
        </div>
    </div>

    <div id="mobile-controls">
        <div class="touch-btn" id="btn-move">ORDRE<br>MOVE</div>
        <div id="mobile-actions">
            <div class="touch-btn" onclick="buyUnit('tank')">+TANK</div>
            <div class="touch-btn" onclick="buyUnit('harvester')">+HARV</div>
            <div class="touch-btn" onclick="activateTurretPlacement()">+DEF</div>
        </div>
    </div>

    <div id="game-overlay">
        <h1 id="overlay-title">VICTOIRE</h1>
        <button class="build-btn" style="width: auto; padding: 10px 30px;" onclick="window.location.reload()">REJOUER</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    function resizeCanvas() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resizeCanvas);
    resizeCanvas();

    const isTouchDevice = ('ontouchstart' in window) || (navigator.maxTouchPoints > 0);
    if (isTouchDevice) {
        document.getElementById('mobile-controls').style.display = 'flex';
    }

    let credits = 1200;
    let selectedUnit = null;
    let gameOver = false;
    let placementMode = false;

    const mapWidth = 2200;
    const mapHeight = 2200;
    const camera = { x: 0, y: 0 };

    const units = [];
    const buildings = [];
    const fields = [];
    const lasers = [];

    const fogGridSize = 50; 
    const fogCols = Math.ceil(mapWidth / fogGridSize);
    const fogRows = Math.ceil(mapHeight / fogGridSize);
    const fogMap = Array(fogCols).fill(0).map(() => Array(fogRows).fill(0));

    // Génération des mines d'or
    function spawnOreFields() {
        for(let f=0; f<8; f++) {
            let cx = 500 + Math.random() * (mapWidth - 1000);
            let cy = 500 + Math.random() * (mapHeight - 1000);
            for(let i=0; i<12; i++) {
                fields.push({
                    x: cx + (Math.random() - 0.5) * 160,
                    y: cy + (Math.random() - 0.5) * 160,
                    amount: 600,
                    radius: 8 + Math.random() * 6
                });
            }
        }
    }
    spawnOreFields();

    // Structures de départ
    const playerHQ = { type: 'hq', x: 200, y: 200, radius: 55, team: 'player', hp: 2500, maxHp: 2500 };
    const enemyHQ = { type: 'hq', x: mapWidth - 200, y: mapHeight - 200, radius: 55, team: 'enemy', hp: 2500, maxHp: 2500 };
    buildings.push(playerHQ, enemyHQ);

    // Tourelle ennemie défensive de départ
    buildings.push({ type: 'turret', x: mapWidth - 400, y: mapHeight - 350, radius: 25, team: 'enemy', hp: 600, maxHp: 600, range: 220, cooldown: 0 });

    camera.x = playerHQ.x - canvas.width / 2;
    camera.y = playerHQ.y - canvas.height / 2;

    function createUnit(type, x, y, team) {
        units.push({
            id: Math.random(),
            type: type,
            x: x, y: y,
            targetX: x, targetY: y,
            team: team,
            speed: type === 'tank' ? 2.6 : 1.9,
            radius: type === 'tank' ? 16 : 18,
            hp: type === 'tank' ? 320 : 450,
            maxHp: type === 'tank' ? 320 : 450,
            range: 190,
            cooldown: 0,
            targetEnemy: null,
            cargo: 0,
            maxCargo: 400,
            harvesting: false,
            targetOre: null
        });
    }

    createUnit('tank', 320, 200, 'player');
    createUnit('harvester', 200, 320, 'player');
    createUnit('tank', mapWidth - 350, mapHeight - 200, 'enemy');

    function buyUnit(type) {
        let cost = type === 'tank' ? 400 : 500;
        if (credits >= cost) {
            credits -= cost;
            createUnit(type, playerHQ.x + (Math.random() - 0.5) * 80, playerHQ.y + 100, 'player');
        }
    }

    function activateTurretPlacement() {
        if (credits >= 600) {
            placementMode = true;
            document.getElementById('placement-mode').style.display = 'block';
        }
    }

    // Revenu passif minimal
    setInterval(() => { if(!gameOver) credits += 5; }, 1000);

    // Production et vagues d'attaques de l'IA adverse
    setInterval(() => {
        if(!gameOver) {
            createUnit('tank', enemyHQ.x - 100, enemyHQ.y - (Math.random() * 80), 'enemy');
        }
    }, 10000);

    function updateGame() {
        if (gameOver) return;

        document.getElementById('credits-display').innerText = Math.floor(credits);
        document.getElementById('btn-tank').disabled = credits < 400;
        document.getElementById('btn-harvester').disabled = credits < 500;
        document.getElementById('btn-turret').disabled = credits < 600 || placementMode;

        // Mise à jour des Lasers à l'écran
        for (let i = lasers.length - 1; i >= 0; i--) {
            lasers[i].duration--;
            if (lasers[i].duration <= 0) lasers.splice(i, 1);
        }

        // --- GESTION DES BÂTIMENTS & TOURELLES ---
        buildings.forEach(b => {
            if (b.type === 'turret') {
                if (b.cooldown > 0) b.cooldown--;
                
                // Recherche automatique d'une cible par la tourelle
                let targets = units.filter(u => u.team !== b.team);
                for (let t of targets) {
                    if (Math.hypot(t.x - b.x, t.y - b.y) <= b.range) {
                        if (b.cooldown === 0) {
                            t.hp -= 35;
                            b.cooldown = 40;
