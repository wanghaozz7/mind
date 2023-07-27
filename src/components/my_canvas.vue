<template>
  <div style="width: 1426px;height: 800px;position: relative;">
    <canvas width="1426px" height="800px" ref="canvas" tabindex="1" />
    <input type="text" :style="getInputStyle" class="my-input" :value="input.value" @input="handleInput" ref="input"
      v-show="showInput">
  </div>
</template>

<script>
export default {
  props: {
    tree: {
      type: Object,
      default() {
        return {}
      }
    }
  },
  data() {
    return {
      renderTree: null,
      searchArray: null,
      nodeAttrs: null,
      lastClick: null,
      editNode: null,
      animating: false,
      requestAnimation: null,
      input: null,
      onInput: false,
      showInput: false
    }
  },
  methods: {
    render() {
      // 清理画布
      // 计算子树高度
      // 赋值根节点位置并计算所有节点的属性
      // 渲染树
      // 渲染添加按钮
      // 清理事件监听器 添加事件监听器 (防止重复监听)
      const canvas = this.$refs.canvas;
      this.clearCanvas();
      this.getSubtreeHeight(this.renderTree);
      this.renderTree.x = 50;
      this.renderTree.y = 400;
      this.getRenderTreeAttrs(this.renderTree)
      this.treeRender(this.renderTree);
      this.buttonRender();
      canvas.removeEventListener("click", this.handleCanvasClick);
      canvas.addEventListener("click", this.handleCanvasClick);
    },
    // 清理画布
    clearCanvas() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      this.searchArray.length = 0;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    },
    // 计算每个子树的高度(横向)
    getSubtreeHeight(node) {
      // 叶子节点
      const children = node?.children;
      if (!children || children.length === 0) return node.subtreeHeight = this.nodeAttrs.height;
      let height = 0;
      for (let child of children) height += this.getSubtreeHeight(child) + this.nodeAttrs.verticalGap;
      height -= this.nodeAttrs.verticalGap
      return node.subtreeHeight = height;
    },
    // 计算每个节点用来渲染的属性(x,y,width)
    getRenderTreeAttrs(node) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      const { label, fontStyle = "normal 24px 微软雅黑" } = node;

      // 计算宽度
      ctx.font = fontStyle;
      let width = 0;
      for (let char of label) width += ctx.measureText(char).width;
      node.width = width + 2 * this.nodeAttrs.horizonPadding;

      // 计算子节点的x y
      const children = node?.children;
      if (!children || children.length === 0) return;
      const x = node.x + node.width + this.nodeAttrs.horizonGap;
      let top = node.y - node.subtreeHeight / 2;
      for (let child of children) {
        child.x = x;
        child.y = top + child.subtreeHeight / 2;
        top += child.subtreeHeight + this.nodeAttrs.verticalGap;
        this.getRenderTreeAttrs(child);
      }
    },
    // 渲染树
    treeRender(node) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      ctx.roundRect(node).stroke();
      this.searchArray.push(node)
      const children = node?.children;
      if (!children || children.length === 0) return;
      for (let child of children) {
        const start = { x: node.x + node.width, y: node.y + this.nodeAttrs.height / 2 };
        const end = { x: child.x, y: child.y + this.nodeAttrs.height / 2 }
        ctx.bezier({ start, end }).stroke();
        this.treeRender(child)
      }
    },
    // 根据点击的位置返回节点
    searchNode(x, y) {
      // 判断添加按钮
      if (this.editNode.showButton) {
        const button = this.editNode.button;
        if (y >= button.y - button.r && y <= button.y + button.r && x >= button.x && x <= button.x + 2 * button.r) {
          return this.editNode;
        }
      }

      // 判断节点
      for (let node of this.searchArray) {
        const area = {
          xStart: node.x,
          xEnd: node.x + node.width,
          yStart: node.y,
          yEnd: node.y + this.nodeAttrs.height
        }
        if (x >= area.xStart && x <= area.xEnd && y >= area.yStart && y <= area.yEnd) {
          return node;
        }
      }

      return null;
    },
    // 画布点击事件分发 (对象分发 事件类型分发)
    handleCanvasClick(e) {
      const x = e.clientX;
      const y = e.clientY;
      const target = this.searchNode(x, y);
      const time = new Date();
      if (target) {
        if (target.showButton) {
          this.addNode(target.target)
        } else if (target.id == this.lastClick.target?.id) {
          // 如果点击的是同一个目标则根据间距判断双击事件
          if (Math.abs(this.lastClick.time - time) < 300) this.handleNodeDoubleClick(target);
          else this.handleNodeClick(target);
        } else this.handleNodeClick(target);
      } else {
        this.handleInvalidClick();
      }
      // 更新上一次的点击对象
      this.lastClick.target = target;
      this.lastClick.time = time;
    },
    // 节点被单击
    handleNodeClick(node) {
      const button = {
        label: `${node.label}的添加按钮`,
        id: `${node.label}的添加按钮`,
        x: node.x + node.width + 15,
        y: node.y + this.nodeAttrs.height / 2,
        r: 8
      }
      this.editNode.target = node;
      this.editNode.showButton = true;
      this.editNode.button = button;
      this.editNode.renameTag = false;
      this.render()
    },
    // 节点被双击
    handleNodeDoubleClick(node) {
      // 当前节点进入编辑状态

      this.editNode.target = node;
      this.editNode.showButton = false;
      this.editNode.renameTag = true;
      this.getInputAttr(node);
      this.onInput = true;
      this.render();
    },
    // 点击空白区域
    handleInvalidClick() {
      this.editNode = {
        target: null,
        showButton: false,
        renameTag: false,
        button: null
      }
      // this.animating = false;
      // window.cancelAnimationFrame(this.requestAnimation);
      this.onInput = false;
      this.closeInput();
      this.render();
    },
    // 渲染添加按钮
    buttonRender() {
      if (!this.editNode.showButton) return;
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      ctx.circle(this.editNode.button).stroke();
    },
    // 添加新节点
    addNode(target) {
      this.editNode = {
        target: null,
        showButton: false,
        renameTag: false,
        button: null
      }
      const newNode = {
        label: '新节点' + this.searchArray.length,
        id: '新节点' + this.searchArray.length,
      };
      const ergodicTree = (node) => {
        if (node.id === target.id) {
          if (node.children) {
            node.children.push(newNode);
          } else {
            this.$set(node, 'children', [newNode])
          }
          return this.render();
        }
        if (node.children) for (let child of node.children) ergodicTree(child)
      }
      ergodicTree(this.renderTree)
    },
    // 执行动画(进入一个需要不断刷新的状态 —— 拖动、输入)
    animation() {
      const renderAnimation = (timeStamp) => {
        this.render();
        if (this.animating) this.requestAnimation = window.requestAnimationFrame(renderAnimation);
      };
      this.requestAnimation = window.requestAnimationFrame(renderAnimation);
    },
    // 计算Input的属性
    getInputAttr(node) {
      this.input.top = node.y + 9;
      this.input.left = node.x + 11;
      this.input.fontSize = 24;
      this.input.height = 30;
      this.input.width = node.width - 30;
      this.input.value = node.label;
      this.showInput = true;
    },
    closeInput() {
      this.showInput = false
    },
    handleInput(e) {
      const input = this.$refs.input;
      console.log('e', e, input.value);
      this.editNode.target.label = input.value;
      this.render();
    }

  },
  computed: {
    getInputStyle() {
      const top = this.input.top + 'px';
      const left = this.input.left + 'px';
      const height = this.input.height + 'px';
      const fontSize = this.input.fontSize + 'px';
      return {
        top, left, height, fontSize, position: 'absolute'
      }
    }
  },
  created() {
    this.renderTree = Object.assign({}, this.tree);
    this.searchArray = new Array();
    this.nodeAttrs = {
      height: 50,
      verticalGap: 30,
      horizonGap: 150,
      horizonPadding: 15
    }
    this.lastClick = {
      target: null,
      time: new Date()
    }
    this.input = {
      top: undefined,
      left: undefined,
      width: undefined,
      height: undefined,
      fontSize: undefined,
      value: undefined
    }
    this.editNode = {
      target: null,
      showButton: false,
      renameTag: false,
      button: null
    }
  },
  mounted() {
    const that = this;
    // 绘制节点
    CanvasRenderingContext2D.prototype.roundRect = function (arg) {
      const { x, y, width, label, fontStyle = "normal 24px 微软雅黑" } = arg;

      // 样式
      this.lineWidth = 2;
      this.strokeStyle = "#F00";
      this.font = fontStyle;
      this.color = 'black'

      const height = that.nodeAttrs.height;
      const radius = 8;
      if (width < 2 * radius) radius = width / 2;
      if (height < 2 * radius) radius = height / 2;
      this.beginPath();
      this.moveTo(x + radius, y);
      this.arcTo(x + width, y, x + width, y + height, radius);
      this.arcTo(x + width, y + height, x, y + height, radius);
      this.arcTo(x, y + height, x, y, radius);
      this.arcTo(x, y, x + width, y, radius);
      if ((!that.onInput && !that.editNode.target) || that.editNode.target?.label !== label) this.fillText(label, x + 15, y + 35);
      this.closePath();

      return this;
    };
    // 绘制父子节点之间连线
    CanvasRenderingContext2D.prototype.bezier = function (arg) {
      const { start, end } = arg;
      const middle = {
        x: start.x + 50,
        y: start.y
      }

      this.lineWidth = 1;
      this.strokeStyle = "lightblue";

      this.beginPath();
      this.moveTo(start.x, start.y);
      this.lineTo(middle.x, middle.y);
      this.stroke();

      this.beginPath();
      this.moveTo(middle.x, middle.y);
      this.quadraticCurveTo(middle.x, end.y, end.x, end.y);
      this.stroke();

      return this;
    };
    // 绘制按钮
    CanvasRenderingContext2D.prototype.circle = function (arg) {
      const { x, y, r } = arg;

      this.fillStyle = "purple";

      this.beginPath(); //创建一个路径
      this.moveTo(x + r, y);
      this.arc(x, y, r, 0, 2 * Math.PI);
      this.closePath();
      this.fill();

      return this;
    };
    // 输入光标
    CanvasRenderingContext2D.prototype.verticalBar = function (arg) {
      const { x, y, height } = arg;

      this.lineWidth = 1;
      this.strokeStyle = "black";

      this.beginPath();
      this.moveTo(x, y);
      this.lineTo(x, y + height);
      this.stroke();

      return this;
    };
    this.render()
  }
}
</script>

<style scoped>
.my-input {
  border: 0;
  outline: none;
  background-color: transparent;
}
</style>

