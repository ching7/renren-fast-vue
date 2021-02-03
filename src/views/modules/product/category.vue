<!--  -->
<template>
  <div class>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" type="primary" @click="batchSave">batch save</el-button>
    <el-button type="primary" @click="batchDelete">batch delete</el-button>
    <el-tree @node-drop="handleDrop" :draggable="draggable" :allow-drop="allowDrop" :data="menus" :props="defaultProps" :expand-on-click-node="false" :showCheckbox=true node-key="catId"
      :default-expanded-keys="expandedKey"
      ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)">Append</el-button>
          <el-button v-if="node.childNodes.length==0" type="text" size="mini" @click="() => remove(node, data)">Delete</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal='false'>
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
        <el-button type="primary" @click="sumbitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import引入的组件需要注入到对象中才能使用
  components: {},
  data () {
    return {
      title: '提示',
      pCid: [],
      draggable: false,
      updatedNodes: [],
      maxLevel: 0,
      dialogVisibleType: '',
      menus: [],
      expandedKey: [],
      dialogVisible: false,
      category: {
        name: '',
        catId: null,
        parentCid: 0,
        icon: '',
        productUnit: '',
        catLevel: 0,
        showStatus: 1,
        sort: 0
      },
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
      }).then(({ data }) => {
        console.log('成功获取菜单数据' + data.page)
        this.menus = data.page
      })
    },
    batchDelete () {
      let checkNodes = this.$refs.menuTree.getCheckedNodes()
      let catIds = []
      console.log('checkNodes:', checkNodes)
      for (let i = 0; i < checkNodes.length; i++) {
        catIds.push(checkNodes[i].catId)
      }
      this.$confirm(`此是批量delete[${catIds}]?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            message: `菜单删除成功`,
            type: 'success'
          })
          // 刷新
          this.getMenus()
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updatedNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: `成功`,
          type: 'success'
        })
        // 设置默认显示菜单
        this.expandedKey = this.pCid
        this.getMenus()
        this.updatedNodes = []
        this.maxLevel = 0
        // this.pCid = 0
      })
    },
    countNodeLevel (node) {
      // 找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        node.childNodes.forEach(element => {
          if (element.level > this.maxLevel) {
            this.maxLevel = element.level
          }
          this.countNodeLevel(element)
        })
      }
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('tree drop: ', dropNode.label, dropType)
      // 当前拖拽节点最新父节点
      // 当前拖拽节点的顺序
      // 当前节点层级
      let siblings = null
      let pCid = 0
      if (dropType === 'before' || dropType === 'after') {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 遍历正在拖拽的节点
          let catLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 递归修改子节点层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updatedNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel })
        } else {
          this.updatedNodes.push({ catId: siblings[i].data.catId, sort: i })
        }
      }
      console.log('this.updatedNodes', this.updatedNodes)
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let index = 0; index < node.childNodes.length; index++) {
          let cNode = node.childNodes[index].data
          this.updatedNodes.push({ catId: cNode.catId, catLevel: node.childNodes[index].level })
          this.updateChildNodeLevel(node.childNodes[index])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      // 被拖动节点和总结的层数不大于3
      console.log(draggingNode, dropNode, type)
      // 计算当前节点的总层数
      this.countNodeLevel(draggingNode)
      // 当前拖动节点加上父节点不大于3
      // 深度
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      if (type === 'inner') {
        return (deep + dropNode.level) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    addCategory () {
      console.log(this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          message: `成功`,
          type: 'success'
        })
        this.dialogVisible = false
        // 设置默认显示菜单
        this.expandedKey = [this.category.parentCid]
        this.getMenus()
      })
    },
    editCategory () {
      let { catId, name, icon, productUnit } = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      }).then(({ data }) => {
        this.$message({
          message: `成功`,
          type: 'success'
        })
        this.dialogVisible = false
        // 设置默认显示菜单
        this.expandedKey = [this.category.parentCid]
        this.getMenus()
      })
    },
    sumbitData () {
      if (this.dialogVisibleType === 'append') {
        this.addCategory()
      }
      if (this.dialogVisibleType === 'edit') {
        this.editCategory()
      }
    },
    edit (data) {
      this.dialogVisible = true
      this.title = '修改'
      this.dialogVisibleType = 'edit'
      // 修改需要刷新
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'post'
      }).then(({ data }) => {
        console.log(data)
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
      })
    },
    append (data) {
      this.title = '新增'
      this.dialogVisibleType = 'append'
      this.dialogVisible = true
      console.log('append:', data)
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },
    remove (node, data) {
      let catIds = [data.catId]
      this.$confirm(`此是否删除[${data.name}]菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            message: `菜单删除成功`,
            type: 'success'
          })
          // 刷新
          this.getMenus()
          // 设置默认显示菜单
          this.expandedKey = [node.parent.data.catId]
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
      console.log('remove', node, data)
    }
  },
  // 监听属性 类似于data概念
  computed: {},
  // 监控data中的数据变化
  watch: {},
  // 方法集合
  // 生命周期 - 创建完成（可以访问当前this实例）
  created () {
    this.getMenus()
  },
  // 生命周期 - 挂载完成（可以访问DOM元素）
  mounted () { },
  beforeCreate () { }, // 生命周期 - 创建之前
  beforeMount () { }, // 生命周期 - 挂载之前
  beforeUpdate () { }, // 生命周期 - 更新之前
  updated () { }, // 生命周期 - 更新之后
  beforeDestroy () { }, // 生命周期 - 销毁之前
  destroyed () { }, // 生命周期 - 销毁完成
  activated () { } // 如果页面有keep-alive缓存功能，这个函数会触发
}
</script>
<style scoped>
</style>