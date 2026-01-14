
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Crypto-Mon: Yellow Version</title>
    <style>
        /* --- CSS STYLES (The Visuals) --- */
        :root {
            --gb-darkest: #0f380f;
            --gb-dark: #306230;
            --gb-light: #8bac0f;
            --gb-lightest: #9bbc0f;
        }

        body {
            background-color: #202020;
            color: var(--gb-darkest);
            font-family: 'Courier New', Courier, monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            overflow: hidden;
            user-select: none; /* Prevents highlighting text while playing */
        }

        /* The GameBoy Case */
        #gameboy {
            background-color: #c0c0c0;
            padding: 20px 20px 40px 20px;
            border-radius: 10px 10px 40px 10px;
            box-shadow: 5px 5px 15px rgba(0,0,0,0.5);
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .screen-border {
            background-color: #555;
            padding: 15px 15px 30px 15px;
            border-radius: 10px 10px 30px 10px;
            position: relative;
        }

        .power-light {
            width: 8px;
            height: 8px;
            background-color: red;
            border-radius: 50%;
            position: absolute;
            top: 60px;
            left: 5px;
            box-shadow: 0 0 5px red;
            animation: pulse 2s infinite;
        }

        @keyframes pulse {
            0% { opacity: 0.7; }
            50% { opacity: 1; }
            100% { opacity: 0.7; }
        }

        /* The Game Screen */
        #game-container {
            position: relative;
            width: 320px;
            height: 320px;
            background-color: var(--gb-lightest);
            border: 2px solid var(--gb-darkest);
        }

        canvas {
            display: block;
            image-rendering: pixelated;
        }

        /* UI Overlays (Text Box & Battle Menu) */
        #ui-layer {
            position: absolute;
            bottom: 0;
            left: 0;
            width: 100%;
            height: 100px;
            background-color: white;
            border-top: 4px solid var(--gb-darkest);
            box-sizing: border-box;
            padding: 8px;
            display: none; /* Hidden by default */
        }

        .ui-visible {
            display: block !important;
        }

        .battle-text {
            font-size: 14px;
            font-weight: bold;
            margin-bottom: 5px;
            line-height: 1.2;
        }

        .battle-options {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 5px;
        }

        button {
            background: var(--gb-lightest);
            border: 2px solid var(--gb-darkest);
            font-family: inherit;
            font-weight: bold;
            cursor: pointer;
            text-transform: uppercase;
            font-size: 12px;
            padding: 5px;
        }

        button:hover {
            background: var(--gb-dark);
            color: var(--gb-lightest);
        }

        button:active {
            background: var(--gb-darkest);
            color: var(--gb-lightest);
        }

        .controls {
            margin-top: 15px;
            color: #888;
            font-size: 12px;
            text-align: center;
        }
        
        #save-indicator {
            position: absolute;
            top: 5px;
            right: 5px;
            font-size: 10px;
            color: var(--gb-darkest);
            display: none;
        }
    </style>
