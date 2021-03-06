# Wand of Regrowth Simulator

<template class="style-placeholder">
  <style>
    #canvas {
      border: 2px dotted black;
      margin: auto;
    }

    #canvas:hover {
      cursor: pointer;
    }
  </style>
</template>

This tool helps you farm dewdrops and seeds with a Wand of Regrowth.

The little number inside each tile equals the number of high grass patches that will spawn when you zap that tile.

Click on a tile to edit the dungeon layout.

<label>Wand of Regrowth upgrade level: <input type="number" min="0" max="15" value="0" id="wand_level"></label>

Maximum # of high grass spawnable: <output for="wand_level" id="max_high_grass"></output>

<div class="canvas-wrapper">
  <canvas id="canvas"></canvas>
</div>

<script>

document.head.appendChild(
  document.querySelector('template.style-placeholder').content
);

const TileType = Object.freeze({
  FLOOR:      0,
  WALL:       1,
  GRASS:      2,
  HIGH_GRASS: 3,
});

const tile_preset = [
  '                 00000000 0               0 ',
  '               0 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 0               0 00000000 ',
  '                 00000000 00000000 00000000 ',
  '                 00000000 00000000 00000000 ',
  '0                00000000 00000000 00000000 ',
  '00           00  00000000 00000000 00000000 ',
  '000          00  00000000 00000000  0000000 ',
  '0000             00000000 000000000  000000 ',
  '00000            00000000 0000000000 000000 ',
];

class Tile {
  constructor(type = TileType.WALL) {
    this.type = type;
    this.mistAmount = 0;
    this.mistAmountNext = 0;
  }

  isMistSpreadable() { return this.type !== TileType.WALL; }
}

class GameMap {
  constructor() {
    const tiles = this.tiles_ = tile_preset.map(tile_row_str =>
      tile_row_str.split('').map(ch => new Tile(ch === '0' ? TileType.WALL : TileType.FLOOR))
    );

    this.mapWidth_ = tiles[0].length;
    this.mapHeight_ = tiles.length;

    // Safety check
    for (const [y, tile_row] of tiles.entries())
      if (tile_row.length !== tiles[0].length)
       throw new Error(`Map is not rectangular! See row ${y}`);
  }

  isOutsideMap(x, y) {
    return x < 0 || x >= this.mapWidth_ || y < 0 || y >= this.mapHeight_;
  }

  getTileType(x, y) {
    return this.isOutsideMap(x, y) ? TileType.FLOOR : this.tiles_[y][x].type;
  }

  setTileType(x, y, type) {
    if (this.isOutsideMap(x, y))
      throw new Error(`({x}, {y}) is outside bounds!`);
    else
      this.tiles_[y][x].type = type;
  }

  resetMistAndGrass() {
    for (const row of this.tiles_) {
      for (const tile of row) {
        tile.mistAmount = 0;
        if (tile.type === TileType.GRASS || tile.type === TileType.HIGH_GRASS)
          tile.type = TileType.FLOOR;
      }
    }
  }

  getTotalMistAmount() {
    return this.tiles_.reduce(
      (rowSum, row) => rowSum + row.reduce(
        (sum, tile) => sum + tile.mistAmount, 0
      ), 0
    );
  }

  getWidth() { return this.mapWidth_; }
  getHeight() { return this.mapHeight_; }

  getTile_(x, y) { return this.tiles_[y][x]; }

