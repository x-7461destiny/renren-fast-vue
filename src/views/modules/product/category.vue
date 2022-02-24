<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      @node-drop="handleDrop"
      :draggable="draggable"
      :allow-drop="allowDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Modify
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
  <!-- <div>三级分类维护</div> -->
</template>

<script>
export default {
  data() {
    return {
      pCid : [],
      draggable:false,
      updateNodes: [],
      title: "",
      dialogType: "",
      maxLevel: 0,
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null,
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "child",
        label: "name",
      },
    };
  },
  methods: {
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        //  用大括号对data解构似乎没怎么学过
        console.log("成功获取菜单数据……", data.data);
        this.menus = data.data;
      });
    },
    batchDelete() {
      let catIds = []; //存储发送给后端的id
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的元素",checkedNodes);
      for(let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`是否批量删除${catIds} ?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.l.adornData(catIds, false)
        }).then(() => {
          this.$message({
          message: "菜单批量删除成功",
          type: "success",
        });
        this.getMenu();
        })
      }).catch(() => {

      });
    },
    batchSave() {
            //当前拖拽节点的最新层级
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序成功",
          type: "success",
        });
        //刷新菜单，显示效
        this.getMenu();
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
        // this.pCid = 0;
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      // 被拖动当前节点所在的父节点总层数不能大于三

      // 被拖动的当前节点总层数
      console.log("allowDrop:", draggingNode, dropNode, type);
      console.log("------")
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+ 父节点所在的深度和不大于三
      console.log(this.maxLevel);
      let deep =this.maxLevel - draggingNode.data.catLevel + 1;
      console.log("深度", deep);
      if (type == "inner") {
        // console.log("test01------")
        // console.log(`this.maxLevel:${this.maxLevel}; draggingNode.data.catLevel: ${draggingNode.data.catLevel};dropNode.level: ${dropNode.level}`)
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0)
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // console.log("tree drop:", dropNode.label,dropType);
      console.log("handleDrop: ", draggingNode, dropNode, dropType);
      //当前节点的父节点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      this.pCid.push(pCid);
      console.log("siblings", siblings);
      //当前拖拽节点的最新排序
      for (let i = 0; i < siblings.length; i++) {
        // console.log("siblings",siblings)
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;

          //遍历当前拖拽的节点
          if (siblings[i].level != draggingNode.level) {
            // 当前节点的层级发生变化
            // this.updateNodes.push({catId:siblings[i].data.catId,sort:i,parentCid: pCid});
            // if(dropType == 'before' || dropType == 'after') {
            //   catLevel = dropNode.level;
            // } else {
            //   catLevel = dropNode.level +1 ;
            // }
            this.updateChildNodeLevel(siblings[i]);

            //修改子节点的层级
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log(this.updateNodes);

    },
    countNodeLevel(node) {
      console.log("node:", node);
      //找到所有子节点，求最大深度
      if (node.child != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].catLevel > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
      console.log(this.maxLevel);
    },
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
      //发送请求获取节点最新数据修改
      // console.log("the")
      // console.log(data);
    },
    edit(data) {
      console.log("要修改的数据", data);
      this.dialogType = "edit";
      this.title = "修改分类";
      this.dialogVisible = true;
      //发送请求获取节点最新数据修改
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        methods: "get",
      }).then(({ data }) => {
        console.log("teh");
        console.log(data);
        //请求成功
        this.category.name = data.category.name;
        this.category.catId = data.category.catId;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
        this.category.parentCid = data.category.parentCid;
      });
      this.category.name = data.name;
      this.category.catId = data.catId;
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    editCategory() {
      console.log(this.category);
      var { catId, name, icon, productUnit } = this.category;
      // var data = ;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        //shutdown 对话框
        this.dialogVisible = false;
        this.getMenu();
        this.expandedKey = [this.category.parentCid];
      });
    },
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        this.getMenu();
        //刷新新的菜单并展开
        this.expandedKey = [this.category.parentCid];
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除[${data.name}]菜单, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success",
            });
            //刷新出新的菜单
            this.getMenu();
            //设置需要展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {
          console.log(node);
        });
      console.log("remove", node, data);
    },
  },
  created() {
    this.getMenu();
  },
};
</script>

<style>
</style>