</head>
<body>

    <div id="gameboy">
        <div class="screen-border">
            <div class="power-light"></div>
            <div id="game-container">
                <canvas id="gameCanvas" width="320" height="320"></canvas>
                <div id="save-indicator">SAVING...</div>
                
                <div id="ui-layer">
                    <div id="text-display" class="battle-text"></div>
                    
                    <div id="battle-menu" class="battle-options" style="display:none;">
                        <button onclick="game.playerAction('attack')">Panic Sell</button>
                        <button onclick="game.playerAction('heal')">Buy Dip</button>
                        <button onclick="game.playerAction('special')">Rug Pull</button>
                        <button onclick="game.runAway()">Disconnect</button>
                    </div>
                    
                    <div id="continue-prompt" style="display:none; text-align:right; font-size:10px; margin-top:5px;">
                        [Press Space to Continue]
                    </div>
                </div>
            </div>
        </div>
        <div class="controls">
            ARROWS to Move &nbsp;|&nbsp; SPACE to Interact
        </div>
    </div>

    <script>
        /* --- JAVASCRIPT GAME ENGINE --- */

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const uiLayer = document.getElementById('ui-layer');
        const textDisplay = document.getElementById('text-display');
        const battleMenu = document.getElementById('battle-menu');
        const continuePrompt = document.getElementById('continue-prompt');
        const saveInd = document.getElementById('save-indicator');

        // --- 1. ASSETS & CONFIG ---
        const TILE = 32; // Size of one grid square
        const COLS = 10;
        const ROWS = 10;

        // "Sprites" represented by colors
        const COLORS = {
            path: '#9bbc0f',     // Lightest (Walkable)
            wall: '#0f380f',     // Darkest (Block)
            grass: '#306230',    // Dark (Encounter)
            player: '#0f380f',
            uiBg: '#ffffff'
        };

        // The Monsters
        const CRYPTO_DEX = [
            { name: "Bit-Bull", hp: 50, maxHp: 50, dmg: 10, symbol: "₿" },
            { name: "Doge-Pup", hp: 30, maxHp: 30, dmg: 5, symbol: "Ð" },
            { name: "Eth-Gho", hp: 40, maxHp: 40, dmg: 8, symbol: "Ξ" },
            { name: "Pepe-Frog", hp: 25, maxHp: 25, dmg: 12, symbol: "🐸" }
        ];

        // 0=Path, 1=Wall, 2=Volatile Market (Grass)
        const WORLD_MAP = [
            [1,1,1,1,1,1,1,1,1,1],
            [1,0,0,1,0,0,0,0,0,1],
            [1,0,0,1,0,2,2,2,0,1],
            [1,0,0,0,0,2,2,2,0,1],
            [1,0,1,1,0,0,0,0,0,1],
            [1,0,2,2,2,2,0,1,0,1],
            [1,0,2,2,2,2,0,1,0,1],
            [1,0,0,0,0,0,0,0,0,1],
            [1,1,1,1,1,0,0,1,1,1],
            [1,1,1,1,1,1,1,1,1,1]
        ];

        // --- 2. GAME STATE MANAGEMENT ---
        const game = {
            state: 'ROAMING', // ROAMING, BATTLE, DIALOGUE
            player: { x: 1, y: 1, hp: 100, maxHp: 100, name: "Trader", xp: 0 },
            activeEnemy: null,
            
            init: function() {
                // Attempt to load save data
                this.load();
                this.draw();
                
                if(!localStorage.getItem('cryptoMonSave')) {
                    this.showDialogue("Welcome to Blockchain City! Watch out for volatile markets (Dark Grass).");
                } else {
                    this.showDialogue("Welcome back, Trader. Wallet synced.");
                }
            },

            // --- SAVE / LOAD SYSTEM ---
            save: function() {
                const data = {
                    player: this.player
                };
                localStorage.setItem('cryptoMonSave', JSON.stringify(data));
                
                // Visual feedback
                saveInd.style.display = 'block';
                setTimeout(() => { saveInd.style.display = 'none'; }, 500);
            },

            load: function() {
                const savedData = localStorage.getItem('cryptoMonSave');
                if (savedData) {
                    const parsed = JSON.parse(savedData);
                    this.player = parsed.player;
                }
            },

            // --- MOVEMENT ---
            move: function(dx, dy) {
                if (this.state !== 'ROAMING') return;

                let newX = this.player.x + dx;
                let newY = this.player.y + dy;

                // Collision Check
                if (WORLD_MAP[newY][newX] === 1) return;

                this.player.x = newX;
                this.player.y = newY;
                
                // Auto-Save on every step
                this.save();
                this.draw();

                // Encounter Check (Grass)
                if (WORLD_MAP[newY][newX] === 2) {
                    if (Math.random() < 0.2) { // 20% Chance
                        this.startBattle();
                    }
                }
            },

            // --- BATTLE SYSTEM ---
            startBattle: function() {
                this.state = 'BATTLE';
                // Pick random crypto
                const template = CRYPTO_DEX[Math.floor(Math.random() * CRYPTO_DEX.length)];
                this.activeEnemy = { ...template }; // Clone
                
                this.draw(); 
                this.showBattleUI(`Wild ${this.activeEnemy.name} appeared!`);
            },

            playerAction: function(action) {
                if (this.state !== 'BATTLE') return;

                let playerDmg = 0;
                let msg = "";

                // Player Turn
                if (action === 'attack') {
                    playerDmg = Math.floor(Math.random() * 15) + 5;
                    msg = `Used Panic Sell! Dealt ${playerDmg} damage.`;
                    this.activeEnemy.hp -= playerDmg;
                } else if (action === 'heal') {
                    const heal = 20;
                    this.player.hp = Math.min(this.player.hp + heal, this.player.maxHp);
                    msg = `Bought the Dip! Recovered health.`;
                } else if (action === 'special') {
                    playerDmg = Math.floor(Math.random() * 25);
                    msg = `Rug Pull initiated! Dealt ${playerDmg} damage.`;
                    this.activeEnemy.hp -= playerDmg;
                }

                this.updateBattleLog(msg);
                this.draw();

                // Check Win
                if (this.activeEnemy.hp <= 0) {
                    setTimeout(() => {
                        this.showDialogue(`The ${this.activeEnemy.name} crashed! You won!`);
                        this.state = 'DIALOGUE';
                        this.save(); // Save progress after win
                    }, 1000);
                    return;
                }

                // Enemy Turn (After delay)
                setTimeout(() => this.enemyTurn(), 1500);
            },

            enemyTurn: function() {
                if (this.state !== 'BATTLE') return;
                
                const enemyDmg = this.activeEnemy.dmg + Math.floor(Math.random() * 5);
                this.player.hp -= enemyDmg;
                this.updateBattleLog(`${this.activeEnemy.name} attacks! lost ${enemyDmg} HP.`);
                this.draw();

                if (this.player.hp <= 0) {
                    setTimeout(() => {
                        this.showDialogue("Your Portfolio was liquidated... (Game Over)");
                        this.player.hp = 100; // Reset
                        this.player.x = 1; this.player.y = 1;
                        this.save(); // Save the reset state
                    }, 1500);
                }
            },

            runAway: function() {
                this.showDialogue("Disconnected from server safely.");
            },

            // --- UI HELPERS ---
            showDialogue: function(text) {
                this.state = 'DIALOGUE';
                uiLayer.classList.add('ui-visible');
                battleMenu.style.display = 'none';
                continuePrompt.style.display = 'block';
                textDisplay.innerText = text;
            },

            showBattleUI: function(text) {
                uiLayer.classList.add('ui-visible');
                battleMenu.style.display = 'grid';
                continuePrompt.style.display = 'none';
                textDisplay.innerText = text;
            },

            updateBattleLog: function(text) {
                textDisplay.innerText = text;
            },

            dismissDialogue: function() {
                uiLayer.classList.remove('ui-visible');
                this.state = 'ROAMING';
                this.draw();
            },

            // --- RENDERING ---
            draw: function() {
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                if (this.state === 'ROAMING' || this.state === 'DIALOGUE') {
                    // Draw Map
                    for (let y = 0; y < ROWS; y++) {
                        for (let x = 0; x < COLS; x++) {
                            let tile = WORLD_MAP[y][x];
                            if (tile === 1) ctx.fillStyle = COLORS.wall;
                            else if (tile === 2) ctx.fillStyle = COLORS.grass;
                            else ctx.fillStyle = COLORS.path;
                            
                            ctx.fillRect(x*TILE, y*TILE, TILE, TILE);
                            // Grid Lines
                            ctx.strokeStyle = "rgba(0,0,0,0.1)";
                            ctx.strokeRect(x*TILE, y*TILE, TILE, TILE);
                        }
                    }
                    // Draw Player
                    ctx.font = "25px Arial";
                    ctx.fillText("🧑‍💻", this.player.x*TILE + 2, this.player.y*TILE + 26);
                
                } else if (this.state === 'BATTLE') {
                    // Battle Background
                    ctx.fillStyle = "#f0f0f0";
                    ctx.fillRect(0,0, canvas.width, canvas.height);

                    // Enemy HUD
                    ctx.fillStyle = "black";
                    ctx.font = "16px Courier New";
                    ctx.fillText(this.activeEnemy.name, 20, 40);
                    ctx.fillText(`HP: ${this.activeEnemy.hp}`, 20, 60);
                    // Enemy Sprite
                    ctx.font = "60px Arial";
                    ctx.fillText(this.activeEnemy.symbol, 200, 140);

                    // Player HUD
                    ctx.fillStyle = "black";
                    ctx.font = "16px Courier New";
                    ctx.fillText("My Portfolio", 180, 200);
                    ctx.fillText(`HP: ${this.player.hp}`, 180, 220);
                    // Player Sprite
                    ctx.font = "60px Arial";
                    ctx.fillText("🧑‍💻", 50, 220);
                }
            }
        };

        // --- 3. INPUT LISTENER ---
        window.addEventListener('keydown', (e) => {
            // Prevent scrolling with arrows
            if(["ArrowUp","ArrowDown","ArrowLeft","ArrowRight"].indexOf(e.code) > -1) {
                e.preventDefault();
            }

            if (game.state === 'DIALOGUE' && e.code === 'Space') {
                game.dismissDialogue();
                return;
            }

            if (game.state === 'ROAMING') {
                if (e.key === 'ArrowUp') game.move(0, -1);
                if (e.key === 'ArrowDown') game.move(0, 1);
                if (e.key === 'ArrowLeft') game.move(-1, 0);
                if (e.key === 'ArrowRight') game.move(1, 0);
            }
        });

        // Start Game
        game.init();

    </script>
</body>
</html>
