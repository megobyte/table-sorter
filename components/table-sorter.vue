<template lang="pug">
  .table-sorter
    .pagination
      span Страница {{page}} из {{Math.ceil(rows_filtered.length / onpage_prop)}} (Всего {{rows_filtered.length}} элементов)
      .btn(:class="{ disabled: (page == 1) }", @click="page = 1") &#9668;&#9668;	
      .btn(:class="{ disabled: (page == 1) }", @click="page -= 1") &#9668;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page += 1") &#9658;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page = Math.ceil(rows_filtered.length / onpage_prop)") &#9658;&#9658;	
    .filter
      input(v-model="filter", placeholder="Введите текст для поиска")
      .btn.clear(v-if="filter !== ''", @click="filter = ''")
    .table
      .cols
        template(v-for="(col, idx) in cols")
          .col(v-if="col.visible")
            .title {{col.title}}
            .filter
              input(v-model="cols[idx].filter")
              .btn.clear(v-if="cols[idx].filter !== ''", @click="cols[idx].filter = ''")
            .rows
              template(v-for="(row, ridx) in getRows(col)")
                .row(v-html="highlight(row, col)")
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

  computed: {
    rows_filtered: function() {
      var rows = [];

      // global filter
      if (this.filter !== "") {
        this.rows.forEach( (row) => {
          this.cols.every( (col) => {
            if ((row[col.title]+"").toLowerCase().indexOf(this.filter.toLowerCase()) > -1) {
              rows.push(row);
              return false;
            } else {
              return true;
            }
          });
        });
      } else {
        rows = this.rows;
      }

      // column filter
      this.cols.forEach( (col) => {
        if (col.filter !== '') {
          var _rows = [];
          rows.forEach( (row) => {
            if ((row[col.title]+"").toLowerCase().indexOf(col.filter.toLowerCase()) > -1) {
              _rows.push(row);
            }
          });

          rows = _rows;
        }
      });

      return rows;
    }
  },

  watch: {
    table: function() {
      this.load();
    }
  },

  methods: {

    highlight(text, col) {
      String.prototype.splice = function(idx, rem, str) {
        return this.slice(0, idx) + str + this.slice(idx + Math.abs(rem));
      };

      if (this.filter !== "") { // global filter
        var s_i = (text+'').toLowerCase().indexOf(this.filter.toLowerCase());
        var e_i = s_i + this.filter.length;
        if (s_i > -1) {
          text = (text+'').splice(e_i, 0, '</span>');
          text = (text+'').splice(s_i, 0, '<span class="globalhl">');
        }
      }

      if (col.filter !== "") { // column filter
        var s_i = (text+'').toLowerCase().indexOf(col.filter.toLowerCase());
        var e_i = s_i + col.filter.length;
        if (s_i > -1) {
          text = (text+'').splice(e_i, 0, '</span>');
          text = (text+'').splice(s_i, 0, '<span class="localhl">');
        }
      }

      return text;
    },

    getRows(col) {
      var rows = [];
      var from_idx = (this.page - 1) * this.onpage_prop;
      var to_idx = from_idx + this.onpage_prop;
      if (to_idx > this.rows_filtered.length) to_idx = this.rows_filtered.length;
      for (var i = from_idx; i < to_idx; i++) {
        rows.push(this.rows_filtered[i][col.title]);
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
          this.table.data.forEach( (row) => {
            var r = {};
            this.cols.forEach( (col) => {
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
<style lang="scss">
.table-sorter {
  width: 100%;
  padding: 10px;

  $gray: #aaa;

  .globalhl {
    color: #01398e;
    text-decoration: underline;
  }
  .localhl {
    color: #488e01;
    text-decoration: underline;
  }

  .clear {
    &::after {
      content: '';
      display: block;
      background-image: url(data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz48IURPQ1RZUEUgc3ZnIFBVQkxJQyAiLS8vVzNDLy9EVEQgU1ZHIDEuMS8vRU4iICJodHRwOi8vd3d3LnczLm9yZy9HcmFwaGljcy9TVkcvMS4xL0RURC9zdmcxMS5kdGQiPjxzdmcgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxuczp4bGluaz0iaHR0cDovL3d3dy53My5vcmcvMTk5OS94bGluayIgdmVyc2lvbj0iMS4xIiB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTE5LDYuNDFMMTcuNTksNUwxMiwxMC41OUw2LjQxLDVMNSw2LjQxTDEwLjU5LDEyTDUsMTcuNTlMNi40MSwxOUwxMiwxMy40MUwxNy41OSwxOUwxOSwxNy41OUwxMy40MSwxMkwxOSw2LjQxWiIgLz48L3N2Zz4=);
      background-position: center;
      background-repeat: no-repeat;
      background-size: contain;
      width: 60%;
      height: 60%;
    }
  }

  input {
    height: 25px;
    border-radius: 12.5px;
    -webkit-appearance: none;
    border: 1px solid $gray;
    padding-left: 13px;
  }

  .btn {
    cursor: pointer;
  }

  .pagination {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    margin-bottom: 10px;

    .btn {
      margin-left: 10px;
      font-family: initial;
      width: 30px;
      height:30px;
      display: flex;
      justify-content: center;
      align-items: center;
      border: 1px solid $gray;
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

  .filter {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    margin-bottom: 10px;

    input {
      width: 300px;
    }

    .btn {
      width: 25px;
      height: 25px;
      border-radius: 50%;
      border: 1px solid $gray;
      display: flex;
      justify-content: center;
      align-items: center;
      margin-left: 10px;
    }
  }

  .table {

    .cols {
      width: 100%;
      display: flex;
      flex-direction: row;
      justify-content: flex-start;

      .col {
        text-align: left;
        border-left: 1px solid $gray;
        border-top: 1px solid $gray;
        border-bottom: 1px solid $gray;

        &:nth-last-child(1) { border-right: 1px solid $gray; }

        .title {
          font-weight: bold;
          overflow: hidden;
          width: 100%;
          border-bottom: 1px solid $gray;
          padding: 3px;
        }

        .filter {
          width: 100%;
          border-bottom: 1px solid $gray;
          padding: 3px;
          position: relative;

          input {
            width: 100%;
            padding-right: 24px;
          }

          .btn {
            position: absolute;
            right: 6px;
            width: 20px;
            height: 20px;
            border: none;
          }
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
