<template>
  <div class="graph">
    <el-transfer
      class="transfer"
      ref="transfer"
      v-if="tempGraphData"
      v-model="selected"
      filterable
      :titles="['所有入图关系标签', '已选入图关系标签']"
      :format="{
          noChecked: ' ',
          hasChecked: ' '
        }"
      :data="tempGraphData"
      @left-check-change="setChecked"
      @right-check-change="setChecked"
      @change="handleChange"
      :filter-method="onTransferFilter"
    >
      <template #default="{ option }">
        <aui-tree
          :data="[option]"
          show-checkbox
          node-key="key"
          default-expand-all
          :expand-on-click-node="false"
          @check-change="(data, node, vm)=>onTreeNodeCheck(data, node, vm,option)"
        >
          <template #default="node">
            <div
              class="tree-title"
              v-if="node.data.children"
            >
              <img src="@/assets/icons/reference-relationship.svg">
              <span>{{node.data.leftText}}</span>
            </div>
            <div
              v-else
              class="tree-title"
            >
              <span
                class="disabled"
                v-if="node.data.disabled"
              >已失效</span>
              <GraphRelationVue
                inline
                arrow
                :leftText="node.data.leftText"
                :rightText="node.data.rightText"
                :relationText="`${node.data.leftText}.${node.data.rightText}`"
              />
            </div>
          </template>
        </aui-tree>
      </template>
      <template #name="{ option }">
        <span class="qwert">{{option}}</span>
      </template>
    </el-transfer>
  </div>

</template>

<script>
import GraphRelationVue from '@/components/common/GraphRelation.vue'


export default {
  components: {
    GraphRelationVue
  },
  data() {
    return {
      graphData: [
        {
          key: '12',
          leftText: 'qw',
          rightText: 'zxc',
          all: false,
          children: [{
            leftText: 'er',
            rightText: 'zxczxc',
            key: '5345'
          }, {
            leftText: 'ty',
            rightText: 'zxczxc',
            key: '535645',
            disabled: false
          }]
        },
        {
          key: '62',
          leftText: 'as',
          rightText: 'qaz',
          all: false,
          children: [{
            key: '53',
            leftText: 'df',
            rightText: 'qazqaz'
          }]
        }
      ], // 完整的树节点，主要用来给展示用树节点做参照
      selected: [], // 选中的父节点
      tempGraphData: null, // 展示用树节点，主要解决同一棵树在左右同时存在
      leftChecked: [], // 完全控制选择勾选项
      rightChecked: [],
      leftLeafChecked: [], // 真正勾选的子节点
      rightLeafChecked: [],
      realSelected: [] // 真正选择的子节点
    }
  },
  created() {
    this.tempGraphData = [...this.graphData]
  },
  methods: {

    setChecked() {
      // 强制接管transfer的选中项
      this.$refs.transfer.leftChecked = this.leftChecked
      this.$refs.transfer.rightChecked = this.rightChecked
    },
    onTreeNodeCheck(data, node, vm, opt) {
      console.log(data, node, vm, opt)
      // all属性很重要，减轻切换时的计算量
      if (data.isLeaf) {
        const direction = this.realSelected.indexOf(data.key) !== -1 ? 'right' : 'left'
        if (data.checked) {
          this[`${direction}LeafChecked`] = [...this[`${direction}LeafChecked`], data.key]
          this[`${direction}Checked`] = [...this[`${direction}Checked`], opt.key]
        } else {
          this[`${direction}LeafChecked`] = this[`${direction}LeafChecked`].filter(item => item !== data.key)
          this[`${direction}Checked`].splice(this[`${direction}Checked`].indexOf(opt.key), 1)
          opt.all = false
        }
        this.setChecked()
      } else {
        opt.all = data.checked
      }
    },
    onTransferFilter(val, tree) {
      if (val) {
        const result = this.checkTreeParams(val, tree)
        // 如果没有被筛选中，则将被选中项取消选中
        if (!result) {
          const direction = this.selected.find(key => key === tree.node) ? 'right' : 'left'
          this[`${direction}Checked`] = this[`${direction}Checked`].filter(key => key !== tree.key)
          this[`${direction}LeafChecked`] = this[`${direction}LeafChecked`].filter(key => tree.children.find(child => child.key !== key))
          this.setChecked()
        }
        return result
      } else {
        return true
      }
    },
    checkTreeParams(val, node) {
      if (node.leftText.includes(val) || node.rightText.includes(val)) {
        return true
      } else if (node.children) {
        for (let i = 0; i < node.children.length; i++) {
          const child = node.children[i]
          if (this.checkTreeParams(val, child)) return true
        }
      }
      return false
    },
    handleChange(value, direction, movedKeys) {
      if (direction === 'right') {
        // 先配置好真正被选中的key
        this.realSelected = [...this.realSelected, ...this.leftLeafChecked]
        Array.from(new Set(movedKeys)).forEach(key => {
          let realkey = key
          let isMock = false
          if (/^temp__left__.*/.test(realkey)) {
            realkey = realkey.split('temp__left__')[1]
            isMock = true
          }
          const item = this.graphData.find(node => node.key === realkey)
          // 判断当前节点是不是虚拟的
          if (isMock) {
            // 如果节点本来就是虚拟的，要做特殊处理
            const leftTempNode = this.tempGraphData.find(node => node.key === key)
            const rightTempNode = this.tempGraphData.find(node => node.key === `temp__right__${realkey}`)
            // 如果此时左侧已经全部选中，则取消虚拟节点，把真实节点挂载上
            if (leftTempNode.all) {
              this.tempGraphData.splice(this.tempGraphData.indexOf(leftTempNode), 2, { ...item })
              this.selected.splice(this.selected.indexOf(key), 2, realkey)
            } else {
              // 如果此时左侧未全部选中，则取消左侧对应节点，加到右侧
              rightTempNode.children.push(...leftTempNode.children.filter(leaf => this.leftLeafChecked.indexOf(leaf.key) !== -1))
              leftTempNode.children = leftTempNode.children.filter(leaf => this.leftLeafChecked.indexOf(leaf.key) === -1)
            }
          } else {
            const tempNode = this.tempGraphData.find(node => node.key === key)
            // 如果不是全部子节点都被选中，则构建虚拟父节点
            if (!tempNode.all) {
              const leftTempNode = {
                ...tempNode, key: `temp__left__${key}`,
                children: tempNode.children.filter(child => this.realSelected.indexOf(child.key) === -1)
              }
              const rightTempNode = {
                ...tempNode,
                key: `temp__right__${key}`,
                children: tempNode.children.filter(child => this.realSelected.indexOf(child.key) !== -1)
              }
              this.tempGraphData.splice(this.tempGraphData.indexOf(tempNode), 1, leftTempNode, rightTempNode)
              this.selected.splice(this.selected.indexOf(key), 1, rightTempNode.key)
            }
            // 如果是全部选中，则将它取消全选，放到右边
            tempNode.all = false
          }
        })
        // 清空左侧勾选
        this.leftChecked = this.leftLeafChecked = []
      } else {
        this.realSelected = this.realSelected.filter(key => this.rightLeafChecked.indexOf(key) === -1)
        Array.from(new Set(movedKeys)).forEach(key => {
          let realkey = key
          let isMock = false
          if (/^temp__right__.*/.test(realkey)) {
            realkey = realkey.split('temp__right__')[1]
            isMock = true
          }
          const item = this.graphData.find(node => node.key === realkey)
          if (isMock) {
            const rightTempNode = this.tempGraphData.find(node => node.key === key)
            const leftTempNode = this.tempGraphData.find(node => node.key === `temp__left__${realkey}`)
            if (rightTempNode.all) {
              this.tempGraphData.splice(this.tempGraphData.indexOf(leftTempNode), 2, { ...item })
              this.selected = this.selected.filter(k => k !== key)
            } else {
              leftTempNode.children.push(...rightTempNode.children.filter(leaf => this.rightLeafChecked.indexOf(leaf.key) !== -1))
              rightTempNode.children = rightTempNode.children.filter(leaf => this.rightLeafChecked.indexOf(leaf.key) === -1)
            }
          } else {
            const tempNode = this.tempGraphData.find(node => node.key === key)
            if (!tempNode.all) {
              const leftTempNode = {
                ...tempNode, key: `temp__left__${key}`,
                children: tempNode.children.filter(child => this.realSelected.indexOf(child.key) === -1)
              }
              const rightTempNode = {
                ...tempNode,
                key: `temp__right__${key}`,
                children: tempNode.children.filter(child => this.realSelected.indexOf(child.key) !== -1)
              }
              this.tempGraphData.splice(this.tempGraphData.indexOf(tempNode), 1, leftTempNode, rightTempNode)
              this.selected = this.selected.filter(k => k !== key)
              this.selected.push(rightTempNode.key)
            }
            tempNode.all = false
          }
        })
        this.rightChecked = this.rightLeafChecked = []
      }
      // 重新设定勾选项
      this.$nextTick(() => this.setChecked())
    }
  }

}
</script>

