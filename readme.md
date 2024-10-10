
<!DOCTYPE html>
<html>
<head>
  <title>Gorillas Game with Enhanced Features</title>
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
    var previousInputs = [{angle: 45, velocity: 50}, {angle: 45, velocity: 50}];
    var bananaImg = new Image();
    bananaImg.src = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAOCAYAAAB/73ocAAAABmJLR0QA/wD/AP+gvaeTAAACu0lEQVRIie2WP4iVURjHP9cYq+apSqVpS+RyaZSUUkiJQaLo3aUdJYgN0NoSm4aXFQpRExFUpFIqClpUyLlUCSKcH6mFaEDf3jH1u1nZPdnZ2d3Txfw5xe86b9m37HPP8/e77/fh1yAn1A99xTwtBA4AcSBQJmgAtwA+IEbkAl8AC4BQ6AThDYqppFfXArsA1UBVYB+IQQEIzfMJ7ABsAYNwPDABYk0nIkgS3+vG0r19RpIuY11e01CH8IroP6NpoNxJAE2Qx4CbpUq+Ad9kF+xHVvS6ur4dOAh4rFBSiOhbTi0CqI1pAJxBbSNZC6qygFa3JGkrnE1kBn0tANoDYbG1aUepNimVbf1plV7R0UpFfth2eX5oB1wFp6T6i10rQvgA+CLqoWJGgSTgHdSkwMSuB+4Bh4ABi0BRSDs8VD4DY4CLgKxAWzjs1O+i68DIp1eSlZeXioc2gJXgAvAQ+APsDRYDaYRHUBj4F1gAHgOjwDC0E3hgdAygNoIZl9XN6rOXotAHx4EjIdquYAw4EroFJYV6FbeBL0K/Sr2MekFHBpMg2eARcAV0oC3gDPAS3gSeAatA1kE0hEQJEp+0qS2rgGPhvcK0uciyVgO7Aq2nY5drUrR1CjYFfgIXAM+ARcAkzF9YlvAlG1IlgFnwdfArUAV6AEYEn4J1vqc+BkLJbVp17ivA4xN1jnhIXAJeCJ+DC4A98Bt4Gj9InZgT+YRCkBd2Ar6GpJNpAY+BQeBQ+AIPBG+AH6GjYAI4CFwAfjv+31vIqQX9J8pimxBNmCCeCZucVUA5sA14B9gPPgGzgnrWPU0+it6N3vOXpl3ufCz3/ppNxVTmZXUzZzpcPrKzXDY6LBbDFgXwrsoBNISapI6nbkPG0lYwWLCoyVIn6xSot3Q0mYLF3nOfC0fIEZSMi3yUov0gfy2pGcnE/Rr69Z0ap/fif3XQyT4xqGJ4AyYnNZxZUKvFv6Tm6mK/lcJxV7wOtQwqnl/tJg6l+a1kmq7qJgAAAABJRU5ErkJggg==';

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
      var player1Index = Math.floor(Math.random() * (NUM_BUILDINGS / 3));
      var player2Index = Math.floor(NUM_BUILDINGS * (2 / 3) + Math.random() * (NUM_BUILDINGS / 3));

      player1Building = player1Index;
      player2Building = player2Index;

      gorillas = [
        {
          x: buildings[player1Building].x + buildings[player1Building].width / 2,
          y: buildings[player1Building].y,
          color: 'red',
          falling: false,
          vy: 0
        },
        {
          x: buildings[player2Building].x + buildings[player2Building].width / 2,
          y: buildings[player2Building].y,
          color: 'blue',
          falling: false,
          vy: 0
        }
      ];
    }

    // Draw functions
    function drawBuildings() {
      for (var i = 0; i < buildings.length; i++) {
        var b = buildings[i];

        // Save context state
        ctx.save();

        // Create building path
        ctx.beginPath();
        ctx.rect(b.x, b.y, b.width, b.height);

        // Apply holes (damage)
        for (var h = 0; h < b.holes.length; h++) {
          var hole = b.holes[h];
          ctx.save();
          ctx.arc(hole.x, hole.y, hole.radius, 0, Math.PI * 2);
          ctx.clip();
        }

        // Fill building with windows
        ctx.fillStyle = 'gray';
        ctx.fill();

        // Draw windows
        ctx.fillStyle = 'yellow';
        for (var w = b.y + 10; w < canvas.height - 10; w += 20) {
          for (var v = b.x + 5; v < b.x + b.width - 10; v += 20) {
            ctx.fillRect(v, w, 10, 10);
          }
        }

        // Restore context state
        ctx.restore();
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

      // Pre-fill previous inputs
      document.getElementById('angleInput').value = previousInputs[currentPlayer].angle;
      document.getElementById('velocityInput').value = previousInputs[currentPlayer].velocity;
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
        if (!banana && !gorillas[0].falling && !gorillas[1].falling) return;

        if (banana) {
          banana.x += banana.vx * TIME_STEP * 10;
          banana.y += banana.vy * TIME_STEP * 10;
          banana.vy += (GRAVITY * TIME_STEP) / 10;

          // Update banana rotation
          banana.angle += 0.1 * direction;

          checkCollision();
        }

        // Update gorillas if they are falling
        for (var i = 0; i < gorillas.length; i++) {
          var g = gorillas[i];
          if (g.falling) {
            g.vy += GRAVITY * TIME_STEP;
            g.y += g.vy * TIME_STEP;

            // Check if gorilla landed on a building
            var landed = false;
            for (var j = 0; j < buildings.length; j++) {
              var b = buildings[j];
              if (
                g.x > b.x &&
                g.x < b.x + b.width &&
                g.y >= b.y - 10 &&
                !isPointInHole(b, g.x, b.y)
              ) {
                g.y = b.y;
                g.falling = false;
                g.vy = 0;
                landed = true;
                break;
              }
            }

            // If not landed on a building, check if on ground
            if (!landed && g.y >= canvas.height - 10) {
              g.y = canvas.height - 10;
              g.falling = false;
              g.vy = 0;
            }
          }
        }

        drawScene();

        if (banana && (banana.x < 0 || banana.x > canvas.width || banana.y > canvas.height)) {
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

    function isPointInHole(building, x, y) {
      for (var h = 0; h < building.holes.length; h++) {
        var hole = building.holes[h];
        var dx = x - hole.x;
        var dy = y - hole.y;
        var distance = Math.sqrt(dx * dx + dy * dy);
        if (distance < hole.radius) {
          return true;
        }
      }
      return false;
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
          // Add damage immediately
          applyDamage(b, banana.x, banana.y);

          banana = null;
          currentPlayer = 1 - currentPlayer;
          setTimeout(playerTurn, 500);
          return;
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
          return;
        }
      }
    }

    function applyDamage(building, x, y) {
      // Check if banana hits near an existing hole
      var existingHole = null;
      for (var h = 0; h < building.holes.length; h++) {
        var hole = building.holes[h];
        var dx = x - hole.x;
        var dy = y - hole.y;
        var distance = Math.sqrt(dx * dx + dy * dy);
        if (distance < hole.radius) {
          existingHole = hole;
          break;
        }
      }

      if (existingHole) {
        // Increase the hole radius
        existingHole.radius += 10;
      } else {
        // Add a new hole to the building at the collision point
        building.holes.push({
          x: x,
          y: y,
          radius: 20
        });
      }

      // Check if any gorilla is standing over a hole
      for (var i = 0; i < gorillas.length; i++) {
        var g = gorillas[i];
        if (
          g.x > building.x &&
          g.x < building.x + building.width &&
          g.y >= building.y - 10 &&
          isPointInHole(building, g.x, building.y)
        ) {
          g.falling = true;
        }
      }
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

      // Save previous inputs
      previousInputs[currentPlayer].angle = angle;
      previousInputs[currentPlayer].velocity = velocity;

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
