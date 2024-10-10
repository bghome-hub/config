
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tetris Game</title>
    <style>
        #grid {
            display: grid;
            grid-template-columns: repeat(10, 20px);
            grid-gap: 2px;
            width: 200px;
            height: 400px;
            border: 1px solid #ccc;
            overflow-y: scroll;
            padding: 5px;
        }

        .block {
            background-color: blue;
            display: inline-block;
            width: 20px;
            height: 20px;
        }
    </style>
</head>
<body>
    <h1>Tetris Game</h1>
    <div id="grid"></div>
    <button onclick="startGame()">Start Game</button>
    <p>Score: <span id="score">0</span></p>

    <script>
        // Tetromino types
        const tetrominos = [
            [[0, 1], [1, 1], [2, 1], [3, 1]],
            [[0, 1], [1, 1], [0, 0], [1, 0]],
            [[0, 1], [0, 1], [1, 1], [1, 0]],
            [[0, 0], [1, 0], [2, 0], [2, 1]]
        ];

        let score = 0;
        let grid = [];
        let currentTetromino = null;

        function startGame() {
            document.getElementById('grid').innerHTML = '';
            score = 0;
            document.getElementById('score').innerHTML = score;
            grid = Array(20).fill(0).map(() => Array(10).fill(0));
            currentTetromino = tetrominos[Math.floor(Math.random() * tetrominos.length)];
        }

        function addBlock(x, y) {
            const block = document.createElement('div');
            block.classList.add('block');
            grid[x][y] = 1;
            document.getElementById('grid').appendChild(block);
        }

        function removeBlock(x, y) {
            document.getElementById('grid').removeChild(document.querySelector(`.block[data-x="${x}"]`));
            grid[x][y] = 0;
        }

        function checkLines() {
            for (let i = 0; i < 20; i++) {
                let lineFilled = true;
                for (let j = 0; j < 10; j++) {
                    if (grid[i][j] === 0) {
                        lineFilled = false;
                        break;
                    }
                }
                if (lineFilled) {
                    score++;
                    document.getElementById('score').innerHTML = score;
                    grid.splice(i, 1);
                    grid.unshift(Array(10).fill(0));
                }
            }
        }

        function moveTetromino(dx, dy) {
            currentTetromino.forEach(([x, y]) => {
                const blockX = x + dx;
                const blockY = y + dy;
                if (blockX >= 0 && blockX < 10 && blockY >= 0 && blockY < 20 && grid[blockY][blockX] === 1) {
                    alert('Collision detected!');
                    return false;
                }
            });
            currentTetromino.forEach(([x, y]) => {
                const blockX = x + dx;
                const blockY = y + dy;
                removeBlock(x, y);
                addBlock(blockX, blockY);
            });
            return true;
        }

        function rotateTetromino() {
            currentTetromino = tetrominos[tetrominos.indexOf(currentTetromino)].map(([x, y]) => [y, -x]);
        }

        startGame();

        document.addEventListener('keydown', (e) => {
            switch (e.key) {
                case 'ArrowDown':
                    moveTetromino(0, 1);
                    checkLines();
                    break;
                case 'ArrowUp':
                    rotateTetromino();
                    addBlock(currentTetromino[0][0], currentTetromino[0][1]);
                    break;
                case 'ArrowLeft':
                    moveTetromino(-1, 0);
                    checkLines();
                    break;
                case 'ArrowRight':
                    moveTetromino(1, 0);
                    checkLines();
                    break;
            }
        });

        document.getElementById('grid').addEventListener('click', (e) => {
            const block = e.target.getBoundingClientRect();
            if (block.x + block.width >= currentTetromino[0][0] * 20 && block.x <= currentTetromino[0][0] * 20 + 20) {
                if (block.y + block.height >= currentTetromino[0][1] * 20 && block.y <= currentTetromino[0][1] * 20 + 20) {
                    moveTetromino(-currentTetromino[0][0], -currentTetromino[0][1]);
                }
            }
        });
    </script>
</body>
</html>
