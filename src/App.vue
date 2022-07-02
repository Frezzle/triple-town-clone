<template>
  <div id="app">
    <div>
      History: {{ turns }} years ...
      Points: {{ points }} ... 
      Holding <span :class="[states[holding]]">{{ states[holding] }}</span>
    </div>
    <div class="grid">
      <div
        class="row"
        v-for="(row, i) in grid"
        :key="i"
      >
        <span
          :class="['cell', getStateClass(i, j)]"
          v-for="(cell, j) in row"
          :key="j"
          @click="click(i, j)"
        >
          <span>{{ getStateClass(i, j) }}</span>
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
      states: [
        'empty',
        'grass',
        'bush',
        'tree',
        'hut',
        'house',
        'mansion',
        'castle',
        'sky_castle',
      ],
      holding: 1, // grass
      points: 0,
      turns: 0,
    };
  },
  created() {
    this.generateGrid();
  },
  methods: {
    generateGrid() {
      // initialize grid with random cell states
      const grid = []
      for (let i = 0; i < this.gridHeight; i++) {
        const row = []
        for (let j = 0; j < this.gridWidth; j++) {
          let cell = 0; // empty
          const r = Math.random();
          if (r > 0.95) cell = 4; // hut
          else if (r > 0.85) cell = 3; // tree
          else if (r > 0.7) cell = 2; // bush
          else if (r > 0.5) cell = 1; // grass
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
    },
    getState(i, j) {
      return this.grid[i][j];
    },
    getStateClass(i, j) {
      const stateIndex = this.getState(i, j);
      return this.states[stateIndex];
    },
    click(i, j) {
      if (this.getStateClass(i, j) != 'empty') return;
      this.turns++;
      this.setCell(i, j, this.holding);

      this.absorbSurroundingSimilarCells(i, j);

      // hold a potentially-different item next
      let state = 1; // grass
      const r = Math.random();
      if (r > 0.98) state = 4; // hut
      else if (r > 0.92) state = 3; // tree
      else if (r > 0.82) state = 2; // bush
      this.holding = state;
    },
    // absorbSurroundingSimilarCells absorbs surrounding cells of the given cell,
    // checking adjacent cells and those adjacent cells.
    // Adjacent means directly above, below, left or right of the cell (not diagonal).
    // Returns true if surrounding cells were absorbed.
    absorbSurroundingSimilarCells(i, j) {
      let state = this.grid[i][j];

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
        // reset adjacent cells
        for (const [i, j] of matchingCells)
          this.setCell(i, j, 0);
        
        // upgrade this cell
        state++;
        this.setCell(i, j, state);

         // award points
        this.points += state*state;

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
      if (i < this.grid.length-1 && this.grid[i+1][j] == state)
        adjacentMatchingCells.push([i+1,j]);
      if (j < this.grid[0].length-1 && this.grid[i][j+1] == state)
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
</style>
