<template>
  <el-tree :data="menus" :props="defaultProps" node-key="catId" ref="menuTree" @node-click="nodeclick">
  </el-tree>
</template>

<script>
export default {
  data() {
    return {
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
        this.menus = data.data;
      });
    },
    nodeclick(data,node, component) {
      console.log("子组件的节点被点击",data,node, component)  
      //向父组件发送事件
      this.$emit("tree-node-click",data,node, component)
    }
    
  },
  created() {
      this.getMenu();  
  }
};
</script>

<style>
</style>