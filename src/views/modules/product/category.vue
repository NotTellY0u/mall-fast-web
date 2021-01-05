<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId"
             ref="menuTree"
             :default-expanded-keys="expandedKey" :draggable="draggable" :allow-drop="allowDrop"
             @node-drop="handleDrop">
    <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <=2" type="text" size="mini" @click="() => append(data)">
            Append
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            edit
          </el-button>
          <el-button v-if="node.childNodes.length === 0" type="text" size="mini" @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
    <el-button @click="dialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="submitData">确 定</el-button>
  </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'category',
  data () {
    return {
      pCid: 0,
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: '',
      dialogType: '',
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: '',
        productUnit: ''
      },
      dialogVisible: false,
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
    batchDelete () {
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否删除${catIds}菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({data}) => {
          this.$message({
            message: '菜单批量删除成功',
            type: 'success'
          })
          this.getMenus()
        })
      }).catch(() => {
      })
    },
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单顺序修改成功',
          type: 'success'
        })
        // 刷新菜单
        this.getMenus()
        this.expandedKey = [this.pCid]
        this.updateNodes = []
        this.maxLevel = 0
        this.pCid = 0
      })
    },
    // 共四个参数，依次为：被拖拽节点对应的 Node、结束拖拽时最后进入的节点、被拖拽节点的放置位置（before、after、inner）、event
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('draggingNode: ', draggingNode, 'dropNode: ', dropNode, dropType)
      // 1.当前节点最新的父节点ID
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid = pCid
      // 2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.data.catLevel
          if (siblings[i].data.catLevel !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 修改子节点的层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel})
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }
      // 3.当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let cNode = node.childNodes[i].data
          this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      // 1.被拖动的当前节点以及所在的父节点总层数不能大于3

      // 1).被拖动的当前节点总层数
      // console.log('allowDrop', draggingNode, dropNode, type)
      // this.maxLevel = 0
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      if (type === 'inner') {
        // console.log(`this.maxLevel: ${this.maxLevel}; draggingNode.data.catLevel: ${draggingNode.data.catLevel}; dropNode.level: ${dropNode.level}`)
        return (deep + dropNode.level) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    countNodeLevel (node) {
      // 找到所有子节点，统计最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    edit (data) {
      console.log('要修改的数据', data)
      this.title = '编辑分类'
      this.dialogType = 'edit'
      this.dialogVisible = true

      // 发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 请求成功
        console.log('要回显的数据', data)
        this.category.name = data.category.name
        this.category.catId = data.category.catId
        this.category.icon = data.category.icon
        this.category.productUnit = data.category.productUnit
        this.category.parentCid = data.category.parentCid
      })
    },
    append (data) {
      console.log('append', data)
      this.category.name = ''
      this.category.catId = null
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
      this.title = '添加分类'
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
    },
    submitData () {
      if (this.dialogType === 'add') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    editCategory () {
      let {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId: catId, name: name, icon: icon, productUnit: productUnit}, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单修改成功',
          type: 'success'
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandedKey = [this.category.parentCid]
      })
    },
    // 添加三级分类
    addCategory () {
      console.log('提交的三级分类数据', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单保存成功',
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false
        this.getMenus()
        this.expandedKey = [this.category.parentCid]
      })
    },

    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(`是否删除${data.name}菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          this.$message({
            message: '菜单删除成功',
            type: 'success'
          })
          // 刷新出新的菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [node.parent.data.catId]
        })
      }).catch((data) => {
      })
    }
  },
  created () {
    this.getMenus()
  }
}
</script>

<style scoped>

</style>
