soccer-game/
│
├── server.js
├── package.json
└── public/
    └── index.html
    {
  "name": "soccer-game",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.0",
    "ws": "^8.0.0"
  }
}
const express = require("express");
const http = require("http");
const WebSocket = require("ws");

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

app.use(express.static("public"));

let players = {};
let ball = { x: 500, y: 300, vx: 0, vy: 0 };

function broadcast() {
    const data = JSON.stringify({
        type: "state",
        players,
        ball
    });

    wss.clients.forEach(client => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(data);
        }
    });
}

wss.on("connection", (ws) => {

    const id = Math.random().toString(36).substr(2, 9);

    players[id] = {
        x: 200 + Math.random() * 600,
        y: 300,
        score: 0
    };

    ws.send(JSON.stringify({
        type: "init",
        id,
        players,
        ball
    }));

    ws.on("message", (msg) => {
        const data = JSON.parse(msg);

        if (data.type === "move") {
            if (players[id]) {
                players[id].x = data.x;
                players[id].y = data.y;
            }
        }

        broadcast();
    });

    ws.on("close", () => {
        delete players[id];
        broadcast();
    });
});

const PORT = process.env.PORT || 3000;

server.listen(PORT, () => {
    console.log("Server running on port " + PORT);
});
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Online Soccer Game</title>

<style>
body {
    margin: 0;
    background: #111;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    font-family: monospace;
}

canvas {
    background: linear-gradient(#2e8b57, #1b5e3b);
    border: 3px solid black;
}
</style>
</head>

<body>

<canvas id="game" width="1000" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host);

let myId = null;
let players = {};
let ball = { x: 500, y: 300 };

const keys = {};

document.addEventListener("keydown", e => keys[e.key] = true);
document.addEventListener("keyup", e => keys[e.key] = false);

ws.onmessage = (msg) => {
    const data = JSON.parse(msg.data);

    if (data.type === "init") {
        myId = data.id;
        players = data.players;
        ball = data.ball;
    }

    if (data.type === "state") {
        players = data.players;
        ball = data.ball;
    }
};

function sendMove(p) {
    ws.send(JSON.stringify({
        type: "move",
        x: p.x,
        y: p.y
    }));
}

function update() {

    if (!players[myId]) return;

    let p = players[myId];
    const speed = 5;

    if (keys["w"]) p.y -= speed;
    if (keys["s"]) p.y += speed;
    if (keys["a"]) p.x -= speed;
    if (keys["d"]) p.x += speed;

    p.x = Math.max(20, Math.min(980, p.x));
    p.y = Math.max(20, Math.min(580, p.y));

    sendMove(p);
}

function drawField() {
    ctx.fillStyle = "#2e8b57";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.strokeStyle = "white";
    ctx.lineWidth = 3;

    ctx.beginPath();
    ctx.moveTo(500, 0);
    ctx.lineTo(500, 600);
    ctx.stroke();

    ctx.beginPath();
    ctx.arc(500, 300, 80, 0, Math.PI * 2);
    ctx.stroke();

    ctx.fillStyle = "white";
    ctx.fillRect(0, 250, 10, 100);
    ctx.fillRect(990, 250, 10, 100);
}

function draw() {

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    drawField();

    ctx.fillStyle = "white";
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, 12, 0, Math.PI * 2);
    ctx.fill();

    for (let id in players) {
        ctx.fillStyle = id === myId ? "blue" : "red";

        ctx.beginPath();
        ctx.arc(players[id].x, players[id].y, 20, 0, Math.PI * 2);
        ctx.fill();
    }
}

function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
}

loop();
</script>

</body>
</html>
