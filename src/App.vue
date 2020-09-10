<template>
  <div id="app">
    <b-container fluid>
      <b-row>
        <b-col lg="4" class="my-3">
          <b-form-group
              label="Filter"
              label-cols-sm="3"
              label-align-sm="right"
              label-size="sm"
              label-for="filterInput"
              class="mb-0"
          >
            <b-input-group size="sm">
              <b-form-input
                  v-model="filterInput"
                  type="text"
                  id="filterInput"
                  placeholder="Type to filter, Enter to submit"
                  v-on:keyup.enter="setFilter"
              ></b-form-input>
              <b-input-group-append>
                <b-button :disabled="!filterInput" @click="setFilter">
                  Filter
                </b-button>
                <b-button :disabled="!filterInput" @click="clearFilter">
                  Clear
                </b-button>
              </b-input-group-append>
            </b-input-group>
          </b-form-group>
        </b-col>
        <b-col class="my-3">
          Proto-Elamite sign values pulled from various academic sources into a
          Google Sheet.
        </b-col>
        <b-col>
          <canvas
              @mousedown="startPainting"
              @mouseup="finishedPainting"
              @mousemove="draw"
              id="canvas"
          ></canvas>
          <br/>
          <b-row>
            <b-col>
              <b-button @click="searchDrawing" variant="success"
              >Search
              </b-button
              >
            </b-col>
            <b-col>
              <b-button @click="clearDrawing" variant="danger">Clear</b-button>
            </b-col>
          </b-row>
        </b-col>
      </b-row>

      <b-pagination
          v-model="currentPage"
          :total-rows="totalRows"
          :per-page="perPage"
          align="fill"
          size="sm"
          class="my-2"
      ></b-pagination>

      <b-table
          striped
          hover
          small
          fixed
          primary-key="index"
          :fields="fields"
          :per-page="perPage"
          :current-page="currentPage"
          :items="table"
          :filter="filter"
          :sort-desc="sortDesc"
          :sort-by.sync="sortBy"
          @filtered="onFiltered"
      >
        <template v-slot:cell(imageURL)="data">
          <img :src="data.item.imageURL"/>
        </template>
      </b-table>
    </b-container>
    <div v-if="loading" class="loader">
      <b-spinner label="Loading..."></b-spinner>
    </div>
  </div>
</template>

<script>
import pixelmatch from "pixelmatch";

const URL =
    "https://spreadsheets.google.com/feeds/list/1nvGgBh8s2qOBRIQVigJCH3AqW7Wwx1k7xn5LQDhY_Hk/od6/public/values?alt=json";

function convertSignName(signName) {
  signName = signName.toUpperCase();
  return signName.replace(/~/g, "-");
}

function parseJSON(data) {
  let rows = [];
  for (let i = 0; i < data.feed.entry.length; i++) {
    let entry = data.feed.entry[i];
    let keys = Object.keys(entry);
    let newRow = {};
    newRow.index = i;
    keys.forEach((key) => {
      if (key.indexOf("gsx$") > -1) {
        let name = key.substring(4);
        let value = entry[key].$t;
        newRow[name] = value;
      }
    });
    try {
      newRow["imageURL"] = require(`./assets/signs/${convertSignName(
          newRow["signdahl"]
      )}.png`);
    } catch (e) {
      // console.log(e)
      newRow["imageURL"] = require("./assets/signs/empty.png");
    }
    rows.push(newRow);
  }
  return rows;
}

/* eslint-disable */
function parseData(csv) {
  const lines = csv.split("\n");
  const headers = lines[0].split(",");
  const body = [];
  for (let i = 1; i < lines.length; i++) {
    let row = {};
    let currentLine = lines[i].split(",");
    for (let j = 0; j < headers.length; j++) {
      row["index"] = i;
      row[headers[j]] = currentLine[j];
      try {
        row["imageURL"] = require(`./assets/signs/${convertSignName(
            row["Sign (Dahl)"]
        )}.png`);
      } catch (e) {
        // console.log(e)
        row["imageURL"] = require("./assets/signs/empty.png");
      }
    }
    body.push(row);
  }
  return body;
}