  simulateMist(x, y, initialAmount) {
    if (this.isOutsideMap(x, y))
      throw new Error(`({x}, {y}) is outside bounds!`);

    this.getTile_(x, y).mistAmount = initialAmount;
    let searchXMin = Math.max(0, x - 1), searchXMax = Math.min(this.mapWidth_ - 1, x + 1);
    let searchYMin = Math.max(0, y - 1), searchYMax = Math.min(this.mapHeight_ - 1, y + 1);
    let isMistRemaining = true;

    while (isMistRemaining) {
      // For each tile, compute mistAmountNext (amount of mist that will be present on the next turn)
      for (let x = searchXMin; x <= searchXMax; ++x) {
        for (let y = searchYMin; y <= searchYMax; ++y) {
          const tile = this.getTile_(x, y);
          if (!tile.isMistSpreadable())
            continue;

          // This tile
          let spreadableTileCount = 1;
          let totalMistAmountInAdjacentTiles = tile.mistAmount;

          if (x - 1 >= 0) {
            const leftTile = this.getTile_(x - 1, y);
            if (leftTile.isMistSpreadable()) {
              ++spreadableTileCount;
              totalMistAmountInAdjacentTiles += leftTile.mistAmount;
            }
          }

          if (x + 1 < this.mapWidth_) {
            const rightTile = this.getTile_(x + 1, y);
            if (rightTile.isMistSpreadable()) {
              ++spreadableTileCount;
              totalMistAmountInAdjacentTiles += rightTile.mistAmount;
            }
          }

          if (y - 1 >= 0) {
            const aboveTile = this.getTile_(x, y - 1);
            if (aboveTile.isMistSpreadable()) {
              ++spreadableTileCount;
              totalMistAmountInAdjacentTiles += aboveTile.mistAmount;
            }
          }

          if (y + 1 < this.mapHeight_) {
            const belowTile = this.getTile_(x, y + 1);
            if (belowTile.isMistSpreadable()) {
              ++spreadableTileCount;
              totalMistAmountInAdjacentTiles += belowTile.mistAmount;
            }
          }

          tile.mistAmountNext = Math.max(0, Math.floor(totalMistAmountInAdjacentTiles / spreadableTileCount) - 1);
        }
      }

      let affectedXMin = this.mapWidth_, affectedXMax = -1;
      let affectedYMin = this.mapHeight_, affectedYMax = -1;
      isMistRemaining = false;

      // Update mistAmount
      for (let x = searchXMin; x <= searchXMax; ++x) {
        for (let y = searchYMin; y <= searchYMax; ++y) {
          const tile = this.getTile_(x, y);
          if (!tile.isMistSpreadable())
            continue;

          tile.mistAmount = tile.mistAmountNext;
          if (tile.mistAmount <= 0)
            continue;

          // Postcondition: This tile has mist
          isMistRemaining = true;
          affectedXMin = Math.min(affectedXMin, x);
          affectedXMax = Math.max(affectedXMax, x);
          affectedYMin = Math.min(affectedYMin, y);
          affectedYMax = Math.max(affectedYMax, y);

          if (tile.mistAmount > 9)
            tile.type = TileType.HIGH_GRASS;
          else if (tile.type !== TileType.HIGH_GRASS)
            tile.type = TileType.GRASS;
        }
      }

      // Set search region for next turn
      searchXMin = Math.max(0, affectedXMin - 1);
      searchXMax = Math.min(this.mapWidth_ - 1, affectedXMax + 1);
      searchYMin = Math.max(0, affectedYMin - 1);
      searchYMax = Math.min(this.mapHeight_ - 1, affectedYMax + 1);
    }

    let grassCount = 0;
    let highGrassCount = 0;
    for (const row of this.tiles_) {
      for (const tile of row) {
        if (tile.type === TileType.HIGH_GRASS)
          ++highGrassCount;
        else if (tile.type === TileType.GRASS)
          ++grassCount;
      }
    }

    this.resetMistAndGrass();
    return {grassCount, highGrassCount};
  }
}


const gameMap = new GameMap;

// function computeOptimalZapLocation(wandLevel) {
//   const initialAmount = 40 + wandLevel * 20;
//   let maxGrassCount = 0, maxHighGrassCount = 0;
//   let maxX, maxY;
//
//   for (let x = 0; x < gameMap.getWidth(); ++x) {
//     for (let y = 0; y < gameMap.getHeight(); ++y) {
//       const {grassCount, highGrassCount} = gameMap.simulateMist(x, y, initialAmount);
//       if (highGrassCount >= maxHighGrassCount) {
//         maxHighGrassCount = highGrassCount;
//         maxGrassCount = grassCount;
//         maxX = x;
//         maxY = y;
//       }
//     }
//   }
//
//   return {x: maxX, y: maxY, highGrassCount: maxHighGrassCount, grassCount: maxGrassCount};
// }

const TILE_SIZE = 20;
/** @type {HTMLCanvasElement} */
const canvas = document.getElementById('canvas');
canvas.width = TILE_SIZE * gameMap.getWidth();
canvas.height = TILE_SIZE * gameMap.getHeight();

const context = canvas.getContext('2d');
context.font = '12px monospace';

const wallTileImage = new Image;
wallTileImage.src = './images/wall-tile.png';

function updateMapDisplay() {
  const wandLevel = parseInt(document.getElementById('wand_level').value);
  if (!Number.isInteger(wandLevel))
    return;

  requestAnimationFrame(() => {
    let maxHighGrassCount = 0;
    let x = 1, y = 2;
    for (let x = 0; x < gameMap.getWidth(); ++x) {
      for (let y = 0; y < gameMap.getHeight(); ++y) {
        if (gameMap.getTileType(x, y) != TileType.WALL) {
          const {grassCount, highGrassCount} = gameMap.simulateMist(x, y, 40 + wandLevel * 20);
          maxHighGrassCount = Math.max(maxHighGrassCount, highGrassCount);

          const hue = 90 - highGrassCount * 10;
          const saturation = 1 - highGrassCount * 0.01;
          const lightness = 0.8 - highGrassCount * 0.05;

          context.fillStyle = `hsl(${hue}, ${saturation * 100}%, ${lightness * 100}%)`;
          context.fillRect(x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
          context.fillStyle = lightness < 0.5 ? '#fff' : '#000';
          context.fillText(highGrassCount, x * TILE_SIZE + (TILE_SIZE * 3 / 16), y * TILE_SIZE + (TILE_SIZE * 3 / 4));
        }
        else {
          // context.fillStyle = `#000`;
          // context.fillRect(x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
          context.drawImage(wallTileImage, x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
        }
      }
    }

    document.getElementById('max_high_grass').value = maxHighGrassCount;
  });
}

canvas.addEventListener('click', event => {
  x = Math.floor(event.offsetX / TILE_SIZE);
  y = Math.floor(event.offsetY / TILE_SIZE);

  gameMap.setTileType(x, y, gameMap.getTileType(x, y) === TileType.WALL ? TileType.FLOOR : TileType.WALL);
  updateMapDisplay();
});

document.getElementById('wand_level').addEventListener('change', event => { updateMapDisplay(); });
wallTileImage.addEventListener('load', () => {
  document.getElementById('wand_level').dispatchEvent(new Event('change'));
});

</script>