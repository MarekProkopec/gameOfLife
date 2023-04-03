<template>
  <main>
    <div class="input">
      <label for="columnCount">Size of grid: {{ gridSize }}</label>
      <input
        type="range"
        name="columnCount"
        id="columnCount"
        v-model="gridSize"
        min="5"
        max="100"
        @input="createTable"
      />
      <label for="columnCount">Speed: {{ speed }}</label>
      <input
        type="range"
        name="simulationSpeed"
        id="simulationSpeed"
        v-model="speed"
        min="10"
        max="2000"
        step="10"
        @input="createTable"
      />

      <label for="dostroyOnHit">Destroy on hitting wall:</label>
      <input
        type="checkbox"
        name="dostroyOnHit"
        id="dostroyOnHit"
        v-model="dostroyOnHit"
      />

      <div class="buttons">
        <button class="start" v-if="!running" @click="start">Start</button>
        <button class="pause" v-else @click="pause">Pause</button>
        <button class="step" @click="update">Step</button>
        <button class="clear" @click="grid = createClearGrid()">Clear</button>
      </div>
    </div>

    <div class="gridContainer" @mousewheel="controlZoom">
      <div
        class="grid"
        ref="grid"
        @panzoomstart="startDrag"
        @panzoomend="endDrag"
      >
        <div class="row" v-for="(row, y) in grid" :key="y">
          <div
            class="cell"
            v-for="(cell, x) in row"
            :key="x"
            @click="changeCell(x, y)"
            @dblclick="lastSelected = [x, y]"
            :class="[grid[y][x] ? 'active' : '']"
          >
            <p v-if="zoom > 1">
              {{ getNumberOfNeighbours(x, y) }}
            </p>
          </div>
        </div>
      </div>
    </div>
  </main>
  <aside>
    <p>Spawn on: <br />x:{{ lastSelected[0] }}, y:{{ lastSelected[1] }}</p>
    <h3>Templates:</h3>
    <div class="templates">
      <div
        class="template"
        v-for="(item, index) in templates"
        :key="index"
        @click="spawnTemplate(index)"
      >
        <div class="info">
          <h3 class="name">{{ item["name"] }}</h3>
          <h5 class="type">{{ item["type"] }}</h5>
        </div>
        <canvas
          :ref="setItemRef"
          :draw="draw(index)"
          :height="item['template'].length * canvasRes"
          :width="getWidth(item['template']) * canvasRes"
        ></canvas>
      </div>
    </div>
  </aside>
</template>

<script>
import Panzoom from "@panzoom/panzoom";
import { templates } from "./js/templates.js";


export default {
  name: "App",
  data() {
    return {
      canvasRes: 20,
      canvasEls: [],
      grid: Array,
      gridSize: Number,
      checkSpaces: [
        [-1, -1],
        [-1, 0],
        [-1, 1],
        [0, -1],
        [0, 1],
        [1, -1],
        [1, 0],
        [1, 1],
      ],
      interval: null,
      speed: 500,
      running: false,
      dragging: false,
      zoom: 1,
      dostroyOnHit: false,
      lastSelected: [0, 0],
      templates: templates,
    };
  },
  methods: {
    changeCell(x, y) {
      if (this.dragging) return;
      this.grid[y][x] = !this.grid[y][x];
    },
    createTable() {
      let newGrid = [];
      for (let y = 0; y < this.gridSize; y++) {
        let temp = [];
        for (let x = 0; x < this.gridSize; x++) {
          try {
            temp.push(this.grid[y][x]);
          } catch (error) {
            temp.push(false);
          }
        }
        newGrid.push(temp);
      }
      this.grid = newGrid;
      if (this.lastSelected.find((e) => e > this.gridSize - 1))
        this.lastSelected = [1, 1];
    },
    startDrag() {
      setTimeout(() => {
        this.dragging = true;
      }, 200);
    },
    endDrag() {
      setTimeout(() => {
        this.dragging = false;
      }, 210);
    },
    getNumberOfNeighbours(x, y) {
      let ans = 0;

      for (let i of this.checkSpaces) {
        let targetX = x + i[0];
        let targetY = y + i[1];

        if (!this.dostroyOnHit) {
          if (targetX > this.gridSize - 1) targetX = 0;
          if (targetX < 0) targetX = this.gridSize - 1;

          if (targetY > this.gridSize - 1) targetY = 0;
          if (targetY < 0) targetY = this.gridSize - 1;
        }

        try {
          if (this.grid[targetY][targetX]) {
            ans += 1;
            continue;
          }
        } catch {
          continue;
        }
      }
      return ans;
    },
    createClearGrid() {
      let newGrid = [];
      for (let y = 0; y < this.gridSize; y++) {
        let temp = [];
        for (let x = 0; x < this.gridSize; x++) {
          temp.push(false);
        }
        newGrid.push(temp);
      }
      return newGrid;
    },
    update() {
      let newGrid = this.createClearGrid();

      for (let y = 0; y < this.grid.length; y++) {
        for (let x = 0; x < this.grid.length; x++) {
          const alive = this.grid[y][x];
          const nb = this.getNumberOfNeighbours(x, y);

          if (alive) {
            if (nb < 2 || nb > 3) {
              newGrid[y][x] = false;
              continue;
            }
            newGrid[y][x] = true;
            continue;
          }
          if (nb == 3) {
            newGrid[y][x] = true;
            continue;
          }
          newGrid[y][x] = false;
        }
      }
      this.grid = newGrid;
    },
    start() {
      this.running = true;
      this.interval = setInterval(() => {
        this.update();
      }, this.speed);
    },
    pause() {
      clearInterval(this.interval);
      this.running = false;
    },
    controlZoom(e) {
      e.deltaY > 0 ? this.panzoom.zoomOut() : this.panzoom.zoomIn();
      this.zoom = this.panzoom.getScale();
    },
    getWidth(arr) {
      let max = 0;
      for (let item of arr) {
        if (item.length > max) max = item.length;
      }
      return max;
    },
    debug(e) {
      console.log(e);
    },
    draw(index) {
      setTimeout(() => {
        const canvas = this.canvasEls[index];
        const template = this.templates[index]["template"];
        const ctx = canvas.getContext("2d");
        ctx.fillStyle = "rgb(0, 0, 0)";

        for (let y = 0; y < template.length; y++) {
          for (let x = 0; x < template[y].length; x++) {
            if (template[y][x])
              ctx.fillRect(
                x * this.canvasRes,
                y * this.canvasRes,
                this.canvasRes,
                this.canvasRes
              );
          }
        }
      }, 1);
    },
    setItemRef(el) {
      if (el) {
        this.canvasEls.push(el);
      }
    },
    spawnTemplate(index) {
      const t = this.templates[index].template;

      for (let y = 0; y < t.length; y++) {
        for (let x = 0; x < t[y].length; x++) {
          if (!this.dostroyOnHit) {
            if (t[y][x]) {
              this.grid[(this.lastSelected[1] + y) % this.gridSize][
                (this.lastSelected[0] + x) % this.gridSize
              ] = true;
            }
          } else {
            if (t[y][x]) {
              if (
                this.lastSelected[1] + y > this.gridSize - 1 ||
                this.lastSelected[0] + x > this.gridSize - 1
              )
                continue;
              this.grid[(this.lastSelected[1] + y) % this.gridSize][
                (this.lastSelected[0] + x) % this.gridSize
              ] = true;
            }
          }
        }
      }
    },
  },
  created() {
    this.gridSize = 10;
    this.createTable();
  },
  beforeUpdate() {
    this.canvasEls = [];
  },
  mounted() {
    this.panzoom = Panzoom(this.$refs.grid, {
      maxScale: 2,
      minScale: 0.25,
      animate: true,
    });
  },
};
</script>

