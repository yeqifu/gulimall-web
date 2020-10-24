<template>
  <div>
    <el-tree
      :data="menu"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKeys"
      draggable
      :allow-drop="allowDrop"
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
      dialogFormType: "", //append,edit
      dialogFormTitle: "", //弹出框的标题
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
      if (dropNode.data.label === "二级 3-1") {
        return type !== "inner";
      } else {
        return true;
      }
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