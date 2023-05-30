<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <!-- :expand-on-click-node="false" 点击字段不进行选中
  show-checkbox node-key="catId"  显示复选框  给node设置key
  :default-expanded-keys="expandedKey"  展开的分类列表
  draggable  对话框
   :allow-drop="allowDrop"  节点可以拖拽

  -->
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)">Append</el-button>

          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)">update</el-button>

          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">Delete</el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
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
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具 js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    return {
      draggable: false,
      // 1.分类的属性
      menus: [],
      expandedKey: [],
      defaultProps: {
        // data中的属性字段
        children: 'children',
        label: 'name'
      },
      // 2.对话框
      title: '',
      dialogType: '', // 判断是edit 还是save
      dialogVisible: false,
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
      pCid: [],
      // 3. 拖动的最大层级
      maxLevel: 0,
      // 4. 正在修改的节点数组
      updateNodes: []
    }
  },
  methods: {
    handleNodeClick (data) {
      // console.log(data);
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取菜单数据', data.data)
        this.menus = data.data
      })
    },
    // 弹出添加对话框
    append (data) {
      console.log('append...', data)
      this.title = '添加分类'
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1

      this.category.name = ''
      this.category.catId = null
      this.category.icon = ''
      this.category.productUnit = ''
    },
    // 弹出修改对话框
    edit (data) {
      console.log('修改方法', data)
      this.dialogType = 'edit'
      this.title = '修改分类'
      this.dialogVisible = true
      // 从数据库中查询出最新的节点信息
      // this.category.name = data.name
      // this.category.catId = data.catId
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        console.log('要回显的数据', data)
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
      })
    },
    submitData () {
      if (this.dialogType == 'add') {
        this.addCategory()
      }
      if (this.dialogType == 'edit') {
        this.editCategory()
      }
    },
    // 添加方法
    addCategory () {
      console.log('addCategory...', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '保存成功!'
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandedKey = [this.category.parentCid]
      })
    },
    // 修改方法
    editCategory () {
      let {catId, name, icon, productUnit} = this.category
      let data = {catId, name, icon, productUnit}
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData(data, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '修改成功!'
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandedKey = [this.category.parentCid]
      })
    },
    // 删除方法
    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】菜单`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            this.$message({
              type: 'success',
              message: '删除成功!'
            })
            this.getMenus()
            // 展开菜单
            this.expandedKey = [node.parent.data.catId]
          })
        })
        .catch(() => {
          this.$message({
            type: 'cancel',
            message: '取消删除!'
          })
        })
    },
    // 拖拽函数
    allowDrop (draggingNode, dropNode, type) {
      // 1.被拖动的节点，以及所在的父节点总层数不能大于三
      console.log('被拖动的节点', draggingNode, dropNode, type)
      // (1) 被拖动的当前节点的总层数
      this.countNodeLevel(draggingNode)
      console.log('正在拖动的节点深度', this.maxLevel)
      let deep = Math.abs(this.maxLevel - draggingNode.level + 1)
      console.log('真正的级数', deep)
      this.maxLevel = 0
      if (type == 'inner') {
        return (deep + dropNode.level) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    // 找到正在拖动的子节点的最大深度
    countNodeLevel (node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
// 拖拽成功触发的函数
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)
      // 1.当前节点的最新的父节点id
      let pCid = 0
      let siblings = null
      if (dropType == 'inner') {
        pCid = dropNode.data.catId
        // 该数组存储有 当前被拖动的节点
        siblings = dropNode.childNodes
      } else {
        pCid = (dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId)
        siblings = dropNode.parent.childNodes
      }
      this.pCid.push(pCid)
      // 2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        // 遍历的是当前被拖动的节点
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 如果当前层级发生变化
          let catLevel = draggingNode.level
          if (siblings[i].level != draggingNode.level) {
            // 当前节点 层级发生变化
            catLevel = siblings[i].level
            // 当前节点中的 子节点也要发生变化
            this.updateChildNodes(siblings[i])
          }
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel})
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }
      console.log(this.updateNodes)
    },
    // 修改子节点的层级
    updateChildNodes (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let cNode = node.childNodes[i].data
          this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
          this.updateChildNodes(node.childNodes[i])
        }
      }
    },
    // 批量保存
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '修改成功'
        })
        this.getMenus()
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.pCid = []
      })
    },
    // 批量删除
    batchDelete () {
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('选中的元素', checkedNodes)
      let catIds = []
      let name = []
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
        name.push(checkedNodes[i].name)
      }
      this.$confirm(`是否批量删除【${name}】菜单`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              type: 'success',
              message: '批量删除成功!'
            })
            this.getMenus()
          })
        })
        .catch(() => {
          this.$message({
            type: 'cancel',
            message: '取消删除!'
          })
        })
    }
  },
  // 计算属性 类似于 data 概念
  computed: {},
  // 监控 data 中的数据变化
  watch: {},

  // 生命周期 - 创建完成（可以访问当前 this 实例）
  created () {
    this.getMenus()
  },
  // 生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted () {
  },
  beforeCreate () {
  }, // 生命周期 - 创建之前
  beforeMount () {
  }, // 生命周期 - 挂载之前
  beforeUpdate () {
  }, // 生命周期 - 更新之前
  updated () {
  }, // 生命周期 - 更新之后
  beforeDestroy () {
  }, // 生命周期 - 销毁之前
  destroyed () {
  }, // 生命周期 - 销毁完成
  activated () {
  } // 如果页面有 keep-alive 缓存功能，这个函数会触发
}
</script>
<style scoped>
</style>