async function imageToUint8Array(image, context) {
  return new Promise((resolve, reject) => {
    context.width = 108;
    context.height = 108;
    context.drawImage(image, 0, 0);
    context.canvas.toBlob(blob => blob.arrayBuffer()
        .then(buffer => resolve(new Uint8Array(buffer))).catch(reject)
    )
  });
}

export default {
  name: "App",
  data() {
    return {
      fullTable: [],
      table: [],
      fields: ["imageURL", "signdahl", "proposedmeaning", "comments", { key: "diff", sortable: true}],
      sortDesc: false,
      sortBy: 'signdahl',
      loading: true,
      filterInput: "",
      filter: null,
      perPage: 25,
      currentPage: 1,
      totalRows: 1,
      vueCanvas: null,
      painting: false,
      canvas: null,
      ctx: null,
    };
  },
  created() {
    fetch(URL)
        .then((response) => {
          return response.json();
        })
        .then((data) => {
          this.table = parseJSON(data);
          this.fullTable = Array.from(this.table)
          this.loading = false;
          this.totalRows = this.table.length;
        });
  },
  methods: {
    setFilter() {
      this.loading = true;
      setTimeout(() => {
        this.filter = this.filterInput;
      });
    },
    clearFilter() {
      this.filter = "";
      this.filterInput = "";
    },
    onFiltered(filteredItems) {
      this.totalRows = filteredItems.length;
      this.currentPage = 1;
      this.loading = false;
    },
    startPainting(e) {
      this.painting = true;
      this.draw(e);
    },
    finishedPainting() {
      this.painting = false;
      this.ctx.beginPath();
    },
    draw(e) {
      if (!this.painting) return;

      this.ctx.lineWidth = 4;
      this.ctx.lineCap = "round";
      this.ctx.fillStyle = "#000000";
      this.ctx.lineTo(e.offsetX, e.offsetY);
      this.ctx.stroke();

      this.ctx.beginPath();
      this.ctx.moveTo(e.offsetX, e.offsetY);
    },
    clearDrawing() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.sortDesc = false
      this.sortBy = 'signdahl'
      this.table.forEach((item) => item.diff = '')
    },
    async searchDrawing() {
      this.loading = true;
      const drawingPixelData = this.ctx.getImageData(0, 0, 108, 108);

      const diffCanvas = document.createElement("canvas");
      diffCanvas.width = 108
      diffCanvas.height = 108
      const diffContext = diffCanvas.getContext("2d");
      const diffImage = diffContext.createImageData(108, 108);

      function afterImageLoad(src) {
        return new Promise((resolve, reject) => {
          const signImage = new Image(108, 108);

          signImage.onload = async () => {
            const signCanvas = document.createElement('canvas');
            const signContext = signCanvas.getContext("2d")
            signContext.drawImage(signImage, 0, 0, 108, 108)
            resolve(signContext.getImageData(0, 0, 108, 108));
          }
          signImage.src = src;
        })
      }

      for (let i = 0; i < this.table.length; i++) {

        let signImageSrc = this.table[i].imageURL;
        let signImageData = await afterImageLoad(signImageSrc)

        let diff = pixelmatch(
            drawingPixelData.data,
            signImageData.data,
            diffContext.data,
            108,
            108,
        );
        this.table[i].diff = diff
      }
      this.loading = false
      this.sortBy = 'diff'
    },
  },
  mounted() {
    this.canvas = document.getElementById("canvas");
    this.ctx = this.canvas.getContext("2d");

    // Resize canvas
    this.canvas.height = 108;
    this.canvas.width = 108;
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.loader {
  align-items: center;
  background: rgb(23, 22, 22);
  opacity: 0.3;
  display: flex;
  height: 100vh;
  justify-content: center;
  left: 0;
  position: fixed;
  top: 0;
  transition: opacity 0.3s linear;
  width: 100%;
  z-index: 9999;
}

#canvas {
  border: 3px solid black;
}
</style>
