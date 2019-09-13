<template lang="pug">
  .table-sorter(@mousemove="doDrag")
    .pagination
      span Страница {{page}} из {{Math.ceil(rows_filtered.length / onpage_prop)}} (Всего {{rows_filtered.length}} элементов)
      .btn(:class="{ disabled: (page == 1) }", @click="page = 1") &#9668;&#9668;
      .btn(:class="{ disabled: (page == 1) }", @click="page -= 1") &#9668;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page += 1") &#9658;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page = Math.ceil(rows_filtered.length / onpage_prop)") &#9658;&#9658;
      .onpage-select
        | Записей на странице:&nbsp;
        select(v-model="onpage_prop")
          template(v-for="p in onpageSelect")
            option(:value="p") {{p}}
    .filter
      input(type="text", v-model="filter", placeholder="Введите текст для поиска")
      .btn.clear(v-if="filter !== ''", @click="filter = ''")
      .btn.options(@click="popup = true")
    .table
      .cols
        template(v-for="(col, idx) in cols")
          .col(v-if="col.visible", :class="{dragging: col.dragging}", :ref="col.title")
            .title(@mousedown="startDrag($event, cols[idx])") {{col.title}}
            .filter
              input(type="text", v-model="cols[idx].filter", @change="page = 1", @keyup="page = 1")
              .btn.clear(v-if="cols[idx].filter !== ''", @click="cols[idx].filter = ''")
            .rows
              template(v-for="(row, ridx) in getRows(col)")
                .row(v-html="highlight(row, col)")
    .pagination
      span Страница {{page}} из {{Math.ceil(rows_filtered.length / onpage_prop)}} (Всего {{rows_filtered.length}} элементов)
      .btn(:class="{ disabled: (page == 1) }", @click="page = 1") &#9668;&#9668;
      .btn(:class="{ disabled: (page == 1) }", @click="page -= 1") &#9668;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page += 1") &#9658;
      .btn(:class="{ disabled: (page == Math.ceil(rows_filtered.length / onpage_prop)) }", @click="page = Math.ceil(rows_filtered.length / onpage_prop)") &#9658;&#9658;
      .onpage-select
        | Записей на странице:&nbsp;
        select(v-model="onpage_prop")
          template(v-for="p in onpageSelect")
            option(:value="p") {{p}}
    .popup(:class="{ active: popup }")
      .overlay(@click="popup = false")
      .window
        .close(@click="popup = false") &times;
        .content
          | Видимость столбцов:
          .items
            template(v-for="(col, idx) in cols")
              label.item(@click="updateSettings()")
                input(type="checkbox", v-model="cols[idx].visible")
                .title {{col.title}}
</template>
<script>
// eslint-disable-next-line
String.prototype.splice = function(idx, rem, str) {
  // резалка текста, чтобы добавлять теги, если есть фильтр
  return this.slice(0, idx) + str + this.slice(idx + Math.abs(rem))
}

if (!window.localStorage) {
  // если нет localStorage, эмулируем с помощью куков (https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API/Local_storage)
  window.localStorage = {
    getItem(sKey) {
      if (!sKey || !this.hasOwnProperty(sKey)) {
        return null
      }
      return unescape(
        document.cookie.replace(
          new RegExp(
            '(?:^|.*;\\s*)' +
              // eslint-disable-next-line
              escape(sKey).replace(/[\-\.\+\*]/g, '\\$&') +
              '\\s*\\=\\s*((?:[^;](?!;))*[^;]?).*'
          ),
          '$1'
        )
      )
    },
    key(nKeyId) {
      return unescape(
        document.cookie
          // eslint-disable-next-line
          .replace(/\s*\=(?:.(?!;))*$/, '')
          // eslint-disable-next-line
          .split(/\s*\=(?:[^;](?!;))*[^;]?;\s*/)[nKeyId]
      )
    },
    setItem(sKey, sValue) {
      if (!sKey) {
        return
      }
      document.cookie =
        escape(sKey) +
        '=' +
        escape(sValue) +
        '; expires=Tue, 19 Jan 2038 03:14:07 GMT; path=/'
      // eslint-disable-next-line
      this.length = document.cookie.match(/\=/g).length
    },
    length: 0,
    removeItem(sKey) {
      if (!sKey || !this.hasOwnProperty(sKey)) {
        return
      }
      document.cookie =
        escape(sKey) + '=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/'
      this.length--
    },
    hasOwnProperty(sKey) {
      return new RegExp(
        // eslint-disable-next-line
        '(?:^|;\\s*)' + escape(sKey).replace(/[\-\.\+\*]/g, '\\$&') + '\\s*\\='
      ).test(document.cookie)
    }
  }
  window.localStorage.length = (
    document.cookie.match(/\=/g) || window.localStorage // eslint-disable-line
  ).length
}

