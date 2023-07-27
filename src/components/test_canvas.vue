<template>
  <canvas
    width="1426px"
    height="800px"
    ref="canvas"
    style="margin-top: 100px"
  />
</template>

<script>
export default {
  name: "test_canvas",
  data() {
    return {
      canvasTree: {
        label: "我是根节点",
        id: 1,
        children: [
          {
            label: "苹果",
            id: 2,
            children: [
              {
                label: "西瓜",
                id: 6,
                children: [
                  {
                    label: "奇怪的水果",
                    id: 10,
                  },
                ],
              },
              {
                label: "西红",
                id: 7,
              },
            ],
          },
          {
            label: "香蕉",
            id: 3,
          },
          {
            label: "橙子",
            id: 4,
          },
          {
            label: "苹果",
            id: 32,
            children: [
              {
                label: "西瓜",
                id: 36,
                children: [
                  {
                    label: "奇怪的水果",
                    id: 310,
                  },
                  {
                    label: "奇怪的火龙果",
                    id: 311,
                  },
                ],
              },
              // {
              //   label: "西红",
              //   id: 47,
              // },
            ],
          },
          {
            label: "火龙果",
            id: 5,
          },
        ],
      },
      nodeArray: [],
      nodeGap: 30,
      nodeHeight: 50,
      nodePaddingHorizen: 15,
      clickEvent: {
        node: null,
        time: new Date(),
      },
      requestAnimation: undefined,
    };
  },
  methods: {
    handleClick(e) {
      const x = e.clientX;
      const y = e.clientY;
      const callback = (node) => {
        this.render();
        if (node?.isButton && node.isButton) {
          console.log("添加节点");
          this.searchNodeInTree(this.canvasTree, node.follow, (node) => {
            const newNode = {
              label: "我是新节点",
              id: 12321312,
            };
            console.log("找到了", node.label);
            if (node.children && node.children.length !== 0) {
              node.children.push(newNode);
            } else {
              this.$set(node, "children", [newNode]);
            }
          });
          this.render();
          return;
        }
        const oldNode = this.clickEvent.node;
        const oldTime = this.clickEvent.time;
        this.clickEvent.node = node;
        this.clickEvent.time = new Date();
        if (oldNode === node && this.clickEvent.time - oldTime <= 300) {
          console.log("双击了节点", node.label, this.clickEvent.time - oldTime);
        } else {
          console.log("单击了节点", node.label, this.clickEvent.time - oldTime);

          // 给节点添加一个按钮
          const canvas = this.$refs.canvas;
          const ctx = canvas.getContext("2d");
          const arg = {
            x: node.x + node.width + 25,
            y: node.y + this.nodeHeight / 2,
            r: 10,
          };
          ctx.circle(arg).stroke();
          const res = this.nodeArray.find((x) => {
            if (x.follow && x.follow == node.id) return true;
            return false;
          });
          if (!res) {
            this.nodeArray.push({
              label: node.label + "的添加按钮",
              x: arg.x,
              y: arg.y - arg.r,
              width: arg.r * 2,
              height: arg.r * 2,
              isButton: true,
              follow: node.id,
            });
          }
        }
      };

      this.findNode(x, y, callback);
    },
    // 在画布上找到这个坐标对应的元素并执行相应的回调
    findNode(x, y, callback = () => {}) {
      const clientRect = this.$refs.canvas.getBoundingClientRect();
      const canvasX = clientRect.x;
      const canvasY = clientRect.y;
      for (let node of this.nodeArray) {
        const nodeX = {
          start: canvasX + node.x,
          end: canvasX + node.x + node.width,
        };
        const nodeY = {
          start: canvasY + node.y,
          end: canvasY + node.y + node.height,
        };
        if (
          x >= nodeX.start &&
          x <= nodeX.end &&
          y >= nodeY.start &&
          y <= nodeY.end
        ) {
          callback(node);
        }
      }
    },
    // 递归获得每个子树的高度
    getTreeHeight(node) {
      if (!node.children || node.children.length === 0) {
        node.height = 50;
        return 50;
      }
      let height = 0;
      for (let child of node.children)
        height += this.getTreeHeight(child) + this.nodeGap;
      height -= this.nodeGap;
      node.height = height;
      return height;
    },
    // 计算子节点的坐标
    getNodeCoordinate(node) {
      const { label, fontStyle = "normal 24px 微软雅黑" } = node;
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");

      // 计算宽度
      ctx.font = fontStyle;
      let width = 0;
      for (let char of label) width += ctx.measureText(char).width;
      node.width = width + this.nodePaddingHorizen * 2;
      if (!node.children || node.children.length === 0) return;
      const len = node.children.length;

      let x = node.x + 150 + node.width,
        y = len === 1 ? node.y : node.y + this.nodeHeight / 2 - node.height / 2;

      for (let i = 0; i < len; i++) {
        const child = node.children[i];
        let preHeight = 0;
        if (i != 0) {
          preHeight = node.children[i - 1].height;
          y += preHeight;
        }
        // 从上到下开始为节点赋值(x不变 y加上子树高度 + gap)
        child.x = x;
        child.y = y;
        y += this.nodeGap;
        this.getNodeCoordinate(child);
      }
    },
    renderTree() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      const func = (node) => {
        this.nodeArray.push(node);
        // 画出当前节点
        ctx.roundRect(node).stroke();

        const start = {
          x: node.x + node.width,
          y: node.y + this.nodeHeight / 2,
        };
        const middle = {
          x: start.x + 50,
          y: start.y,
        };

        ctx.beginPath();
        ctx.moveTo(start.x, start.y); //起始位置
        ctx.lineTo(middle.x, middle.y); //停止位置
        ctx.stroke();

        if (node.children && node.children.length !== 0) {
          for (let child of node.children) {
            func(child);
            // 子节点和父节点直接的连线
            const end = {
              x: child.x,
              y: child.y + this.nodeHeight / 2,
            };
            ctx.bezier({ middle, end }).stroke();
          }
        }
      };
      func(this.canvasTree);
    },
    render() {
      console.log(this.canvasTree);
      const canvas = this.$refs.canvas;
      canvas.removeEventListener("click", this.handleClick);
      this.clearCanvas();
      canvas.addEventListener("click", this.handleClick);
      this.canvasTree.x = 50;
      this.canvasTree.y = 400;
      this.getTreeHeight(this.canvasTree);
      this.getNodeCoordinate(this.canvasTree);
      // this.renderTree();
      for (let node of this.nodeArray) {
        console.log("node", node);

        ctx.roundRect(node).stroke();
      }
    },
    clearCanvas() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      this.nodeArray.length = 0;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    },
    animation() {
      const renderAnimation = (timeStamp) => {
        this.render();
        this.requestAnimation = window.requestAnimationFrame(renderAnimation);
      };
      window.requestAnimationFrame(renderAnimation);
    },
    searchNodeInTree(node, id, callback) {
      if (node.id == id) {
        return callback(node);
      }
      if (node.children) {
        for (let child of node.children) {
          this.searchNodeInTree(child, id, callback);
        }
      }
    },
  },
  mounted() {
    const that = this;
    CanvasRenderingContext2D.prototype.roundRect = function (arg) {
      const { x, y, width, label, fontStyle = "normal 24px 微软雅黑" } = arg;
      this.lineWidth = 2;
      this.strokeStyle = "#F00";
      this.font = fontStyle;
      const height = that.nodeHeight;
      const radius = 8;
      if (width < 2 * radius) radius = width / 2;
      if (height < 2 * radius) radius = height / 2;
      this.beginPath();
      this.moveTo(x + radius, y);
      this.arcTo(x + width, y, x + width, y + height, radius);
      this.arcTo(x + width, y + height, x, y + height, radius);
      this.arcTo(x, y + height, x, y, radius);
      this.arcTo(x, y, x + width, y, radius);
      this.fillText(label, x + 15, y + 35);
      this.closePath();
      return this;
    };
    CanvasRenderingContext2D.prototype.bezier = function (arg) {
      this.lineWidth = 1;
      this.strokeStyle = "lightblue";
      const { middle, end } = arg;
      this.beginPath(); //创建一个路径
      this.moveTo(middle.x, middle.y); //先将画笔移动到两个切点中的某一个
      this.quadraticCurveTo(middle.x, end.y, end.x, end.y); //开始画弧
      this.stroke(); //描边

      // this.closePath();
      return this;
    };
    CanvasRenderingContext2D.prototype.circle = function (arg) {
      const { x, y, r } = arg;
      this.fillStyle = "purple";
      this.beginPath(); //创建一个路径
      this.moveTo(x + r, y);
      this.arc(x, y, r, 0, 2 * Math.PI);
      // this.stroke();
      this.closePath();
      this.fill();
      return this;
    };
    // this.animation();
    this.render();
  },
  beforeDestroy() {
    window.cancelAnimationFrame(this.requestAnimation);
  },
};
</script>

<style scoped></style>
