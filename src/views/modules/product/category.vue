<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button
      v-if="draggable"
      @click="batchSave"
      size="mini"
      round
      style="margin-left: 1%"
      >批量保存</el-button
    >
    <el-button
      type="danger"
      @click="batchDelete"
      size="mini"
      round
      style="margin-left: 1%"
      >批量删除</el-button
    >
    <el-tree
      :data="menu"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <!-- Append和Delete -->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            v-if="node.level <= 2"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            type="text"
            v-if="node.childNodes.length == 0"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <!-- 添加分类的对话框 -->
    <el-dialog
      :title="dialogFormTitle"
      :visible.sync="dialogFormVisible"
      width="30%"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="icon">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      menu: [],
      expandedKeys: [],
      dialogFormVisible: false, //添加分类对话框是否显示，true显示，false不显示
      dialogFormType: "", //对话框的类型  添加：append,修改：edit
      dialogFormTitle: "", //弹出对话框的标题
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: null,
        productUnit: null,
        productCount: 0,
      },
      defaultProps: {
        children: "children",
        label: "name",
      },
      maxLevel: 0,
      updateNodes: [],
      draggable: true, //是否开启拖拽，true 开启拖拽  false 关闭拖拽
      pCid: [],
    };
  },
  methods: {
    //获取菜单数据
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        //解构data，就可以通过.拿到其中的数据了
        console.log("data:");
        console.log(data.data);
        this.menu = data.data;
      });
    },
    //添加一个树的节点
    append(data) {
      console.log("append:");
      console.log(data);
      //弹出添加分类对话框
      this.dialogFormVisible = true;
      this.dialogFormTitle = "添加分类";
      this.dialogFormType = "append";
      //给新添加的分类赋一些默认值
      this.category.parentCid = data.catId;
      //先乘1，将字符串转换为整型，再加一
      this.category.catLevel = data.catLevel * 1 + 1;

      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.sort = 0;
      this.category.productUnit = "";
      this.category.showStatus = 1;
    },
    //修改一个分类
    edit(data) {
      this.dialogFormVisible = true;
      this.dialogFormTitle = "修改分类";
      this.dialogFormType = "edit";
      //发送get请求获取当前的最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //请求成功,回显数据
        console.log("要回显的数据：");
        console.log(data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    //弹出框的确定按钮的事件
    submitData() {
      if (this.dialogFormType == "append") {
        this.addCategory();
      }
      if (this.dialogFormType == "edit") {
        this.editCategory();
      }
    },
    //添加一个三级菜单的分类
    addCategory() {
      console.log("添加三级分类的数据：");
      console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "分类添加成功!",
        });
        //关闭添加分类的弹出框
        this.dialogFormVisible = false;
        //添加成功之后重新加载菜单树
        this.getMenus();
        //设置添加的菜单默认展开
        this.expandedKeys = [this.category.parentCid];
      });
    },
    //修改分类
    editCategory(data) {
      var { name, catId, icon, productUnit } = this.category;
      var data = {
        name: name,
        catId: catId,
        icon: icon,
        productUnit: productUnit,
      };
      console.log("修改分类的数据：");
      console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "分类修改成功!",
        });
        //关闭添加分类的弹出框
        this.dialogFormVisible = false;
        //修改成功之后重新加载菜单树
        this.getMenus();
        //设置修改的菜单默认展开
        this.expandedKeys = [this.category.parentCid];
      });
    },
    //删除一个树的节点
    remove(node, data) {
      console.log(node);
      var ids = [data.catId];
      this.$confirm(`此操作将永久删除【${data.name}】分类, 是否继续?`, "提示", {
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
              type: "success",
              message: "分类删除成功!",
            });
            //删除成功之后重新加载菜单树
            this.getMenus();
            //设置删除的菜单默认展开
            console.log(node.parent.data.catId);
            this.expandedKeys = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    allowDrop(draggingNode, dropNode, type) {
      //1、被拖动的当前节点所在的父节点总层数不能大于3
      console.log("draggingNode,dropNode,type:");
      console.log(draggingNode, dropNode, type);
      //2、统计当前被拖动节点的总层数
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+父节点所在的深度不大于3即可   计算出绝对值
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log("深度：", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handleDrop: ", draggingNode, dropNode, dropType);
      //1、当前节点最新的父节点id
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

      //2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level;
            //修改他子节点的层级
            this.updateChildNodeLevel(siblings[i]);
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

      //3、当前拖拽节点的最新层级
      console.log("updateNodes", this.updateNodes);
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "分类顺序修改成功!",
        });
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedKeys = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    //批量删除
    batchDelete() {
      let catIds = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的元素", checkedNodes);
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`是否批量删除这些菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "分类批量删除成功!",
            });
            //刷新菜单
            this.getMenus();
          });
        })
        .catch(() => {});
    },
  },
  //vue的生命周期，创建完成（可以访问当前的this实例）
  created() {
    this.getMenus();
  },
};
</script>

<style>
</style>