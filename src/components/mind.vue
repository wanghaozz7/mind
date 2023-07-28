<template>
  <div class="mind">
    <canvas ref="canvas" />
    <input type="text" :style="getInputStyle" class="input" @input="handleInput" @change="handleInputChange" ref="input"
      v-show="editNode.showInput">
  </div>
</template>

<script>
export default {
  name: 'mind',
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
      nodeAttrs: {
        height: 50,
        verticalGap: 30,
        horizonGap: 150,
        horizonPadding: 15
      },
      searchArray: new Array(),
      editNode: {
        target: null,
        showButton: false,
        showInput: false,
        addButton: {
          label: undefined,
          id: undefined,
          x: undefined,
          y: undefined,
          r: undefined
        },
        deleteButton: {
          label: undefined,
          id: undefined,
          x: undefined,
          y: undefined,
          r: undefined
        }
      },
      preClick: {
        target: null,
        time: new Date()
      },
      input: {
        top: undefined,
        left: undefined,
        width: undefined,
        height: undefined,
        fontSize: undefined,
        value: undefined
      },
      dragEvent: {
        target: null,
        startX: undefined,
        startY: undefined,
        changeX: undefined,
        changeY: undefined,
        parent: null
      },
      animating: false,
      requestAnimation: null,
      rootPreCoordinate: {
        x: undefined,
        y: undefined
      }
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
  methods: {
    // 更新画布内容
    render() {
      // 清理画布
      // 渲染树
      // 清理事件监听器 添加事件监听器 (防止重复监听)
      const canvas = this.$refs.canvas;
      this.clearCanvas();
      this.treeRender();
      canvas.removeEventListener("click", this.handleCanvasClick);
      canvas.addEventListener("click", this.handleCanvasClick);
    },
    // 清空画布
    clearCanvas() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      this.searchArray.length = 0;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    },
    // 渲染整棵树
    treeRender() {
      const canvas = this.$refs.canvas;
      // 计算子树高度 
      this.getSubtreeHeight(this.renderTree);
      // 根据整个树的高度重新计算画布的高度
      canvas.height = this.renderTree.subtreeHeight + 300;

      // rootPreCoordinate代表根节点拖动前的坐标 当拖动结束后会更新 这里是为他赋初值(从左开始 上下居中)
      if (!this.rootPreCoordinate.x) this.rootPreCoordinate.x = 50;
      if (!this.rootPreCoordinate.y) this.rootPreCoordinate.y = (this.renderTree.subtreeHeight + 300) / 2;

      this.renderTree.x = this.rootPreCoordinate.x;
      this.renderTree.y = this.rootPreCoordinate.y;

      // 如果拖拽的是根节点 则根节点新坐标 = 拖动前坐标 + 拖拽开始后的偏移量
      if (this.dragEvent.target?.id === this.renderTree.id) {
        this.renderTree.x += this.dragEvent.changeX;
        this.renderTree.y += this.dragEvent.changeY;
      }

      this.searchArray.length = 0;
      this.searchArray.push(this.renderTree)
      // 计算每个节点的坐标和宽度 并根据最右侧的节点更新画布的宽度
      this.getRenderTreeAttrs(this.renderTree);
      this.renderNodes(this.renderTree);
      this.buttonRender();
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

      // 根据节点的文字计算宽度
      ctx.font = fontStyle;
      let width = 0;
      if (node.label) for (let char of label) width += ctx.measureText(char).width;
      node.width = width + 2 * this.nodeAttrs.horizonPadding;

      // 计算子节点的x y
      const children = node?.children;
      if (!children || children.length === 0) return;
      const x = node.x + node.width + this.nodeAttrs.horizonGap;
      let top = node.y - node.subtreeHeight / 2;
      for (let child of children) {
        child.x = x;
        child.y = top + child.subtreeHeight / 2;
        // 通过当前节点的x坐标更新画布的最大宽度
        if (Math.ceil(child.x) + 250 > canvas.width) canvas.width = Math.ceil(child.x) + 250;
        top += child.subtreeHeight + this.nodeAttrs.verticalGap;
        this.getRenderTreeAttrs(child);
        this.searchArray.push(child)
      }

    },
    // 递归渲染所有节点
    renderNodes(node) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");

      // 如果正在拖拽节点
      if (this.dragEvent.target) {
        // 如果拖动的是根节点 拖动整棵树
        if (this.dragEvent.target?.id === this.renderTree.id) {
          ctx.dashedBox(this.renderTree).stroke();
          ctx.setLineDash([]);
        } else if (this.dragEvent.target?.id === node.id) {
          console.log(node.label);
          // 如果拖动的是其他节点 则遍历到该节点时复制一个相同的节点并根据拖动偏移量计算出新位置并找到离它最近的节点模拟为新的父节点
          const newNode = Object.assign({}, node);
          newNode.x += this.dragEvent.changeX;
          newNode.y += this.dragEvent.changeY;

          ctx.roundRect(newNode).stroke();
          ctx.dashedBox(node).stroke();
          ctx.setLineDash([]);

          // 找到离当前位置最近的节点作为模拟的父节点并绘制连线
          const parent = this.getCloseNode(newNode);
          this.dragEvent.parent = parent;
          ctx.beginPath();
          ctx.moveTo(parent.x + parent.width, parent.y + this.nodeAttrs.height / 2);
          ctx.lineTo(newNode.x, newNode.y + this.nodeAttrs.height / 2);
          ctx.stroke();
        }
      }

      // 绘制节点 
      ctx.roundRect(node).stroke();

      const children = node?.children;
      if (!children || children.length === 0) return;
      for (let child of children) {
        const start = { x: node.x + node.width, y: node.y + this.nodeAttrs.height / 2 };
        const end = { x: child.x, y: child.y + this.nodeAttrs.height / 2 }
        ctx.ligature({ start, end }).stroke();
        this.renderNodes(child)
      }
    },
    // 渲染按钮
    buttonRender() {
      if (!this.editNode.showButton) return;
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      ctx.circle(this.editNode.addButton).stroke();
      ctx.circle(this.editNode.deleteButton).stroke();
    },
    // 清空按钮状态
    resetState() {
      this.editNode = {
        target: null,
        showButton: false,
        showInput: false,
        addButton: {
          label: undefined,
          id: undefined,
          x: undefined,
          y: undefined,
          r: undefined
        },
        deleteButton: {
          label: undefined,
          id: undefined,
          x: undefined,
          y: undefined,
          r: undefined
        }
      }
    },
    // 画布点击事件转发 (点击对象 点击类型)
    handleCanvasClick(e) {
      const x = e.offsetX;
      const y = e.offsetY;
      const target = this.searchNode(x, y);
      const time = new Date();
      if (target) {
        if (target?.type) {
          if (target.type === 'add') this.addNode(this.editNode.target);
          else if (target.type === 'delete') this.deleteNode(this.editNode.target);
        } else if (target.id == this.preClick.target?.id) {
          // 如果点击的是同一个目标则根据间距判断双击事件
          if (Math.abs(this.preClick.time - time) < 300) this.handleNodeDoubleClick(target);
          else this.handleNodeClick(target);
        } else this.handleNodeClick(target);
      } else {
        // 点击空白区域
        this.handleInvalidClick();
      }
      // 更新上一次的点击对象
      this.preClick.target = target;
      this.preClick.time = time;
    },
    // 根据点击的位置返回被点击的元素
    searchNode(x, y) {
      // 判断按钮
      if (this.editNode.showButton) {
        let button = this.editNode.addButton;
        let circle = {
          x: button.x,
          y: button.y,
          r: button.r
        };
        const coordinate = {
          x,
          y
        }
        if (this.isCircleInclude(circle, coordinate)) return button;
        button = this.editNode.deleteButton;
        circle = {
          x: button.x,
          y: button.y,
          r: button.r
        };
        if (this.isCircleInclude(circle, coordinate)) return button;
      }

      // 判断节点
      for (let node of this.searchArray) {
        const area = {
          xStart: node.x,
          xEnd: node.x + node.width,
          yStart: node.y,
          yEnd: node.y + this.nodeAttrs.height
        }
        if (x >= area.xStart && x <= area.xEnd && y >= area.yStart && y <= area.yEnd) return node;
      }

      // 没点击任何元素
      return null;
    },
    // 判断坐标是否在圆内
    isCircleInclude(circle, coordinate) {
      // 圆心坐标和半径
      const { x, y, r } = circle;
      //首先应该在正方形区域内
      if (coordinate.x >= x - r && coordinate.x <= x + r && coordinate.y >= y - r && coordinate.y <= y + r) {
        const absX = Math.abs(x - coordinate.x);
        const absY = Math.sqrt(r * r - absX * absX);
        if (absY <= r) return true;
      }
      return false
    },
    // 点击空白区域
    handleInvalidClick() {
      this.resetState();
      this.editNode.showInput = false;
      this.render();
    },
    // 节点被单击
    handleNodeClick(node) {
      const addButton = {
        label: `${node.label}的添加按钮`,
        id: `${node.label}的添加按钮`,
        x: node.x + node.width + 15,
        y: node.y + this.nodeAttrs.height / 2,
        r: 8,
        type: 'add'
      }
      const deleteButton = {
        label: `${node.label}的删除按钮`,
        id: `${node.label}的删除按钮`,
        x: node.x - 15,
        y: node.y + this.nodeAttrs.height / 2,
        r: 8,
        type: 'delete'
      }
      this.editNode = {
        target: node,
        addButton,
        deleteButton,
        showButton: true,
        showInput: false
      }
      this.render()
    },
    // 节点被双击
    handleNodeDoubleClick(node) {
      // 当前节点进入编辑状态
      this.editNode.target = node;
      this.editNode.showButton = false;
      this.editNode.showInput = true;
      this.getInputAttr(node);
      this.render();
    },
    // 计算Input的属性
    getInputAttr(node) {
      const input = this.$refs.input;
      input.value = node.label;
      this.input.top = node.y + 9;
      this.input.left = node.x + 13;
      this.input.fontSize = 24;
      this.input.height = 30;
      this.input.width = node.width - 30;
    },
    // 输入回调
    handleInput(e) {
      const input = this.$refs.input;
      this.editNode.target.label = input.value;
      this.editNode.target.id = this.getRandomNodeId(input.value);
      this.render();
    },
    // 捕捉回车
    handleInputChange() {
      // 处理节点名字规范性(不能为空等)
      this.handleInvalidClick();
    },
    // 在一个节点的末位添加一个新节点
    addNode(target) {
      this.resetState();
      const newNode = {
        label: '新节点' + this.searchArray.length,
        id: this.getRandomNodeId('新节点' + this.searchArray.length),
      };
      const ergodicTree = (node) => {
        if (node.id === target.id) {
          if (node.children) node.children.push(newNode)
          else this.$set(node, 'children', [newNode])
          return this.$emit('treeChange', newTree);
        }
        if (node.children) for (let child of node.children) ergodicTree(child)
      }
      const newTree = Object.assign({}, this.tree)
      ergodicTree(newTree);
    },
    // 删除一个节点及其子节点
    deleteNode(target) {
      this.resetState();
      const ergodicTree = (node) => {
        if (!node.children || node.children.length === 0) return;
        for (let idx in node.children) {
          const child = node.children[idx];
          if (child.id === target.id) {
            node.children.splice(idx, 1);
            return this.$emit('treeChange', newTree);
          }
          ergodicTree(child)
        }
      }
      const newTree = Object.assign({}, this.tree)
      ergodicTree(newTree);
    },
    // 随机生成节点id
    getRandomNodeId(label) {
      return label + Date.now() + Math.ceil(Math.random() * 100000);
    },
    // 拖动事件
    mouseDownAndMove(el, callback) {
      const that = this;
      el.addEventListener("mousedown", function () {
        // 当鼠标按下时 添加鼠标移动监听
        document.addEventListener("mousemove", callback);

        // 拖拽事件的开始 
        that.dragEvent.startX = -9999;
        that.dragEvent.startY = -9999;
        // 开启动画 
        that.animating = true;
        that.animation();
      });
      document.addEventListener("mouseup", function () {
        // 当鼠标松开时 移除鼠标移动监听
        document.removeEventListener("mousemove", callback);

        // 拖拽事件的结束 
        // 关闭动画 
        window.cancelAnimationFrame(that.requestAnimation);
        that.animating = false;

        // 更新拖动前的根节点坐标 
        that.rootPreCoordinate.x = that.renderTree.x;
        that.rootPreCoordinate.y = that.renderTree.y;

        // 清理被拖拽子树的状态 
        const ergodicTreeForCleanState = node => {
          if (!node) return;
          node.fillStyle = undefined;
          node.strokeStyle = undefined;
          node.isDragNodeChild = undefined;
          if (node.children && node.children.length !== 0) for (let child of node.children) ergodicTreeForCleanState(child)
        }
        ergodicTreeForCleanState(that.dragEvent.target)

        // 更新被拖拽节点的父节点
        if (that.dragEvent.parent) {
          const newTree = Object.assign({}, that.tree);
          // 现在原来的树中将该节点删除
          const ergodicTreeForDelete = (node) => {
            if (!node.children || node.children.length === 0) return;
            for (let idx in node.children) {
              const child = node.children[idx];
              if (child.id === that.dragEvent.target.id) return node.children.splice(idx, 1);
              ergodicTreeForDelete(child)
            }
          }
          ergodicTreeForDelete(newTree);
          // 找到父节点并在插入在合适的位置
          const ergodicTreeForInsert = node => {
            if (node.id === that.dragEvent.parent.id) {
              // 如果父节点是叶子节点
              if (!node.children || node.children.length === 0) return that.$set(node, 'children', [that.dragEvent.target]);
              else {
                // 遍历子节点找到第一个在鼠标上方的节点
                for (let i = node.children.length - 1; i >= 0; i--) {
                  const child = node.children[i];
                  if (child.y <= that.dragEvent.target.y || i === 0) return node.children.splice(i, 0, that.dragEvent.target);
                }
              }
            }
            if (node.children && node.children.length !== 0) for (let child of node.children) ergodicTreeForInsert(child)
          }
          ergodicTreeForInsert(newTree);
          that.$emit('treeChange', newTree)
        }

        // 清空拖动状态
        that.dragEvent = {
          target: null,
          startX: undefined,
          startY: undefined,
          changeX: undefined,
          changeY: undefined,
          parent: null
        }
      });
    },
    // 拖动事件的回调
    handleMouseDownAndMove(e) {
      if (this.dragEvent.startX === -9999 || this.dragEvent.startY === -9999) {
        this.dragEvent.startX = e.x;
        this.dragEvent.startY = e.y;
        const target = this.searchNode(e.x, e.y);
        if (!target?.type) this.dragEvent.target = target;
      }
      if (!this.dragEvent.target) return;
      this.dragEvent.changeX = e.x - this.dragEvent.startX;
      this.dragEvent.changeY = e.y - this.dragEvent.startY;

    },
    // 开启动画
    animation() {
      const renderAnimation = (timeStamp) => {
        this.render();
        if (this.animating) this.requestAnimation = window.requestAnimationFrame(renderAnimation);
      };
      this.requestAnimation = window.requestAnimationFrame(renderAnimation);
    },
    // 获得一个子树离根节点最远的节点的右侧x坐标 (同时将被拖拽子树的样式改变)
    getFarthestX(node) {
      node.fillStyle = 'gray';
      node.strokeStyle = 'gray';
      node.isDragNodeChild = true;
      if (!node.children || node.children.length === 0) return node.x + node.width;
      let ans = 0;
      for (let child of node.children) ans = Math.max(ans, this.getFarthestX(child));
      return ans;
    },
    // 找到离目标节点最近的节点
    getCloseNode(target) {
      // 找到一个节点 他离当前节点的位置最近且不能是当前节点的子节点(包括自身)
      const x = target.x, y = target.y + this.nodeAttrs.height / 2;
      let minDis = 1e+9, ans = null;
      for (let node of this.searchArray) {
        if (node.isDragNodeChild) continue;

        const coordinate = {
          x: node.x + node.width,
          y: node.y + this.nodeAttrs.height / 2
        };
        const subX = Math.abs(x - coordinate.x);
        const subY = Math.abs(y - coordinate.y);
        const distance = subX * subX + subY * subY;
        if (distance < minDis) {
          ans = node;
          minDis = distance;
        }

      }
      return ans;
    }
  },
  mounted() {
    const that = this;
    // 绘制节点
    CanvasRenderingContext2D.prototype.roundRect = function (arg) {
      const { id, x, y, width, label, fontStyle = "normal 24px 微软雅黑", fillStyle = "black", strokeStyle = "#F00" } = arg;

      // 样式
      this.lineWidth = 2;
      this.strokeStyle = strokeStyle;
      this.font = fontStyle;
      this.fillStyle = fillStyle

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
      if (!that.editNode.showInput || that.editNode.target.id !== id) this.fillText(label, x + 15, y + 35);
      this.closePath();

      return this;
    };
    // 绘制父子节点之间连线
    CanvasRenderingContext2D.prototype.ligature = function (arg) {
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
      this.lineTo(middle.x, end.y);
      this.stroke();

      this.beginPath();
      this.moveTo(middle.x, end.y);
      this.lineTo(end.x, end.y);
      this.stroke();

      return this;
    };
    // 绘制按钮
    CanvasRenderingContext2D.prototype.circle = function (arg) {
      const { x, y, r } = arg;

      this.beginPath(); //创建一个路径
      this.moveTo(x + r, y);
      this.arc(x, y, r, 0, 2 * Math.PI);
      this.closePath();
      this.fill();

      return this;
    };
    // 绘制一个包裹着整个子树的虚线框
    CanvasRenderingContext2D.prototype.dashedBox = function (node) {
      const { x, y, subtreeHeight } = node;
      const height = subtreeHeight + 30;
      const width = that.getFarthestX(node) - node.x + 30;

      // 虚线框左侧边的中点
      const rawCoordinate = {
        x: x - 15,
        y: y + that.nodeAttrs.height / 2
      }
      const corner = [
        {
          x: rawCoordinate.x,
          y: rawCoordinate.y + height / 2
        },
        {
          x: rawCoordinate.x + width,
          y: rawCoordinate.y + height / 2
        },
        {
          x: rawCoordinate.x + width,
          y: rawCoordinate.y - height / 2
        },
        {
          x: rawCoordinate.x,
          y: rawCoordinate.y - height / 2
        }
      ]
      this.setLineDash([10, 10]);
      for (let i = 0; i < corner.length; i++) {
        const start = corner[i];
        const end = corner[(i + 1) % 4];
        this.beginPath();
        this.moveTo(start.x, start.y);
        this.lineTo(end.x, end.y);
        this.stroke();
      }

      return this;
    };

    const canvas = this.$refs.canvas;
    // 绑定拖动事件
    this.mouseDownAndMove(canvas, this.handleMouseDownAndMove);
    this.renderTree = Object.assign({}, this.tree);
    this.render();
  },
  watch: {
    tree: {
      handler(newVal, oldVal) {
        this.renderTree = Object.assign({}, newVal);
        this.render();
      }
    }
  }
}
</script>

<style lang="less" scoped>
.mind {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: auto;

  .input {
    border: 0;
    outline: none;
    background-color: transparent;
  }
}
</style>