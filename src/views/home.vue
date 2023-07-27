<template>
  <div>
    <!-- <my_canvas :tree="tree"></my_canvas> -->
    <mind :tree="tree" @treeChange="handleTreeChange" />
  </div>
</template>

<script>
import my_canvas from '@/components/my_canvas.vue'
import mind from '@/components/mind.vue'
export default {
  name: "home",
  components: {
    my_canvas,
    mind
  },
  data() {
    return {
      tree: {
        label: '植物',
        id: '植物',
        children: [
          {
            label: '水果',
            id: '水果',
            children: [
              {
                label: '苹果',
                id: '苹果'
              }, {
                label: '香蕉',
                id: '香蕉'
              }, {
                label: '橙子',
                id: '橙子'
              }
            ]
          },
          {
            label: '蔬菜',
            id: '蔬菜',
            children: [
              {
                label: '茄子',
                id: '茄子',
                children: [
                  {
                    label: '茄子1',
                    id: '茄子1',
                    children: [
                      {
                        label: '茄子2',
                        id: '茄子2',
                      },
                      {
                        label: '茄子3',
                        id: '茄子3',
                      },
                      {
                        label: '茄子4',
                        id: '茄子5',
                      }
                    ]
                  }
                ]
              }, {
                label: '南瓜',
                id: '南瓜'
              }
            ]
          }
        ]
      }
    }
  },
  methods: {
    handleTreeChange(newTree) {
      this.tree = newTree
    }
  }
};
</script>

<style scoped></style>