<style lang="scss">
$gridSize: Min(80vh, 80vw);
$cellSize: 20px;

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body,
html,
#app {
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;

  display: flex;
  align-items: center;
  justify-content: space-evenly;
  flex-direction: row;

  main {
    .input {
      display: flex;
      flex-direction: column;
      width: $gridSize;

      .buttons {
        display: flex;
        width: 100%;
        justify-content: space-evenly;
        align-items: stretch;
        margin-top: 5px;

        button {
          padding: 4px 8px;
          font-weight: bold;
          border-radius: 5px;
          min-width: 80px;

          &.start {
            background: green;
            color: white;
          }
          &.pause {
            background: red;
            color: white;
          }
        }
      }
    }
    .gridContainer {
      border: solid 5px black;
      display: flex;
      align-items: center;
      justify-content: center;
      height: $gridSize;
      width: $gridSize;

      .grid {
        display: flex;
        flex-direction: column;
        height: fit-content;
        width: fit-content;

        .row {
          display: flex;
          height: 100%;
          width: 100%;

          .cell {
            border: solid 1px black;
            height: $cellSize;
            width: $cellSize;

            display: flex;
            align-items: center;
            justify-content: center;

            &.active {
              background: black;
              color: white;
            }
          }
        }
      }
    }
  }

  aside {
    height: 70%;
    box-shadow: black 0px 0px 0 4px;
    width: 20%;
    border-radius: 15px;

    display: flex;
    flex-direction: column;
    overflow-y: auto;
    padding: 5px;

    & > h3 {
      border-bottom: solid 2px rgba(0, 0, 0, 0.2);
      padding-bottom: 5px;
      margin-bottom: 10px;
    }

    .templates {
      .template {
        height: 150px;
        width: 100%;
        border: solid 2px black;
        border-radius: 10px;
        padding: 5px;
        margin: 5px 0;
        transition: 0.1s linear box-shadow;
        cursor: pointer;

        display: grid;
        grid-template-columns: max-content auto;
        gap: 10px;
        overflow: auto;

        &:hover {
          box-shadow: 0 0 0 2px black;
        }
        .info {
          align-self: flex-start;
          display: flex;
          flex-direction: column;
          align-items: flex-start;
          justify-content: center;

          h3 {
            overflow-wrap: break-word;
            max-width: min-content;
          }

          h5 {
            color: rgba(0, 0, 0, 0.8);
            font-weight: normal;
          }
        }

        canvas {
          border: solid 2px rgba(0, 0, 0, 0.404);
          max-width: max-content;
          max-height: max-content;
          height: 100%;
          justify-self: flex-end;
        }
      }
    }
  }
}
</style>
