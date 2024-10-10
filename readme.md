<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Gorillas Game in A-Frame</title>
    <script src="https://aframe.io/releases/1.4.0/aframe.min.js"></script>
  </head>
  <body>
    <a-scene>
      <!-- Sky background -->
      <a-sky color="#87CEEB"></a-sky>

      <!-- Ground plane -->
      <a-plane rotation="-90 0 0" width="100" height="100" color="#7CFC00"></a-plane>

      <!-- Generate buildings -->
      <a-entity id="city"></a-entity>

      <!-- Player 1 Gorilla -->
      <a-sphere
        id="gorilla1"
        position="-45 12.5 -10"
        radius="1"
        color="#FF4500">
      </a-sphere>

      <!-- Player 2 Gorilla -->
      <a-sphere
        id="gorilla2"
        position="45 12.5 -10"
        radius="1"
        color="#1E90FF">
      </a-sphere>

      <!-- Camera -->
      <a-entity
        camera
        position="0 20 50"
        rotation="-20 0 0"
        look-controls
        wasd-controls>
      </a-entity>
    </a-scene>

    <!-- Scripts -->
    <script>
      // Generate buildings component
      AFRAME.registerComponent('generate-buildings', {
        init: function () {
          const city = this.el;
          const buildingCount = 20;
          for (let i = 0; i < buildingCount; i++) {
            const height = Math.random() * 10 + 5;
            const building = document.createElement('a-box');
            building.setAttribute('position', {
              x: (i - buildingCount / 2) * 5,
              y: height / 2,
              z: -10,
            });
            building.setAttribute('depth', 5);
            building.setAttribute('width', 5);
            building.setAttribute('height', height);
            building.setAttribute('color', '#8B4513');
            city.appendChild(building);
          }
        },
      });

      // Attach the generate-buildings component to the city entity
      document.querySelector('#city').setAttribute('generate-buildings', '');

      // Throw banana component
      AFRAME.registerComponent('throw-banana', {
        schema: {
          angle: { type: 'number', default: 45 },
          velocity: { type: 'number', default: 20 },
          origin: { type: 'vec3', default: { x: 0, y: 0, z: 0 } },
          fromPlayer: { type: 'number', default: 1 },
        },
        init: function () {
          const banana = this.el;
          let angleRad = (this.data.angle * Math.PI) / 180;
          const velocity = this.data.velocity;
          const origin = this.data.origin;
          const fromPlayer = this.data.fromPlayer;

          // Adjust angle for Player 2
          if (fromPlayer === 2) {
            angleRad = Math.PI - angleRad;
          }

          let time = 0;
          const gravity = -9.81;
          const deltaTime = 0.02;

          const velocityX = velocity * Math.cos(angleRad);
          const velocityY = velocity * Math.sin(angleRad);

          banana.setAttribute('position', origin);

          this.interval = setInterval(() => {
            time += deltaTime;
            const x = origin.x + velocityX * time;
            const y = origin.y + velocityY * time + 0.5 * gravity * time * time;
            if (y <= 0) {
              clearInterval(this.interval);
              banana.parentNode.removeChild(banana);
              // Start next turn
              nextTurn();
              return;
            }
            banana.setAttribute('position', { x: x, y: y, z: origin.z });
          }, deltaTime * 1000);
        },
        remove: function () {
          if (this.interval) {
            clearInterval(this.interval);
          }
        },
      });

      // Collision detection component
      AFRAME.registerComponent('check-collision', {
        tick: function () {
          const banana = this.el;
          const bananaPos = banana.getAttribute('position');
          const fromPlayer = parseInt(banana.getAttribute('from-player'));
          const targetGorillaId = fromPlayer === 1 ? '#gorilla2' : '#gorilla1';
          const targetGorilla = document.querySelector(targetGorillaId);
          const targetGorillaPos = targetGorilla.getAttribute('position');

          const distanceToTargetGorilla = distance(bananaPos, targetGorillaPos);

          if (distanceToTargetGorilla < 1) {
            alert(`Player ${fromPlayer} wins!`);
            banana.parentNode.removeChild(banana);
            // Stop the game or reset
            gameOver = true;
          }
        },
      });

      // Helper function to calculate distance
      function distance(pos1, pos2) {
        const dx = pos1.x - pos2.x;
        const dy = pos1.y - pos2.y;
        const dz = pos1.z - pos2.z;
        return Math.sqrt(dx * dx + dy * dy + dz * dz);
      }

      let currentPlayer = 1;
      let gameOver = false;

      // Function to handle player input and throw the banana
      function throwBanana(fromPlayer) {
        const angle = prompt(`Player ${fromPlayer}, enter angle (degrees):`, '45');
        if (angle === null) return; // Player cancelled
        const velocity = prompt(`Player ${fromPlayer}, enter velocity:`, '20');
        if (velocity === null) return;
        const banana = document.createElement('a-sphere');
        banana.setAttribute('color', '#FFFF00');
        banana.setAttribute('radius', '0.5');
        const originId = fromPlayer === 1 ? '#gorilla1' : '#gorilla2';
        const originEl = document.querySelector(originId);
        const originPos = originEl.getAttribute('position');
        banana.setAttribute('throw-banana', {
          angle: parseFloat(angle),
          velocity: parseFloat(velocity),
          origin: originPos,
          fromPlayer: fromPlayer,
        });
        banana.setAttribute('from-player', fromPlayer.toString());
        banana.setAttribute('check-collision', '');
        document.querySelector('a-scene').appendChild(banana);
      }

      // Function to handle turns
      function nextTurn() {
        if (gameOver) return;
        currentPlayer = currentPlayer === 1 ? 2 : 1;
        throwBanana(currentPlayer);
      }

      // Start the game
      window.onload = function () {
        throwBanana(currentPlayer);
      };
    </script>
  </body>
</html>
