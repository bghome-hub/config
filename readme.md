Sure! Let's create a simplified version of the classic "Gorillas" game using the A-Frame framework. In this game, two players take turns throwing projectiles at each other across a city skyline. We'll use A-Frame to build the 3D environment and handle the interactions.

Below is a step-by-step guide with code snippets to help you build the game.


---

1. Set Up the Basic A-Frame Scene

First, create an HTML file and include the A-Frame library.

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Gorillas Game in A-Frame</title>
    <script src="https://aframe.io/releases/1.4.0/aframe.min.js"></script>
  </head>
  <body>
    <a-scene>
      <!-- Your scene elements will go here -->
    </a-scene>
  </body>
</html>


---

2. Create the City Skyline

We'll generate random buildings to create a city skyline.

<a-scene>
  <!-- Sky background -->
  <a-sky color="#87CEEB"></a-sky>

  <!-- Ground plane -->
  <a-plane rotation="-90 0 0" width="100" height="100" color="#7CFC00"></a-plane>

  <!-- Generate buildings -->
  <a-entity id="city"></a-entity>

  <!-- Other elements will go here -->
</a-scene>

Add the following script to generate buildings:

<script>
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

  document.querySelector('#city').setAttribute('generate-buildings', '');
</script>


---

3. Add Gorillas on Top of Buildings

Place two gorillas (or placeholders) on top of the buildings at each end.

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

Adjust the y position based on the building heights if necessary.


---

4. Implement Projectile Motion for the Banana

We'll create a component to handle the banana's movement based on angle and velocity.

<script>
  AFRAME.registerComponent('throw-banana', {
    schema: {
      angle: { type: 'number', default: 45 },
      velocity: { type: 'number', default: 20 },
      origin: { type: 'vec3', default: { x: 0, y: 0, z: 0 } },
    },
    init: function () {
      const banana = this.el;
      const angleRad = (this.data.angle * Math.PI) / 180;
      const velocity = this.data.velocity;
      const origin = this.data.origin;

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
          return;
        }
        banana.setAttribute('position', { x: x, y: y, z: origin.z });
      }, deltaTime * 1000);
    },
  });
</script>


---

5. Create a Function to Handle Player Input

Since we can't get real-time input in this static example, we'll simulate it. In a real game, you'd capture user input for angle and velocity.

<script>
  function throwBanana(fromPlayer) {
    const angle = prompt(`Player ${fromPlayer}, enter angle (degrees):`, '45');
    const velocity = prompt(`Player ${fromPlayer}, enter velocity:`, '20');
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
    });
    document.querySelector('a-scene').appendChild(banana);
  }

  // Simulate turns
  setTimeout(() => {
    throwBanana(1);
  }, 1000);

  setTimeout(() => {
    throwBanana(2);
  }, 5000);
</script>


---

6. Detect Collision with the Opponent

We need to detect when the banana hits the other gorilla.

<script>
  AFRAME.registerComponent('check-collision', {
    tick: function () {
      const banana = this.el;
      const bananaPos = banana.getAttribute('position');
      const gorilla1 = document.querySelector('#gorilla1');
      const gorilla2 = document.querySelector('#gorilla2');
      const gorilla1Pos = gorilla1.getAttribute('position');
      const gorilla2Pos = gorilla2.getAttribute('position');

      const distanceToGorilla1 = distance(bananaPos, gorilla1Pos);
      const distanceToGorilla2 = distance(bananaPos, gorilla2Pos);

      if (distanceToGorilla1 < 1) {
        alert('Player 2 wins!');
        banana.parentNode.removeChild(banana);
      } else if (distanceToGorilla2 < 1) {
        alert('Player 1 wins!');
        banana.parentNode.removeChild(banana);
      }
    },
  });

  function distance(pos1, pos2) {
    const dx = pos1.x - pos2.x;
    const dy = pos1.y - pos2.y;
    const dz = pos1.z - pos2.z;
    return Math.sqrt(dx * dx + dy * dy + dz * dz);
  }
</script>

Attach the check-collision component to the banana entity:

banana.setAttribute('check-collision', '');

Update the throw-banana component's init function to include this line.


---

7. Add Camera and Controls

Add a camera so you can view the scene.

<a-entity
  camera
  position="0 20 50"
  rotation="-20 0 0"
  look-controls
  wasd-controls>
</a-entity>


---

8. Putting It All Together

Your final HTML file should look like this:

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Gorillas Game in A-Frame</title>
    <script src="https://aframe.io/releases/1.4.0/aframe.min.js"></script>
  </head>
  <body>
    <a-scene>
      <a-sky color="#87CEEB"></a-sky>
      <a-plane rotation="-90 0 0" width="100" height="100" color="#7CFC00"></a-plane>
      <a-entity id="city"></a-entity>
      <a-sphere id="gorilla1" position="-45 12.5 -10" radius="1" color="#FF4500"></a-sphere>
      <a-sphere id="gorilla2" position="45 12.5 -10" radius="1" color="#1E90FF"></a-sphere>
      <a-entity
        camera
        position="0 20 50"
        rotation="-20 0 0"
        look-controls
        wasd-controls>
      </a-entity>
    </a-scene>

    <!-- Include all the scripts here -->
    <!-- ... (All the scripts from previous steps) ... -->
  </body>
</html>


---

9. Next Steps

Enhance Graphics: Replace the spheres with gorilla models.

Improve Controls: Create a user interface for input instead of prompt().

Randomize Buildings: Adjust building heights and positions dynamically.

Add Wind Factor: Introduce wind to affect the banana's trajectory.

Multiplayer Support: Allow two players to play over the network.



---

Note: This is a simplified example to demonstrate how you might create a game similar to "Gorillas" using A-Frame. Developing a fully functional game would require more advanced features and handling edge cases.

Let me know if you need further assistance or have questions about specific parts of the code!

