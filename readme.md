
<!DOCTYPE html>
<html>
<head>
  <title>Gorillas Game with Building Damage and Banana Image</title>
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
    var GRAVITY = 9.8 * 100; // Scale gravity for better visual
    var TIME_STEP = 0.02;
    var BANANA_WIDTH = 30;
    var BANANA_HEIGHT = 15;

    // Game variables
    var buildings = [];
    var gorillas = [];
    var currentPlayer = 0;
    var banana = null;
    var gameOver = false;

    // Load banana image
    var bananaImg = new Image();
    bananaImg.src = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAYAAACqaXHeAAAFIklEQVR4Xu3bUYhVZRzH8c+3usViAAqNqqoIUKAKkFopYSqAjBRQVUIFBK1IlKhUKIigIgiiIvoIDKULRS6qC+IVFAKCJaD0oFoYmARVUJKRKlCKGJVq1Vaa5Jr9/S2zd3a3fOnc3dnZ2e3ee++53Pvznf6vXbX3dZlHy+z8Pj43+99zgz+EH81vUGR0QHhmpHVB0eQdR0QHR2QSx8KjIXMNYzeb8kPiqU5qB3ZMH/dx9H15FzwV9qHHEF3uU4Zsyi4JbdmY1IPrBOPCOjGOf2I6FHRZd9BrnCq/QJ6xjkxDdNV/rzDGZep6iXwvWNRnFpdj+avRGtfQUzq8Yo3quk6Me6rPIWt9BTNauV+An6hxcPV0xz/vA8oZ6xqX4afgBTjr1VOYftj/6bPHl9Ba0x6sqnxF/AtOj3g/PbXkdearYPF1PXjTv/TKjGmSivx20bop6u6R+H1NVEeHUoIYc5Sqa9N3j9KX0Gv+14v/hTrG+s56v3NL9ZzGeXPok+udRnN6mowqdwilfOqzlbfotfb9mEoDzmbTUpJHvjUrVq1bq7+7jm9fVHVY2S3h2WV9ANwDGxO8IPVu0P3kVejGzOdkf0nfJRR/7kZr8mDoAjHV+XMSqJswg48lLnsab6E22Y5ewSPpWdw6izSqt6Az0HEj/DNOvzR9Aa3fjpZJ9RnOq0POY9pDrvuR+eBZ5s8VPSxncVPTpYjggE+DAqYalclflbZrQJPG6TXmpm0GLVY3txyF+jJvaE6BpblWM6r1SfuFPdxpHsXx0ALgTdBpXYFrKFezqUHUUV1RnGpZpBsKknPTmSAJwbXBie4yjHtZ7B2zN8dmpre22U6Aj80dX8a9R1ER0d1U9t86/iqTdgy6/qIdRHh0O6QZfNzjC/N+DnLAFs74+eMcOk2jdTOhBegeabjTA79m7yN+6Vj8AEiZx1KezGvZUc6rhaxHsW4cI9ayjpV76ivx23nAPDR3m6gj+KpJ2wLYI5aAj1rJ0o++CO7JD4dpVD+d/XeHgPpGV1DdZ34qMN2PvFOv2+hxagIcX3gZ2eQzUR9HBRvbk6FehZwT0ELgjtAsbErh3d0Mc2kgLAC3wYkAXG8gFrchcFhCLABzwYkAHF84Frcl8MhlRHgYSHQBY6ym+F2CYj0mC8FxeCxcA28FrQMyA8zXB6YB2sR7CBwC++DMoT0iLADTwZkCbFoyBNjz9BEjYx2lV+0rZzHnhXIXAA2sC2iT1HR0VwF3gYODuLPqv1MfO5KjAuAptF42KC94Sry7XwNH8eNlcS4SrgIvBWtE6wnyl2sPHgZVfi8KjAXRJR0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0RlER0Rldc6P/45/AtfxnDkDAAAAABJRU5ErkJggg==';

    // Generate buildings
    function generateBuildings() {
      buildings = [];
      for (var i = 0; i < NUM_BUILDINGS; i++) {
        var buildingHeight = MIN_BUILDING_HEIGHT + Math.random() * (MAX_BUILDING_HEIGHT - MIN_BUILDING_HEIGHT);
        buildings.push({
          x: i * BUILDING_WIDTH,
          y: canvas.height - buildingHeight,
          width: BUILDING_WIDTH,
          height: buildingHeight,
          holes: [] // Array to store damage holes
        });
      }
    }

    // Place gorillas on buildings
    function placeGorillas() {
      var availableBuildings = buildings.slice();
      var player1Index = Math.floor(Math.random() * (NUM_BUILDINGS / 3));
      var player2Index = Math.floor(NUM_BUILDINGS * (2 / 3) + Math.random() * (NUM_BUILDINGS / 3));

      player1Building = player1Index;
      player2Building = player2Index;

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
    }

    // Draw functions
    function drawBuildings() {
      for (var i = 0; i < buildings.length; i++) {
        var b = buildings[i];

        // Save context state
        ctx.save();

        // Draw building
        ctx.fillStyle = 'gray';
        ctx.fillRect(b.x, b.y, b.width, b.height);

        // Apply holes (damage)
        ctx.globalCompositeOperation = 'destination-out';
        ctx.fillStyle = 'black';
        for (var h = 0; h < b.holes.length; h++) {
          var hole = b.holes[h];
          ctx.beginPath();
          ctx.arc(hole.x, hole.y, hole.radius, 0, Math.PI * 2);
          ctx.fill();
        }

        // Restore context state
        ctx.restore();

        // Draw windows
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
        ctx.save();
        ctx.translate(banana.x, banana.y);
        ctx.rotate(banana.angle);
        ctx.drawImage(bananaImg, -BANANA_WIDTH / 2, -BANANA_HEIGHT / 2, BANANA_WIDTH, BANANA_HEIGHT);
        ctx.restore();
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
        angle: angleRad * direction
      };

      animateBanana();
    }

    function animateBanana() {
      function update() {
        if (!banana) return;

        banana.x += banana.vx * TIME_STEP * 10;
        banana.y += banana.vy * TIME_STEP * 10;
        banana.vy += (GRAVITY * TIME_STEP) / 10;

        // Update banana rotation
        banana.angle += 0.1 * direction;

        drawScene();

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

        requestAnimationFrame(update);
      }

      var direction = currentPlayer === 0 ? 1 : -1;
      update();
    }

    function checkCollision() {
      // Collision with buildings
      for (var i = 0; i < buildings.length; i++) {
        var b = buildings[i];
        if (
          banana.x + BANANA_WIDTH / 2 > b.x &&
          banana.x - BANANA_WIDTH / 2 < b.x + b.width &&
          banana.y + BANANA_HEIGHT / 2 > b.y &&
          banana.y - BANANA_HEIGHT / 2 < b.y + b.height
        ) {
          // Add a hole to the building at the collision point
          b.holes.push({
            x: banana.x,
            y: banana.y,
            radius: 20
          });
          banana = null;
          currentPlayer = 1 - currentPlayer;
          setTimeout(playerTurn, 500);
          return true;
        }
      }

      // Collision with gorillas
      for (var i = 0; i < gorillas.length; i++) {
        if (i === currentPlayer) continue; // Skip collision with the throwing gorilla
        var g = gorillas[i];
        var dx = banana.x - g.x;
        var dy = banana.y - (g.y - 10);
        var distance = Math.sqrt(dx * dx + dy * dy);
        if (distance < 20) {
          gameOver = true;
          banana = null;
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

    // Initialize the game
    function initGame() {
      generateBuildings();
      placeGorillas();
      drawScene();
      setTimeout(playerTurn, 500);
    }

    // Handle window resize
    window.addEventListener('resize', function() {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
      BUILDING_WIDTH = canvas.width / NUM_BUILDINGS;
      MIN_BUILDING_HEIGHT = canvas.height / 4;
      MAX_BUILDING_HEIGHT = canvas.height / 1.5;
      // Recalculate building positions
      for (var i = 0; i < buildings.length; i++) {
        buildings[i].x = i * BUILDING_WIDTH;
        buildings[i].width = BUILDING_WIDTH;
        buildings[i].y = canvas.height - buildings[i].height;
      }
      // Reposition gorillas
      gorillas[0].x = buildings[player1Building].x + buildings[player1Building].width / 2;
      gorillas[0].y = buildings[player1Building].y;
      gorillas[1].x = buildings[player2Building].x + buildings[player2Building].width / 2;
      gorillas[1].y = buildings[player2Building].y;
      drawScene();
    });

    // Start the game when the banana image is loaded
    bananaImg.onload = function() {
      initGame();
    };
  </script>
</body>
</html>