<style lang="less" scoped>
.graph {
  height: 100%;
  padding: 20px;
  box-sizing: border-box;
}

.transfer {
  display: flex;
  align-items: center;
  height: 100%;
}

.tree-title {
  display: flex;
  font-weight: 700;
  font-size: 13px;
  align-items: center;
  & > img {
    padding: 5px;
  }
  & > .disabled {
    color: #fa8b16;
    font-size: 12px;
    padding: 2px;
    border: 1px solid #fa8b16;
    line-height: 12px;
    margin: 0 5px;
  }
}

/deep/.el-transfer-panel {
  flex: 1;
  height: 100%;
}

/deep/.el-transfer-panel__filter {
  width: calc(100% - 24px) !important;
}

/deep/.el-checkbox__input {
  display: none;
}

/deep/ .el-checkbox__label {
  max-width: 100% !important;
  padding-left: 0 !important;
}

/deep/.el-transfer-panel__item {
  height: auto;
}

/deep/.aui-tree-node__content {
  margin: 2px 0;
  height: auto;
}

// 设定title
/deep/.el-transfer-panel:first-child {
  .el-transfer-panel__header {
    .el-checkbox__label {
      span {
        &::before {
          content: '参考关系';
          display: inline-block;
          background-image: url('~@/assets/icons/reference-relationship.svg');
          background-repeat: no-repeat;
          background-position-y: center;
          padding-left: 18px;
          font-size: 14px;
          margin-right: 5px;
        }

        &::after {
          content: '关系实体';
          display: inline-block;
          background-image: url('~@/assets/icons/relationship-entity.svg');
          background-repeat: no-repeat;
          background-position-y: center;
          padding-left: 18px;
          font-size: 14px;
        }
      }
    }
  }
}
</style>
