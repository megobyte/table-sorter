<template lang="pug">
  .container
    table-sorter(:table="table")
</template>

<script>
import TableSorter from '~/components/table-sorter.vue'

export default {
  components: {
    TableSorter
  },
  data() {
    return {
      table: {}
    }
  },
  async mounted() {
    const r = await this.$axios.get('/data.json')
    this.table = r.data
  }
}
</script>

<style lang="scss">
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
}
</style>
