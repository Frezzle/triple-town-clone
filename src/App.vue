<template>
  <div id="app">
    <div>History: {{ turns }} years ... Points: {{ points }} ... Holding {{ states[holding] }}</div>
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
      grid: [
        [0,0,0,0,0,0],
        [0,0,0,0,0,0],
        [0,0,0,0,0,0],
        [0,0,0,0,0,0],
        [0,0,0,0,0,0],
        [0,0,0,0,0,0],
      ],
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
  methods: {
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
    },
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
      }
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
