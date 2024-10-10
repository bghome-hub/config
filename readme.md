<!DOCTYPE html>
<html>
<head>
  <title>Gorillas Game</title>
  <style>
    body { margin: 0; overflow: hidden; background-color: skyblue; }
    canvas { display: block; background-color: lightblue; }
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>
  <script>
    // Set up the canvas
    var canvas = document.getElementById('gameCanvas');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    var ctx = canvas.getContext('2d');

    // Game constants
    var NUM_BUILDINGS = 20;
    var MIN_BUILDING_HEIGHT = canvas.height / 8;
    var MAX_BUILDING_HEIGHT = canvas.height / 2;
    var BUILDING_WIDTH = canvas.width / NUM_BUILDINGS;
    var GRAVITY = 9.8;
    var VELOCITY_SCALE = 2;
    var GRAVITY_SCALE = 0.5;

    // Game variables
    var buildings = [];
    var gorillas = [];
    var currentPlayer = 0;
    var banana = null;

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
        y: buildings[player1Building].y - 10,
        color: 'red'
      },
      {
        x: buildings[player2Building].x + buildings[player2Building].width / 2,
        y: buildings[player2Building].y - 10,
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
          for (var v = b.x + 5; v < b.x + b.width - 5; v += 15) {
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
        ctx.arc(g.x, g.y, 10, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    function drawBanana() {
      if (banana) {
        ctx.fillStyle = 'yellow';
        ctx.beginPath();
        ctx.arc(banana.x, banana.y, banana.radius, 0, Math.PI * 2);
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
      var angle = parseFloat(prompt("Player " + (currentPlayer + 1) + ": Enter angle (degrees):"));
      var velocity = parseFloat(prompt("Player " + (currentPlayer + 1) + ": Enter velocity:"));

      if (isNaN(angle) || isNaN(velocity)) {
        alert("Invalid input. Try again.");
        playerTurn();
        return;
      }

      var angleRad = angle * Math.PI / 180;
      var gorilla = gorillas[currentPlayer];
      var x0 = gorilla.x;
      var y0 = gorilla.y;

      banana = {
        x: x0,
        y: y0,
        vx: velocity * Math.cos(angleRad) * (currentPlayer === 0 ? 1 : -1) * VELOCITY_SCALE,
        vy: -velocity * Math.sin(angleRad) * VELOCITY_SCALE,
        radius: 5
      };

      animateBanana();
    }

    function animateBanana() {
      function update() {
        banana.x += banana.vx * 0.02;
        banana.y += banana.vy * 0.02;
        banana.vy += GRAVITY * GRAVITY_SCALE * 0.02;

        if (checkCollision()) {
          return;
        }

        if (banana.x < 0 || banana.x > canvas.width || banana.y > canvas.height) {
          banana = null;
          currentPlayer = 1 - currentPlayer;
          playerTurn();
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
        if (banana.x > b.x && banana.x < b.x + b.width && banana.y > b.y) {
          banana = null;
          currentPlayer = 1 - currentPlayer;
          playerTurn();
          return true;
        }
      }

      // Collision with gorillas
      for (var i = 0; i < gorillas.length; i++) {
        var g = gorillas[i];
        var dx = banana.x - g.x;
        var dy = banana.y - g.y;
        var distance = Math.sqrt(dx * dx + dy * dy);
        if (distance < banana.radius + 10) {
          alert("Player " + (currentPlayer + 1) + " wins!");
          resetGame();
          return true;
        }
      }

      return false;
    }

    function resetGame() {
      location.reload();
    }

    // Start the game
    drawScene();
    playerTurn();
  </script>
</body>
</html>
