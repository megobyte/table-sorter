<template lang="pug">
  .table-sorter
    .pagination
      span Страница {{page}} из {{Math.ceil(rows.length / onpage_prop)}} (Всего {{rows.length}} элементов)
      .btn(:class="{ disabled: (page == 1) }", @click="page = 1") &#9668;&#9668;	
      .btn(:class="{ disabled: (page == 1) }", @click="page -= 1") &#9668;
      .btn(:class="{ disabled: (page == Math.ceil(rows.length / onpage_prop)) }", @click="page += 1") &#9658;
      .btn(:class="{ disabled: (page == Math.ceil(rows.length / onpage_prop)) }", @click="page = Math.ceil(rows.length / onpage_prop)") &#9658;&#9658;	
    .table
      .cols
        template(v-for="(col, idx) in cols")
          .col(v-if="col.visible")
            .title {{col.title}}
            .filter
              input(v-model="cols[idx].filter")
            .rows
              template(v-for="(row, ridx) in getRows(col)")
                .row {{row}}
</template>
<script>
export default {
  props: {
    table: {
      type: [Object, Array],
      required: true
    },
    onpage: {
      type: Number,
      default: 10
    }
  },
  data() {
    return {
      cols: [],
      rows: [],
      filter: '',
      page: 1,
      onpage_prop: this.onpage
    }
  },
  watch: {
    table: function() {
      this.load();
    }
  },
  methods: {
    getRows(col) {
      var rows = [];
      var from_idx = (this.page - 1) * this.onpage_prop;
      var to_idx = from_idx + this.onpage_prop;
      for (var i = from_idx; i < to_idx; i++) {
        rows.push(this.rows[i][col.title]);
      }
      return rows;
    },

    load() {
      if ( (this.table !== []) && (this.table !== {}) ) { // check is table not empty
        if (typeof this.table.cols !== "undefined") { // check table is object
          this.table.cols.forEach((el) => {
            var col = {
              title: el,
              order: this.cols.length,
              visible: true,
              filter: ''
            };
            this.cols.push(col);
          });
          this.table.data.forEach((row) => {
            var r = {};
            this.cols.forEach((col) => {
              r[col.title] = row[col.order];
            });
            this.rows.push(r);
          });
        }
      }
    }
  },
  mounted: function() {
    this.load();
  }
}
</script>
<style lang="scss" scoped>
.table-sorter {
  width: 100%;

  .pagination {
    display: flex;
    justify-content: flex-start;
    align-items: center;

    .btn {
      margin-left: 10px;
      font-family: initial;
      display: block;
      width: 30px;
      height:30px;
      display: flex;
      justify-content: center;
      align-items: center;
      border: 1px solid gray;
      border-radius: 50%;
      font-size: 10px;
      cursor: pointer;

      &.disabled,
      &:disabled {
        opacity: .2;
        pointer-events: none;
      }
    }
  }

  .table {
    width:100%;

    .cols {
      width: 100%;
      display: flex;
      flex-direction: row;
      justify-content: flex-start;

      .col {
        text-align: left;
        border-left: 1px solid gray;

        .title {
          font-weight: bold;
          overflow: hidden;
          width: 100%;
          border-bottom: 1px solid gray;
          padding: 3px;
        }

        .filter {
          width: 100%;
          border-bottom: 1px solid gray;
          padding: 3px;
        }

        .rows {
          .row {
            width: 100%;
            padding: 3px;
            border-bottom: 1px solid #eee;
          }
        }
      }
    }
  }
}
</style>
