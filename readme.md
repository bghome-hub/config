<!DOCTYPE html>
<html>
<head>
  <title>Gorillas Game</title>
  <style>
    body { margin: 0; overflow: hidden; background-color: skyblue; }
    canvas { display: block; background-color: lightblue; }
    #inputOverlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
      background: rgba(0, 0, 0, 0.5);
    }
    #inputOverlay div {
      background: white;
      padding: 20px;
      border-radius: 5px;
    }
    #inputOverlay label {
      display: block;
      margin-bottom: 5px;
    }
    #inputOverlay input {
      width: 100%;
      margin-bottom: 10px;
    }
    #inputOverlay button {
      width: 100%;
      padding: 10px;
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>

  <!-- Input Overlay -->
  <div id="inputOverlay" style="display: none;">
    <div>
      <h2>Player <span id="playerNumber"></span>'s Turn</h2>
      <label for="angleInput">Enter angle (degrees):</label>
      <input type="number" id="angleInput" min="0" max="180">
      <label for="velocityInput">Enter velocity:</label>
      <input type="number" id="velocityInput" min="0">
      <button id="submitBtn">Throw Banana</button>
    </div>
  </div>

  <script>
    // Set up the canvas
    var canvas = document.getElementById('gameCanvas');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    var ctx = canvas.getContext('2d');

    // Game constants
    var NUM_BUILDINGS = 15;
    var MIN_BUILDING_HEIGHT = canvas.height / 4;
    var MAX_BUILDING_HEIGHT = canvas.height / 1.5;
    var BUILDING_WIDTH = canvas.width / NUM_BUILDINGS;
    var GRAVITY = 9.8;
    var TIME_STEP = 0.02;
    var BANANA_RADIUS = 5;

    // Game variables
    var buildings = [];
    var gorillas = [];
    var currentPlayer = 0;
    var banana = null;
    var gameOver = false;

    // Generate buildings
    for (var i = 0; i < NUM_BUILDINGS; i++) {
      var buildingHeight = MIN_BUILDING_HEIGHT + Math.random() * (MAX_BUILDING_HEIGHT - MIN_BUILDING_HEIGHT);
      buildings.push({
        x: i * BUILDING_WIDTH,
        y: canvas.height - buildingHeight,
        width: BUILDING_WIDTH,
        height: buildingHeight
      });
    }

    // Place gorillas on buildings
    var player1Building = Math.floor(Math.random() * (NUM_BUILDINGS / 3));
    var player2Building = Math.floor(NUM_BUILDINGS * (2 / 3) + Math.random() * (NUM_BUILDINGS / 3));

    gorillas = [
      {
        x: buildings[player1Building].x + buildings[player1Building].width / 2,
        y: buildings[player1Building].y,
        color: 'red'
      },
      {
        x: buildings[player2Building].x + buildings[player2Building].width / 2,
        y: buildings[player2Building].y,
        color: 'blue'
      }
    ];

    // Draw functions
    function drawBuildings() {
      for (var i = 0; i < buildings.length; i++) {
        var b = buildings[i];
        ctx.fillStyle = 'gray';
        ctx.fillRect(b.x, b.y, b.width, b.height);

        // Windows
        ctx.fillStyle = 'yellow';
        for (var w = b.y + 10; w < canvas.height - 10; w += 20) {
          for (var v = b.x + 5; v < b.x + b.width - 10; v += 20) {
            ctx.fillRect(v, w, 10, 10);
          }
        }
      }
    }

    function drawGorillas() {
      for (var i = 0; i < gorillas.length; i++) {
        var g = gorillas[i];
        ctx.fillStyle = g.color;
        ctx.beginPath();
        ctx.arc(g.x, g.y - 10, 10, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    function drawBanana() {
      if (banana) {
        ctx.fillStyle = 'yellow';
        ctx.beginPath();
        ctx.arc(banana.x, banana.y, BANANA_RADIUS, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    function drawScene() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawBuildings();
      drawGorillas();
      drawBanana();
    }

    // Game logic
    function playerTurn() {
      if (gameOver) return;
      // Display input overlay
      var inputOverlay = document.getElementById('inputOverlay');
      inputOverlay.style.display = 'flex';
      document.getElementById('playerNumber').innerText = currentPlayer + 1;
      document.getElementById('angleInput').value = '';
      document.getElementById('velocityInput').value = '';
    }

    function startBananaThrow(angle, velocity) {
      var angleRad = angle * Math.PI / 180;
      var gorilla = gorillas[currentPlayer];
      var direction = currentPlayer === 0 ? 1 : -1;

      banana = {
        x: gorilla.x,
        y: gorilla.y - 10,
        vx: velocity * Math.cos(angleRad) * direction,
        vy: -velocity * Math.sin(angleRad),
      };

      animateBanana();
    }

    function animateBanana() {
      function update() {
        if (!banana) return;

        banana.x += banana.vx * TIME_STEP;
        banana.y += banana.vy * TIME_STEP;
        banana.vy += GRAVITY * TIME_STEP;

        if (checkCollision()) {
          banana = null;
          return;
        }

        if (banana.x < 0 || banana.x > canvas.width || banana.y > canvas.height) {
          banana = null;
          currentPlayer = 1 - currentPlayer;
          setTimeout(playerTurn, 500);
          return;
        }

        drawScene();
        requestAnimationFrame(update);
      }

      update();
    }

    function checkCollision() {
      // Collision with buildings
      for (var i = 0; i < buildings.length; i++) {
        var b = buildings[i];
        if (
          banana.x > b.x &&
          banana.x < b.x + b.width &&
          banana.y > b.y &&
          banana.y < b.y + b.height
        ) {
          currentPlayer = 1 - currentPlayer;
          setTimeout(playerTurn, 500);
          return true;
        }
      }

      // Collision with gorillas
      for (var i = 0; i < gorillas.length; i++) {
        var g = gorillas[i];
        var dx = banana.x - g.x;
        var dy = banana.y - (g.y - 10);
        var distance = Math.sqrt(dx * dx + dy * dy);
        if (distance < BANANA_RADIUS + 10) {
          gameOver = true;
          drawScene();
          setTimeout(function() {
            alert("Player " + (currentPlayer + 1) + " wins!");
            resetGame();
          }, 100);
          return true;
        }
      }

      return false;
    }

    function resetGame() {
      location.reload();
    }

    // Event listeners for input overlay
    document.getElementById('submitBtn').addEventListener('click', function() {
      var angle = parseFloat(document.getElementById('angleInput').value);
      var velocity = parseFloat(document.getElementById('velocityInput').value);

      if (isNaN(angle) || isNaN(velocity) || angle < 0 || angle > 180 || velocity <= 0) {
        alert('Please enter valid angle and velocity values.');
        return;
      }

      document.getElementById('inputOverlay').style.display = 'none';
      startBananaThrow(angle, velocity);
    });

    // Start the game
    drawScene();
    playerTurn();
  </script>
</body>
</html>
