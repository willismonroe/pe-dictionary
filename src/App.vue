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
        <b-col>
          Proto-Elamite sign values pulled from various academic sources.
        </b-col>
      </b-row>

      <b-pagination
        v-model="currentPage"
        :total-rows="totalRows"
        :per-page="perPage"
        align="fill"
        size="sm"
        class="my-0"
      ></b-pagination>

      <b-table
        striped
        hover
        small
        fixed
        primary-key="index"
        :per-page="perPage"
        :current-page="currentPage"
        :items="table"
        :filter="filter"
        @filtered="onFiltered"
      >
        <template v-slot:cell(imageURL)="data">
          <img :src="data.item.imageURL" />
        </template>
      </b-table>
    </b-container>
    <div v-if="loading" class="loader">
      <b-spinner label="Loading..."></b-spinner>
    </div>
  </div>
</template>

<script>
const URL =
  'https://docs.google.com/spreadsheets/d/e/2PACX-1vSCKaOZPrqm4bIptb6HUtqdBtOyeq0IrJ88sAdD0J0CB0CdpYveWi7iNVHstUP2q2E9Vj_G191nApV1/pub?gid=0&single=true&output=csv'

function convertSignName(signName) {
  signName = signName.toUpperCase()
  return signName.replace(/~/g, '-')
}

function parseData(csv) {
  const lines = csv.split('\n')
  const headers = lines[0].split(',')
  const body = []
  for (let i = 1; i < lines.length; i++) {
    let row = {}
    let currentLine = lines[i].split(',')
    for (let j = 0; j < headers.length; j++) {
      row['index'] = i
      row[headers[j]] = currentLine[j]
      try {
        row['imageURL'] = require(`./assets/signs/${convertSignName(
          row['Sign (Dahl)'],
        )}.png`)
      } catch (e) {
        // console.log(e)
        row['imageURL'] = require('./assets/signs/empty.png')
      }
    }
    body.push(row)
  }
  return body
}

export default {
  name: 'App',
  data() {
    return {
      table: [],
      loading: true,
      filterInput: '',
      filter: null,
      perPage: 25,
      currentPage: 1,
      totalRows: 1,
    }
  },
  created() {
    fetch(URL)
      .then((response) => {
        return response.text()
      })
      .then((res) => {
        this.table = parseData(res)
        this.loading = false
        this.totalRows = this.table.length
      })
  },
  methods: {
    setFilter() {
      this.loading = true
      setTimeout(() => {
        this.filter = this.filterInput
      })
    },
    clearFilter() {
      this.filter = ''
      this.filterInput = ''
    },
    onFiltered(filteredItems) {
      this.totalRows = filteredItems.length
      this.currentPage = 1
      this.loading = false
    },
  },
}
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
</style>