export default {
  name: 'TableSorter',
  props: {
    table: {
      // таблица с данными. Либо объект с содержимым cols: Array of strings и data: Array of strings, либо массив с объектами col: value
      type: [Object, Array],
      required: true
    },
    onpage: {
      // сколько записей показывать на странице
      type: Number,
      default: 10
    },
    onpageSelect: {
      // массив значений выпадающего меню выбора количества записей на странице
      type: Array,
      default() {
        return [10, 20, 50, 100]
      }
    }
  },

  data() {
    return {
      cols: [], // колонки
      rows: [], // данные
      filter: '', // глобальный фильтр
      page: 1, // страница, которая отображается
      onpage_prop: this.onpage, // сколько записей показывать
      key: '', // ключ для считывания натроек из localStorage
      popup: false, // отображение окна видимости столбцов

      dragging: false,
      col: '',
      xpos: 0
    }
  },

  computed: {
    rows_filtered() {
      // отдает отфильтрованные данные, чтобы можно было посчитать кол-во элементов и страниц
      let rows = []

      // global filter
      if (this.filter !== '') {
        this.rows.forEach((row) => {
          this.cols.every((col) => {
            if (!col.visible) return false
            if (
              (row[col.title] + '')
                .toLowerCase()
                .includes(this.filter.toLowerCase())
            ) {
              rows.push(row)
              return false
            } else {
              return true
            }
          })
        })
      } else {
        rows = this.rows
      }

      // column filter
      this.cols.forEach((col) => {
        if (col.filter !== '') {
          const _rows = []
          rows.forEach((row) => {
            if (
              (row[col.title] + '')
                .toLowerCase()
                .includes(col.filter.toLowerCase())
            ) {
              _rows.push(row)
            }
          })

          rows = _rows
        }
      })

      return rows
    }
  },

  watch: {
    table() {
      // если данные обновились вне компонента, перезагружаем
      this.load()
    },
    filter() {
      // сбрасываем страницу при изменении фильтра
      this.page = 1
    }
  },
  mounted() {
    window.addEventListener('keydown', (e) => {
      // esc закрывает попап
      if (e.keyCode === 27) {
        this.popup = false
      }
    })
    window.addEventListener('mouseup', this.stopDrag) // драг окончен
    this.load() // загружаем данные
  },
  methods: {
    startDrag(e, col) {
      // начало сортировки столбцов
      e.preventDefault() // чтобы не выделялся текст при драге
      col.dragging = true
      this.dragging = true
      this.xpos = e.clientX
      this.col = col.title
    },

    doDrag(e) {
      // сортировка столбцов
      if (this.dragging) {
        // если сейчас тащится какой-то столбец, то работаем
        e.preventDefault() // чтобы не выделялся текст при драге
        const xpos = e.clientX
        const gap = this.$refs[this.col][0].getBoundingClientRect().width / 2
        if (xpos < this.xpos) {
          // мышка двигается влево
          const diff = this.xpos - xpos
          if (diff > gap) {
            // если мышка сдвинулась более чем на половину ширины столбца, переносим его влево
            this.colSort(-1)
          }
        } else {
          // мышка двигается вправо
          const diff = xpos - this.xpos
          if (diff > gap) {
            // если мышка сдвинулась более чем на половину ширины столбца, переносим его вправо
            this.colSort(1)
          }
        }
      }
    },

    stopDrag() {
      // кнопка мыши отпущена
      this.cols.forEach((col) => {
        col.dragging = false
      })
      this.dragging = false
    },

    colSort(direction) {
      // сортируем столбцы
      this.dragging = false
      setTimeout(
        function(that) {
          that.dragging = true
        },
        100,
        this
      ) // чтобы столбец успел переехать на свое место

      let visible = 0
      this.cols.forEach((col) => {
        if (col.visible) visible += 1
      }) // считаем, сколько видимых столбцов

      this.cols.forEach((col, idx) => {
        if (col.title === this.col) {
          if (direction < 0) {
            // вдигаем стобец влево
            if (col.order > 0) {
              let neibor = 1
              while (this.cols[idx - neibor].visible === false) {
                neibor -= 1
              }
              const title = this.cols[idx - neibor].title
              const width = this.$refs[title][0].getBoundingClientRect().width

              this.cols[idx - neibor].order += 1
              if (neibor > 1) {
                for (let c = neibor - 1; c >= 1; c--) {
                  this.cols[idx - c].order += 1
                }
              }

              col.order -= neibor
              this.xpos -= width
            }
          } else {
            // двигаем столбец вправо
            let neibor = 1
            if (col.order < visible - 1) {
              while (this.cols[idx + neibor].visible === false) {
                neibor += 1
              }
              const title = this.cols[idx + neibor].title
              const width = this.$refs[title][0].getBoundingClientRect().width

              this.cols[idx + neibor].order -= 1
              if (neibor > 1) {
                for (let c = neibor - 1; c >= 1; c--) {
                  this.cols[idx + c].order -= 1
                }
              }

              col.order += 1
              this.xpos += width
            }
          }
        }
      })

      this.cols.sort((a, b) => {
        // сортируем столбцы по новым значениям
        if (a.order < b.order) {
          return -1
        }
        if (a.order > b.order) {
          return 1
        }
        return 0
      })

      this.updateSettings() // сохраняем сортировку
    },

    highlight(text, col) {
      // подсвечиваем отфильтрованные данные
      if (this.filter !== '') {
        // global filter
        const startIndex = (text + '')
          .toLowerCase()
          .indexOf(this.filter.toLowerCase())
        const endIndex = startIndex + this.filter.length
        if (startIndex > -1) {
          text = (text + '').splice(endIndex, 0, '</span>')
          text = (text + '').splice(startIndex, 0, '<span class="globalhl">')
        }
      }

      if (col.filter !== '') {
        // column filter
        const startIndex = (text + '')
          .toLowerCase()
          .indexOf(col.filter.toLowerCase())
        const endIndex = startIndex + col.filter.length
        if (startIndex > -1) {
          text = (text + '').splice(endIndex, 0, '</span>')
          text = (text + '').splice(startIndex, 0, '<span class="localhl">')
        }
      }

      return text
    },

    getRows(col) {
      // получаем данные для столбца
      const rows = []
      const fromIdx = (this.page - 1) * this.onpage_prop
      let toIdx = fromIdx + this.onpage_prop
      if (toIdx > this.rows_filtered.length) toIdx = this.rows_filtered.length
      for (let i = fromIdx; i < toIdx; i++) {
        rows.push(this.rows_filtered[i][col.title])
      }
      return rows
    },

    load() {
      // загружаем данные в компонент
      if (this.table !== [] && this.table !== {}) {
        // check is table not empty
        const cols = []

        if (typeof this.table.cols !== 'undefined') {
          // check table is object
          this.table.cols.forEach((el) => {
            const col = {
              title: el,
              order: this.cols.length,
              visible: true,
              dragging: false,
              pos: {
                left: 0,
                top: 0
              },
              filter: ''
            }
            this.cols.push(col)
            cols.push((el + '').toLowerCase())
          })
          this.table.data.forEach((row) => {
            const r = {}
            this.cols.forEach((col) => {
              r[col.title] = row[col.order]
            })
            this.rows.push(r)
          })
        } else if (typeof this.table[0] !== 'undefined') {
          // check table is array
          const row = this.table[0]
          const _cols = []
          for (const k in row) {
            _cols.push(k)
          }
          _cols.forEach((el) => {
            const col = {
              title: el,
              order: this.cols.length,
              visible: true,
              dragging: false,
              pos: {
                left: 0,
                top: 0
              },
              filter: ''
            }
            this.cols.push(col)
            cols.push((el + '').toLowerCase())
          })
          this.table.forEach((row) => {
            const r = {}
            this.cols.forEach((col) => {
              r[col.title] = row[col.title]
            })
            this.rows.push(r)
          })
        }

        /**
         * Ключ для сохранения настроек. Формируется на основе отсортированных названий столбцов.
         * Если набор столбцов в данных совпадает, то настройки загрузятся
         */
        cols.sort()
        this.key = JSON.stringify(cols)

        const lastCols = window.localStorage.getItem(this.key)
        if (lastCols !== null) {
          this.cols = JSON.parse(lastCols)
          this.cols.forEach((col) => {
            col.dragging = false
          })
        }
      }
    },

    updateSettings() {
      // обновляем настройки
      setTimeout(
        function(that) {
          const data = JSON.stringify(that.cols)
          window.localStorage.setItem(that.key, data)
        },
        100,
        this
      )
    }
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

  .btn {
    &.clear {
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

    &.options {
      &::after {
        content: '';
        display: block;
        background-image: url(data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz48IURPQ1RZUEUgc3ZnIFBVQkxJQyAiLS8vVzNDLy9EVEQgU1ZHIDEuMS8vRU4iICJodHRwOi8vd3d3LnczLm9yZy9HcmFwaGljcy9TVkcvMS4xL0RURC9zdmcxMS5kdGQiPjxzdmcgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxuczp4bGluaz0iaHR0cDovL3d3dy53My5vcmcvMTk5OS94bGluayIgdmVyc2lvbj0iMS4xIiB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTMsNUg5VjExSDNWNU01LDdWOUg3VjdINU0xMSw3SDIxVjlIMTFWN00xMSwxNUgyMVYxN0gxMVYxNU01LDIwTDEuNSwxNi41TDIuOTEsMTUuMDlMNSwxNy4xN0w5LjU5LDEyLjU5TDExLDE0TDUsMjBaIiAvPjwvc3ZnPg==);
        background-position: center;
        background-repeat: no-repeat;
        background-size: contain;
        width: 60%;
        height: 60%;
      }
    }
  }

  input[type='text'] {
    height: 25px;
    border-radius: 12.5px;
    -webkit-appearance: none;
    border: 1px solid $gray;
    padding-left: 13px;
  }

  .btn {
    cursor: pointer;
  }

  .popup {
    position: fixed;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    z-index: 1000;
    opacity: 0;
    pointer-events: none;
    transition: all 300ms ease;

    &.active {
      opacity: 1;
      pointer-events: initial;

      .window {
        top: 5%;
      }
    }

    .overlay {
      position: absolute;
      left: 0;
      top: 0;
      width: 100%;
      height: 100%;
      background: #000;
      opacity: 0.7;
      cursor: pointer;
    }

    .window {
      position: absolute;
      left: 50%;
      top: -5%;
      transform: translateX(-50%);
      border: 1px solid $gray;
      background: #fff;
      box-shadow: 0 3px 5px rgba(#000, 0.3);
      padding: 20px 10px;
      transition: all 300ms ease;

      .close {
        position: absolute;
        right: 5px;
        top: 5px;
        font-size: 1.3em;
        cursor: pointer;
        display: flex;
        width: 15px;
        height: 15px;
        justify-content: center;
        align-items: center;
      }

      .content {
        .items {
          padding: 5px 0;
          .item {
            display: flex;
            justify-content: flex-start;
            align-items: center;

            input {
              margin-right: 10px;
            }
          }
        }
      }
    }
  }

  .pagination {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    margin-bottom: 10px;

    .onpage-select {
      margin-left: auto;
    }

    .btn {
      margin-left: 10px;
      font-family: initial;
      width: 30px;
      height: 30px;
      display: flex;
      justify-content: center;
      align-items: center;
      border: 1px solid $gray;
      border-radius: 50%;
      font-size: 10px;
      cursor: pointer;

      &.disabled,
      &:disabled {
        opacity: 0.2;
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

      &.options {
        margin-left: auto;
      }
    }
  }

  .table {
    margin-bottom: 10px;

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
        transition: all 300ms ease;

        &:nth-last-child(1) {
          border-right: 1px solid $gray;
        }

        &.dragging {
          transform: translateY(-5px);
        }

        .title {
          font-weight: bold;
          overflow: hidden;
          width: 100%;
          border-bottom: 1px solid $gray;
          padding: 3px;
          cursor: ew-resize;
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
