<template>
  <div id="app">
    <div>
      Stash:
      <span id="stash" :class="stashed" @click="clickStash">{{ stashed }}</span>
    </div>
    <div id="holding" :class="holding">{{ holding }}</div>
    <div>
      History: {{ turns }} years ...
      Points: {{ points }}
    </div>
    <div class="grid">
      <div
        class="row"
        v-for="(row, i) in grid"
        :key="i"
      >
        <span
          :class="['cell', grid[i][j]]"
          v-for="(cell, j) in row"
          :key="j"
          @click="clickCell(i, j)"
        >
          <span>{{ grid[i][j] }}</span>
        </span>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      gridHeight: 6,
      gridWidth: 6,
      grid: [], // 2D array of cells
      stateInfo: {
        empty: {
          next: '', // doesn't merge to become anything
          points: 0, // hence cannot give points
        },
        grass: {
          next: 'bush',
          points: 5,
        },
        bush: {
          next: 'tree',
          points: 10,
        },
        tree: {
          next: 'hut',
          points: 25,
        },
        hut: {
          next: 'house',
          points: 75,
        },
        house: {
          next: 'mansion',
          points: 250,
        },
        mansion: {
          next: 'castle',
          points: 1000,
        },
        castle: {
          next: 'sky_castle',
          points: 5000,
        },
        sky_castle: {
          next: 'space_station',
          points: 25000,
        },
        destroyer: {},
      },
      holding: 'grass',
      points: 0,
      turns: 0,
      stashed: 'empty',
    };
  },
  created() {
    this.generateGrid();

    document.addEventListener('mousemove', function(e) {
      const holding = document.getElementById('holding');
      holding.style.left = e.clientX + 15 + 'px';
      holding.style.top = e.clientY + 15 + 'px';
    });
  },
  methods: {
    generateGrid() {
      // initialize grid with random cell states
      const grid = []
      for (let i = 0; i < this.gridHeight; i++) {
        const row = []
        for (let j = 0; j < this.gridWidth; j++) {
          let cell = 'empty';
          const r = Math.random();
          if (r > 0.95) cell = 'hut';
          else if (r > 0.85) cell = 'tree';
          else if (r > 0.7) cell = 'bush';
          else if (r > 0.5) cell = 'grass';
          row.push(cell);
        }
        grid.push(row);
      }
      this.grid = grid;

      // Absorb cells into each other to ensure the map starts in a way that could happen naturally in a game.
      // This means, for example, even if we limit random cells to hut or lower, they could still end up merging
      // to a house or more.
      // eslint-disable-next-line no-constant-condition
      while (true) {
        let absorbedAtLeastOnce = false;
        for (let i = 0; i < this.gridHeight; i++) {
          for (let j = 0; j < this.gridWidth; j++) {
            const absorbedThisTime = this.absorbSurroundingSimilarCells(i, j);
            absorbedAtLeastOnce = absorbedAtLeastOnce || absorbedThisTime;
          }
        }
        if (!absorbedAtLeastOnce) break;
      }

      // reset points, as merging during generation may have scored points
      this.points = 0;
    },
    setNextHolding() {
      let state = 'grass';
      const r = Math.random();
      if (r > 0.98) state = 'hut';
      else if (r > 0.92) state = 'tree';
      else if (r > 0.82) state = 'bush';
      else if (r > 0.79) state = 'destroyer';
      this.holding = state;
    },
    clickCell(i, j) {
      if (this.grid[i][j] == 'empty') {
        if (this.holding == 'destroyer') {
          // cannot destroy emptiness; do nothing
          return;
        } else {
          // place something on empty
          this.setCell(i, j, this.holding);
          this.absorbSurroundingSimilarCells(i, j);
        }
      } else { // cell has something
        if (this.holding == 'destroyer') {
          // destroy the thing
          this.setCell(i, j, 'empty');
        } else {
          // can't put something on something; do nothing
          return;
        }
      }

      this.turns++;
      this.setNextHolding();
    },
    // absorbSurroundingSimilarCells absorbs surrounding cells of the given cell,
    // checking adjacent cells and those adjacent cells.
    // Adjacent means directly above, below, left or right of the cell (not diagonal).
    // Returns true if surrounding cells were absorbed.
    absorbSurroundingSimilarCells(i, j) {
      let state = this.grid[i][j];

      // do nothing if cell has a state which does not merge into something
      if (!this.stateInfo[state].next) return;

      // identify all adjacent similar cells...
      let adj = this.getAdjacentMatchingCells(i, j, state);
      // ...and their adjacent cells...
      const adjadj = [];
      for (const [ai, aj] of adj) {
        // ...(only keep adjacent adjacent cells that are not this cell)...
        this.getAdjacentMatchingCells(ai, aj, state).forEach(([aai, aaj]) => {
          if (i != aai || j != aaj) adjadj.push([aai, aaj]);
        });
      }

      // ...if there's 2 or more adjacent (or adjacent adjacent) cells
      // then merge them into this one
      const matchingCells = adj.concat(adjadj);
      if (matchingCells.length >= 2) {
        // reset matching cells
        for (const [i, j] of matchingCells)
          this.setCell(i, j, 'empty');
        
        // upgrade this cell
        this.setCell(i, j, this.stateInfo[state].next);

         // award points
        this.points += this.stateInfo[state].points*this.stateInfo[state].points;

        // might be able to absorb other similar cells now that this cell changed
        this.absorbSurroundingSimilarCells(i, j);

        return true;
      }

      return false;
    },
    getAdjacentMatchingCells(i, j, state) {
      const adjacentMatchingCells = [];
      if (i > 0 && this.grid[i-1][j] == state)
        adjacentMatchingCells.push([i-1,j]);
      if (j > 0 && this.grid[i][j-1] == state)
        adjacentMatchingCells.push([i,j-1]);
      if (i < this.gridHeight-1 && this.grid[i+1][j] == state)
        adjacentMatchingCells.push([i+1,j]);
      if (j < this.gridWidth-1 && this.grid[i][j+1] == state)
         adjacentMatchingCells.push([i,j+1]);
      return adjacentMatchingCells;
    },
    setCell(i, j, state) {
      // make a copy of the row
      const newRow = this.grid[i].slice(0);
      // update the value
      newRow[j] = state;
      // update it in the grid
      this.$set(this.grid, i, newRow);
    },
    clickStash() {
      const stashed = this.stashed;
      this.stashed = this.holding;

      if (stashed == 'empty') {
        // Stash was empty, so we just dropped the first item into it,
        // meaning we need to hold a new random item.
        this.setNextHolding();
      } else {
        // Stash had something, so we just swapped what we were holding.
        this.holding = stashed;
      }
    },
  },  
};
</script>

<style>
body {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  background-color: #184a7b;
  margin-top: 30px;
  color: rgb(233, 135, 135);
  font-weight: bold;
}

.grid {
  display: inline-block;
}

.cell {
  display: inline-flex;
  width: 80px;
  height: 80px;
  background-color: azure;
  border-radius: 10px;
  margin: 2px;

  /* centre inner status value */
  justify-content: center;
  align-items: center;
}

.empty {
  background-color: rgb(31, 169, 31);
}
.grass {
  background-color: green;
}
.bush {
  background-color: rgb(55, 241, 55);
}
.tree {
  background-color: rgb(115, 132, 54);
}
.hut {
  background-color: rgb(115, 12, 12);
}
.house {
  background-color: beige;
}
.mansion {
  background-color: orange;
}
.castle {
  background-color: grey;
}
.sky_castle {
  background-color: darkgrey;
}
.space_station {
  background-color: purple;
}
.destroyer {
  background-color: yellow;
  color: black;
}

#holding {
  position: absolute;
  box-shadow: 5px 3px 20px 2px black;
  padding: 2px 6px;
  border-radius: 5px;
}
</style>
