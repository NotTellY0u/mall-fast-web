<template>
  <el-tree :data="menus" :props="defaultProps" node-key="catId" ref="menuTree" @node-click="nodeClick">
  </el-tree>
</template>

<script>
export default {
  data () {
    return {
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  methods: {
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取到菜单数据', data.data)
        this.menus = data.data
      })
    },
    nodeClick (data, node, component) {
      console.log('子组件category的节点被点击', data, node, component)
      this.$emit('tree-node-click', data, node, component)
    }
  },

  created () {
    this.getMenus()
  },
  name: 'category.vue'
}
</script>

<style scoped>

</style>
