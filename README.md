<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rhythm Engine: SV & Quaver Support</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
    <style>
        body { margin: 0; background: #050505; color: #fff; font-family: 'Segoe UI', sans-serif; overflow: hidden; }
        
        #game-container {
            display: flex; justify-content: center; align-items: center; height: 100vh;
            position: relative;
        }

        canvas { background: #111; border-left: 2px solid #333; border-right: 2px solid #333; }

        /* UI Overlays */
        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.9); display: flex; flex-direction: column;
            justify-content: center; align-items: center; z-index: 10;
            transition: opacity 0.3s;
        }
        .hidden { opacity: 0; pointer-events: none; }

        .drop-zone {
            border: 2px dashed #00d2ff; padding: 40px; border-radius: 10px;
            text-align: center; cursor: pointer; transition: 0.2s;
            background: rgba(0, 210, 255, 0.05);
        }
        .drop-zone:hover { background: rgba(0, 210, 255, 0.15); }

        h1 { margin: 0 0 10px; color: #00d2ff; text-shadow: 0 0 15px #00d2ff; }
        p { color: #888; font-size: 0.9em; margin-bottom: 20px; }

        button {
            margin-top: 20px; padding: 12px 30px; background: #00d2ff; border: none;
            color: #000; font-weight: bold; font-size: 1.1em; cursor: pointer;
            border-radius: 4px; box-shadow: 0 0 15px rgba(0, 210, 255, 0.5);
        }
        button:disabled { background: #444; color: #888; box-shadow: none; cursor: not-allowed; }

        /* HUD */
        #hud-top { position: absolute; top: 20px; right: 20px; text-align: right; font-family: monospace; font-size: 24px; pointer-events: none; }
        #hud-combo { position: absolute; bottom: 20px; left: 20px; font-size: 40px; color: #00d2ff; font-weight: bold; pointer-events: none; }
        #feedback { 
            position: absolute; top: 60%; left: 50%; transform: translate(-50%, -50%); 
            font-size: 32px; font-weight: 800; pointer-events: none; text-shadow: 0 2px 5px rgba(0,0,0,0.8);
        }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="gameCanvas" width="500" height="800"></canvas>

    <div id="hud-top">
        <div id="score">000000</div>
        <div id="acc" style="font-size:16px; color:#aaa">100.00%</div>
    </div>
    <div id="hud-combo">0x</div>
    <div id="feedback"></div>

    <div id="menu" class="overlay">
        <h1>Flux Engine</h1>
        <p>Supports .osz (Osu) and .qp (Quaver)</p>
        
        <label class="drop-zone">
            <span>Click to import Map Archive</span>
            <input type="file" id="fileInput" accept=".osz,.qp,.zip" style="display:none">
        </label>

        <div id="loading-text" style="margin-top:15px; height:20px; color:#aaa"></div>
        <button id="btn-start" onclick="engine.start()" disabled>PLAY</button>
        <div style="margin-top:20px; font-size:0.8em; color:#555">Controls: D F J K</div>
    </div>

    <div id="results" class="overlay hidden">
        <h1>Map Finished</h1>
        <h2 id="final-score">Score: 0</h2>
        <button onclick="location.reload()">Load New Map</button>
    </div>
</div>

<script>
/**
 * AUDIO CONTEXT & CORE CONFIG
 */
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
const CONFIG = {
    keys: ['d', 'f', 'j', 'k'],
    colors: ['#00d2ff', '#ff00aa', '#ff00aa', '#00d2ff'],
    laneWidth: 100, // 4 lanes = 400px, plus padding
    hitY: 700,
    baseScrollSpeed: 1.0, // Pixels per ms (Base)
    hitWindows: { 300: 45, 100: 90, 50: 135, miss: 180 }
};

/**
 * SV INTEGRATOR
 * This calculates the "Visual Position" of notes based on Scroll Velocity changes.
 * Instead of y = time * speed, y = integral(speed(t) dt).
 */
class VisualIntegrator {
    constructor() {
        this.checkpoints = []; // Stores { time, visualPos, speed }
    }

    // Process raw SV points: [{time, multiplier}]
    build(svPoints, maxTime) {
        // Sort points by time
        svPoints.sort((a, b) => a.time - b.time);
        
        // Ensure a starting point at time 0
        if (svPoints.length === 0 || svPoints[0].time > 0) {
            svPoints.unshift({ time: 0, multiplier: 1.0 });
        }

        this.checkpoints = [];
        let currentVisualPos = 0;
        
        for (let i = 0; i < svPoints.length; i++) {
            const curr = svPoints[i];
            const next = svPoints[i+1];
            
            // Push current state
            this.checkpoints.push({
                time: curr.time,
                visualPos: currentVisualPos,
                speed: curr.multiplier * CONFIG.baseScrollSpeed
            });

            // Calculate distance to next point
            if (next) {
                const duration = next.time - curr.time;
                currentVisualPos += duration * (curr.multiplier * CONFIG.baseScrollSpeed);
            }
        }
    }

    // Get the visual Y coordinate (absolute pixels) for a specific time
    getVisualPos(time) {
        // Find the active checkpoint (last one before 'time')
        // Binary search is better, but linear is fine for <1000 SV lines
        let cp = this.checkpoints[0];
        for (let i = 1; i < this.checkpoints.length; i++) {
            if (this.checkpoints[i].time > time) break;
            cp = this.checkpoints[i];
        }
        
        const timeOffset = time - cp.time;
        return cp.visualPos + (timeOffset * cp.speed);
    }
}

/**
 * GAME ENGINE
 */
const engine = {
    state: {
        playing: false,
        startTime: 0,
        mapData: null,
        audioBuffer: null,
        audioSource: null,
        score: 0,
        combo: 0,
        maxCombo: 0,
        stats: {300:0, 100:0, 50:0, miss:0},
        heldKeys: [false,false,false,false],
        activeNotes: [],
        integrator: new VisualIntegrator()
    },

    init: () => {
        // Setup Inputs
        window.addEventListener('keydown', (e) => {
            if (e.repeat || !engine.state.playing) return;
            const idx = CONFIG.keys.indexOf(e.key.toLowerCase());
            if (idx !== -1) {
                engine.state.heldKeys[idx] = true;
                engine.processHit(idx);
            }
        });
        window.addEventListener('keyup', (e) => {
            const idx = CONFIG.keys.indexOf(e.key.toLowerCase());
            if (idx !== -1) engine.state.heldKeys[idx] = false;
        });
    },

    processHit: (col) => {
        const now = (audioCtx.currentTime - engine.state.startTime) * 1000;
        
        // Find closest hittable note in column
        const notes = engine.state.activeNotes.filter(n => n.col === col && !n.hit);
        if(!notes.length) return;

        // Get closest note by time
        const note = notes.reduce((prev, curr) => 
            Math.abs(curr.time - now) < Math.abs(prev.time - now) ? curr : prev
        );

        const diff = Math.abs(note.time - now);

        if (diff <= CONFIG.hitWindows[50]) {
            note.hit = true;
            let val = 0;
            if (diff <= CONFIG.hitWindows[300]) { val = 300; engine.feedback("PERFECT", "#00d2ff"); }
            else if (diff <= CONFIG.hitWindows[100]) { val = 100; engine.feedback("GOOD", "#00ff00"); }
            else { val = 50; engine.feedback("BAD", "#ffff00"); }
            
            engine.state.stats[val]++;
            engine.state.combo++;
            engine.state.score += val + (engine.state.combo * 2);
            if(engine.state.combo > engine.state.maxCombo) engine.state.maxCombo = engine.state.combo;
        }
    },

    miss: (note) => {
        note.missed = true;
        engine.state.combo = 0;
        engine.state.stats.miss++;
        engine.feedback("MISS", "#ff3333");
    },

    feedback: (text, color) => {
        const el = document.getElementById('feedback');
        el.innerText = text;
        el.style.color = color;
        el.style.transform = "translate(-50%, -50%) scale(1.3)";
        setTimeout(() => el.style.transform = "translate(-50%, -50%) scale(1)", 50);
        
        // Clear previous timeout to prevent clearing new text
        if(engine.fbTimeout) clearTimeout(engine.fbTimeout);
        engine.fbTimeout = setTimeout(() => el.innerText = "", 500);
    },

    start: () => {
        if(!engine.state.audioBuffer) return;
        document.getElementById('menu').classList.add('hidden');
        
        // Audio
        const src = audioCtx.createBufferSource();
        src.buffer = engine.state.audioBuffer;
        src.connect(audioCtx.destination);
        src.start(0);
        engine.state.audioSource = src;
        
        engine.state.startTime = audioCtx.currentTime;
        engine.state.playing = true;
        
        requestAnimationFrame(engine.loop);
    },

    loop: () => {
        if(!engine.state.playing) return;
        const now = (audioCtx.currentTime - engine.state.startTime) * 1000;
        
        // Update Physics
        engine.update(now);
        // Render
        engine.draw(now);

        // End Check
        if (now > (engine.state.audioBuffer.duration * 1000) + 1000) {
            engine.end();
        } else {
            requestAnimationFrame(engine.loop);
        }
    },

    update: (now) => {
        // Pre-calculate visual position of the song right now
        const currentVisualPos = engine.state.integrator.getVisualPos(now);

        // Render Window (e.g., 2000px worth of visual distance)
        const renderDist = 2000; 

        // Get notes to render
        engine.state.activeNotes = engine.state.mapData.notes.filter(n => {
            // Already hit or missed way back? Skip
            if (n.hit || n.missed) return false;
            
            // Calculate note's relative visual position
            // Since note.visualPos is absolute, and currentVisualPos is absolute
            const relPos = n.visualPos - currentVisualPos;
            
            // Miss Logic
            if (relPos < -200) { // Passed the hit line significantly
                 // In SV, checking pixels is safer than time for negative SVs,
                 // but checking time is standard for hit windows.
                 // Let's rely on time for miss detection to be safe against stop-events.
                 if (now > n.time + CONFIG.hitWindows.miss) {
                     engine.miss(n);
                     return false;
                 }
            }

            // Is it on screen?
            // Top of screen is roughly relPos 1000 (depending on scroll speed)
            return relPos > -500 && relPos < renderDist;
        });
    },

    draw: (now) => {
        const ctx = document.getElementById('gameCanvas').getContext('2d');
        const W = ctx.canvas.width;
        const H = ctx.canvas.height;
        const currentVisualPos = engine.state.integrator.getVisualPos(now);
        const laneW = CONFIG.laneWidth;
        const offsetX = (W - (laneW * 4)) / 2;

        // BG
        ctx.fillStyle = '#111';
        ctx.fillRect(0, 0, W, H);

        // Lanes
        for(let i=0; i<4; i++) {
            const lx = offsetX + (i * laneW);
            
            // Lane Line
            ctx.strokeStyle = '#222';
            ctx.beginPath(); ctx.moveTo(lx,0); ctx.lineTo(lx,H); ctx.stroke();

            // Receptor
            const pressed = engine.state.heldKeys[i];
            ctx.fillStyle = pressed ? 'rgba(255,255,255,0.2)' : 'transparent';
            ctx.fillRect(lx, 0, laneW, H); // Light up lane
            
            // Receptor Circle/Bar
            ctx.strokeStyle = pressed ? '#fff' : '#666';
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.arc(lx + laneW/2, CONFIG.hitY, 35, 0, Math.PI*2);
            ctx.stroke();
            
            // Key Label
            if(!pressed) {
                ctx.fillStyle = '#555';
                ctx.font = "bold 20px Arial";
                ctx.fillText(CONFIG.keys[i].toUpperCase(), lx + laneW/2 - 6, CONFIG.hitY + 6);
            }
        }

        // Notes
        engine.state.activeNotes.forEach(n => {
            const lx = offsetX + (n.col * laneW);
            
            // MATH MAGIC:
            // Calculate Y based on Visual Position difference
            const visualDist = n.visualPos - currentVisualPos;
            
            // Note: visualDist decreases as time goes on.
            // When visualDist is 0, note is at receptor.
            const y = CONFIG.hitY - visualDist;

            // Draw Note (Circle Style)
            ctx.fillStyle = CONFIG.colors[n.col];
            ctx.beginPath();
            ctx.arc(lx + laneW/2, y, 32, 0, Math.PI*2);
            ctx.fill();
            
            // White border
            ctx.strokeStyle = "white";
            ctx.lineWidth = 2;
            ctx.stroke();
        });

        // HUD Updates
        document.getElementById('score').innerText = engine.state.score.toString().padStart(7, '0');
        document.getElementById('hud-combo').innerText = engine.state.combo + "x";
        
        // Acc
        const total = engine.state.stats[300] + engine.state.stats[100] + engine.state.stats[50] + engine.state.stats.miss;
        if(total > 0) {
            const w = (engine.state.stats[300]*300) + (engine.state.stats[100]*100) + (engine.state.stats[50]*50);
            const acc = (w / (total*300)) * 100;
            document.getElementById('acc').innerText = acc.toFixed(2) + "%";
        }
    },

    end: () => {
        engine.state.playing = false;
        document.getElementById('final-score').innerText = `Score: ${Math.floor(engine.state.score)}`;
        document.getElementById('results').classList.remove('hidden');
    }
};

engine.init();

/**
 * FILE IMPORT LOGIC
 */
const fileInput = document.getElementById('fileInput');
const txtLoad = document.getElementById('loading-text');

fileInput.addEventListener('change', async (e) => {
    const file = e.target.files[0];
    if(!file) return;
    
    txtLoad.innerText = "Processing archive...";
    
    try {
        const zip = await JSZip.loadAsync(file);
        
        // Detect Format
        const osuFile = Object.keys(zip.files).find(f => f.endsWith('.osu'));
        const quaFile = Object.keys(zip.files).find(f => f.endsWith('.qua'));
        
        let mapData = null;
        let audioName = "";

        if (quaFile) {
            txtLoad.innerText = "Detected Quaver Map...";
            const content = await zip.file(quaFile).async("string");
            const parsed = parseQuaver(content);
            mapData = parsed.data;
            audioName = parsed.audio;
        } else if (osuFile) {
            txtLoad.innerText = "Detected Osu Map...";
            const content = await zip.file(osuFile).async("string");
            const parsed = parseOsu(content);
            mapData = parsed.data;
            audioName = parsed.audio;
        } else {
            throw new Error("No .osu or .qua file found.");
        }

        // Process SV Physics
        txtLoad.innerText = "Calculating Physics (SV)...";
        engine.state.integrator.build(mapData.svPoints, mapData.notes[mapData.notes.length-1].time);
        
        // Pre-calculate visual positions for all notes
        mapData.notes.forEach(n => {
            n.visualPos = engine.state.integrator.getVisualPos(n.time);
        });
        
        engine.state.mapData = mapData;

        // Load Audio
        txtLoad.innerText = `Decoding Audio: ${audioName}`;
        const audioF = Object.keys(zip.files).find(f => f.toLowerCase() === audioName.toLowerCase());
        
        if (audioF) {
            const ab = await zip.file(audioF).async("arraybuffer");
            engine.state.audioBuffer = await audioCtx.decodeAudioData(ab);
            document.getElementById('btn-start').disabled = false;
            txtLoad.innerText = "Ready!";
            document.getElementById('btn-start').innerText = "PLAY MAP";
            document.getElementById('btn-start').style.background = "#00ff00";
        } else {
            throw new Error(`Audio file '${audioName}' not found in zip.`);
        }

    } catch (err) {
        console.error(err);
        txtLoad.innerText = "Error: " + err.message;
    }
});

/**
 * PARSERS
 */

// --- QUAVER PARSER (.qua is YAML) ---
function parseQuaver(text) {
    const lines = text.split(/\r?\n/);
    let audio = "";
    let notes = [];
    let svPoints = []; // {time, multiplier}

    // Simple line scanner for YAML structure
    let section = "";
    let currentObj = {};
    
    for(let line of lines) {
        line = line.trim();
        if(!line || line.startsWith('#')) continue;

        if(line.startsWith("AudioFile:")) audio = line.split(":")[1].trim();

        // Detect Sections (Lists)
        if(line.startsWith("HitObjects:")) { section = "HitObjects"; continue; }
        if(line.startsWith("SliderVelocities:")) { section = "SV"; continue; }
        if(line.startsWith("TimingPoints:")) { section = "Timing"; continue; }

        if (line.startsWith("-")) {
            // New Item in list
            if (Object.keys(currentObj).length > 0) {
                if(section === "HitObjects" && currentObj.StartTime) {
                    // Quaver uses 1-4 for lanes
                    let col = parseInt(currentObj.Lane) - 1; 
                    if(col >= 0 && col < 4) {
                        notes.push({ time: parseInt(currentObj.StartTime), col: col, hit:false });
                    }
                }
                if(section === "SV" && currentObj.StartTime) {
                    svPoints.push({ time: parseInt(currentObj.StartTime), multiplier: parseFloat(currentObj.Multiplier) });
                }
            }
            currentObj = {}; // Reset
            line = line.substring(1).trim(); // Remove dash
        }

        // Parse Key: Value
        if (line.includes(":")) {
            const [k, v] = line.split(":", 2);
            currentObj[k.trim()] = v.trim();
        }
    }
    // Push last object
    if(section === "HitObjects" && currentObj.StartTime) {
         let col = parseInt(currentObj.Lane) - 1; 
         if(col >= 0 && col < 4) notes.push({ time: parseInt(currentObj.StartTime), col: col, hit:false });
    }
    if(section === "SV" && currentObj.StartTime) {
        svPoints.push({ time: parseInt(currentObj.StartTime), multiplier: parseFloat(currentObj.Multiplier) });
    }

    notes.sort((a,b) => a.time - b.time);
    return { audio, data: { notes, svPoints } };
}

// --- OSU PARSER (.osu) ---
function parseOsu(text) {
    const lines = text.split(/\r?\n/);
    let audio = "";
    let section = "";
    let notes = [];
    let svPoints = [];
    
    // Osu Inherited Points Logic
    // uninherited = 1 (Red Line): BPM change. beatLength = ms per beat.
    // uninherited = 0 (Green Line): SV change. beatLength = -100 / multiplier.
    
    for (let line of lines) {
        line = line.trim();
        if(!line) continue;
        if(line.startsWith("[")) { section = line; continue; }

        if(section === "[General]" && line.startsWith("AudioFilename:")) audio = line.split(":")[1].trim();

        if (section === "[TimingPoints]") {
            const p = line.split(",");
            if(p.length < 2) continue;
            const time = parseFloat(p[0]);
            const val = parseFloat(p[1]);
            const uninherited = p[6] === '1';

            if (!uninherited && val < 0) {
                // SV Change (Green Line)
                const mult = 100 / Math.abs(val);
                svPoints.push({ time: time, multiplier: mult });
            } else if (uninherited) {
                // BPM Change (Red Line) - Usually resets SV to 1.0 in osu!mania logic unless combined
                svPoints.push({ time: time, multiplier: 1.0 }); 
            }
        }

        if (section === "[HitObjects]") {
            const p = line.split(",");
            const x = parseInt(p[0]);
            const time = parseInt(p[2]);
            
            // Osu 4k Logic: floor(x * 4 / 512)
            let col = Math.floor(x * 4 / 512);
            col = Math.max(0, Math.min(3, col));
            notes.push({ time, col, hit:false });
        }
    }
    notes.sort((a,b) => a.time - b.time);
    return { audio, data: { notes, svPoints } };
}
</script>
</body>
</html>
