<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gorilla Game</title>
    <style>
        #map {
            white-space: pre;
            font-family: monospace;
            padding: 10px;
            border: 1px solid #ccc;
            width: 200px;
            height: 300px;
            overflow-y: scroll;
        }
    </style>
</head>
<body>
    <h1>Gorilla Game</h1>
    <div id="map"></div>
    <input type="text" id="direction" placeholder="Enter a direction (w, a, s, d)">
    <button onclick="movePlayer()">Move Player</button>
    <p>Score: <span id="score">0</span></p>

    <script>
        // Game constants and variables
        const WIDTH = 10;
        const HEIGHT = 20;

        let map = Array(WIDTH).fill(0).map(() => Array(HEIGHT).fill(' '));
        let playerPosition = [5, 15];
        let score = 0;

        function generateMap() {
            for (let i = 0; i < HEIGHT; i++) {
                for (let j = 0; j < WIDTH; j++) {
                    if (Math.random() < 0.5) {
                        // Randomly spawn gorillas
                        map[j][i] = 'G';
                    } else if (j === playerPosition[0] && i === playerPosition[1]) {
                        map[j][i] = 'P';
                    }
                }
            }

            let mapString = '';
            for (let i = 0; i < HEIGHT; i++) {
                for (let j = 0; j < WIDTH; j++) {
                    mapString += map[j][i];
                }
                mapString += '\n';
            }
            document.getElementById('map').innerHTML = mapString;
        }

        function movePlayer() {
            const direction = document.getElementById('direction').value.trim();
            switch (direction) {
                case 'w':
                    playerPosition[1]--;
                    if (map[playerPosition[0]][playerPosition[1]] === 'G') {
                        // Gorilla killed, add score
                        score++;
                        map[playerPosition[0]][playerPosition[1]] = '';
                    } else if (map[playerPosition[0]][playerPosition[1]] !== '') {
                        alert('You hit something!');
                        playerPosition[1]++;
                    }
                    break;
                case 's':
                    playerPosition[1]++;
                    if (map[playerPosition[0]][playerPosition[1]] === 'G') {
                        // Gorilla killed, add score
                        score++;
                        map[playerPosition[0]][playerPosition[1]] = '';
                    } else if (map[playerPosition[0]][playerPosition[1]] !== '') {
                        alert('You hit something!');
                        playerPosition[1]--;
                    }
                    break;
                case 'a':
                    playerPosition[0]--;
                    if (map[playerPosition[0]][playerPosition[1]] === 'G') {
                        // Gorilla killed, add score
                        score++;
                        map[playerPosition[0]][playerPosition[1]] = '';
                    } else if (map[playerPosition[0]][playerPosition[1]] !== '') {
                        alert('You hit something!');
                        playerPosition[0]++;
                    }
                    break;
                case 'd':
                    playerPosition[0]++;
                    if (map[playerPosition[0]][playerPosition[1]] === 'G') {
                        // Gorilla killed, add score
                        score++;
                        map[playerPosition[0]][playerPosition[1]] = '';
                    } else if (map[playerPosition[0]][playerPosition[1]] !== '') {
                        alert('You hit something!');
                        playerPosition[0]--;
                    }
                    break;
            }

            document.getElementById('score').innerHTML = score;

            generateMap();
        }

        generateMap();
    </script>
</body>
</html>
