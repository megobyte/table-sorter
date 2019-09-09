<template lang="pug">
  .container
    table-sorter(:table="table")
</template>

<script>
import tableSorter from '~/components/table-sorter.vue'

export default {
  components: {
    tableSorter
  },
  data() {
    return {
      table: {}
    }
  },
  mounted: async function() {
    var r = await this.$axios.get("/data.json")
    this.table = r.data;
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
