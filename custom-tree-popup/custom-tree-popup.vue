<template>
  <view class="custom-tree-select-content">
    <uni-popup
      ref="popup"
      :animation="animation"
      :is-mask-click="isMaskClick"
      :mask-background-color="maskBackgroundColor"
      :background-color="backgroundColor"
      :safe-area="safeArea"
      type="bottom"
      @change="change"
      @maskClick="maskClick"
    >
      <view class="popup-content" :style="{ height: contentHeight }">
        <view class="title">
          <view class="left" :style="{ color: cancelTextColor }" @click="close()">
            <text>{{ cancelText }}</text>
          </view>
          <view class="center">
            <text>{{ placeholder }}</text>
          </view>
          <view class="right" :style="{ color: confirmTextColor }" @click="done">
            <text>{{ confirmText }}</text>
          </view>
        </view>
        <view v-if="treeData.length" class="select-content">
          <view v-show="search" class="search-box">
            <uni-easyinput :maxlength="-1" prefixIcon="search" placeholder="搜索" @input="handleSearch"></uni-easyinput>
          </view>
          <view v-show="!filterTreeData.length" class="no-data" center>
            <text>暂无数据</text>
          </view>
          <data-select-item
            v-for="item in filterTreeData"
            :key="item[dataValue]"
            :node="item"
            :dataLabel="dataLabel"
            :dataValue="dataValue"
            :dataChildren="dataChildren"
            :choseParent="choseParent"
          ></data-select-item>
        </view>
        <view v-else class="no-data" center>
          <text>暂无数据</text>
        </view>
      </view>
    </uni-popup>
  </view>
</template>

<script>
import dataSelectItem from './data-select-item.vue'
export default {
  name: 'custom-tree-select',
  components: {
    dataSelectItem
  },
  props: {
    search: {
      type: Boolean,
      default: false
    },
    animation: {
      type: Boolean,
      default: true
    },
    'is-mask-click': {
      type: Boolean,
      default: true
    },
    'mask-background-color': {
      type: String,
      default: 'rgba(0,0,0,0.4)'
    },
    'background-color': {
      type: String,
      default: 'none'
    },
    'safe-area': {
      type: Boolean,
      default: true
    },
    choseParent: {
      type: Boolean,
      default: true
    },
    placeholder: {
      type: String,
      default: '请选择'
    },
    confirmText: {
      type: String,
      default: '完成'
    },
    confirmTextColor: {
      type: String,
      default: '#007aff'
    },
    cancelText: {
      type: String,
      default: '取消'
    },
    cancelTextColor: {
      type: String,
      default: '#333'
    },
    listData: {
      type: Array,
      default: () => []
    },
    dataLabel: {
      type: String,
      default: 'name'
    },
    dataValue: {
      type: String,
      default: 'id'
    },
    dataChildren: {
      type: String,
      default: 'children'
    },
    linkage: {
      type: Boolean,
      default: false
    },
    mutiple: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      treeData: [],
      filterTreeData: [],
      selectedList: [],
      contentHeight: '500px'
    }
  },
  watch: {
    listData: {
      deep: true,
      immediate: true,
      handler(newVal) {
        if (newVal.length) {
          this.treeData = this.deepCopy(newVal)
          this.initData(this.treeData)
          this.filterTreeData = this.treeData
        }
      }
    },
    selectedList: {
      deep: true,
      handler() {
        this.initData(this.treeData)
        this.updateTreeData(this.filterTreeData)
      }
    }
  },
  mounted() {
    this.getContentHeight(uni.getSystemInfoSync())
  },
  methods: {
    done() {
      this.$emit('done', this.selectedList)
      this.close()
    },
    handleSearch(str) {
      this.filterTreeData = this.searchValue(str, this.treeData)
    },
    searchValue(str, arr) {
      const res = []

      arr.forEach((item) => {
        if (item[this.dataLabel].toLowerCase().indexOf(str.toLowerCase()) > -1) {
          res.push(item)
        } else {
          if (item[this.dataChildren]?.length) {
            const data = this.searchValue(str, item[this.dataChildren])
            if (data?.length) {
              res.push({
                ...item,
                [this.dataChildren]: data
              })
            }
          }
        }
      })

      return res
    },
    updateTreeData(arr) {
      if (arr.length) {
        for (let i = 0; i < arr.length; i++) {
          arr[i].checked = this.getTruthNode(arr[i]).checked

          if (arr[i][this.dataChildren]?.length) {
            this.updateTreeData(arr[i][this.dataChildren])
          }
        }
      }
    },
    getContentHeight({ screenHeight = 600 }) {
      this.contentHeight = `${Math.floor(screenHeight * 0.7)}px`
    },
    open() {
      if (this.disabled) return
      this.$refs.popup.open()
    },
    close() {
      this.$refs.popup.close()
    },
    change(data) {
      this.filterTreeData = this.treeData
      this.selectedList.splice(0, this.selectedList.length)
      this.$emit('change', data)
    },
    maskClick() {
      this.$emit('maskClick')
    },
    deepCopy(target) {
      let copyed_objs = [] //此数组解决了循环引用和相同引用的问题，它存放已经递归到的目标对象
      function _deepCopy(target) {
        if (typeof target !== 'object' || !target) {
          return target
        }
        for (let i = 0; i < copyed_objs.length; i++) {
          if (copyed_objs[i].target === target) {
            return copyed_objs[i].copyTarget
          }
        }
        let obj = {}
        if (Array.isArray(target)) {
          obj = [] //处理target是数组的情况
        }
        copyed_objs.push({ target: target, copyTarget: obj })
        Object.keys(target).forEach((key) => {
          if (obj[key]) {
            return
          }
          obj[key] = _deepCopy(target[key])
        })
        return obj
      }
      return _deepCopy(target)
    },
    initData(arr) {
      for (let i = 0; i < arr.length; i++) {
        if (this.selectedList.includes(arr[i][this.dataValue].toString())) {
          this.$set(arr[i], 'checked', true)
        } else {
          this.$set(arr[i], 'checked', false)
        }
        if (arr[i].disabled) {
          this.$set(arr[i], 'disabled', true)
        } else {
          this.$set(arr[i], 'disabled', false)
        }
        if (JSON.stringify(arr[i].visible) === 'false') {
          this.$set(arr[i], 'visible', false)
          if (arr[i][this.dataChildren]?.length) {
            for (let j = 0; j < arr[i][this.dataChildren].length; j++) {
              arr[i][this.dataChildren][j].visible = false
            }
          }
        } else {
          this.$set(arr[i], 'visible', true)
        }
        this.$set(arr[i], 'showChildren', arr[i].showChildren ?? true)
        if (!arr[i].handleNodeClick) {
          this.$set(arr[i], 'handleNodeClick', this.handleNodeClick)
        }
        if (!arr[i].handleHideChildren) {
          this.$set(arr[i], 'handleHideChildren', this.handleHideChildren)
        }
        if (arr[i][this.dataChildren]?.length) {
          this.initData(arr[i][this.dataChildren])
        }
      }
    },
    getChildren(node) {
      if (!node[this.dataChildren]?.length) return []
      const res = node[this.dataChildren].reduce((pre, val) => {
        if (val.visible) {
          return [...pre, val]
        }
        return pre
      }, [])

      for (let i = 0; i < node[this.dataChildren].length; i++) {
        res.push(...this.getChildren(node[this.dataChildren][i]))
      }

      return res
    },
    getParentNode(target, arr) {
      let res = []

      for (let i = 0; i < arr.length; i++) {
        if (arr[i][this.dataValue] === target[this.dataValue]) {
          return [arr[i]]
        }

        if (arr[i][this.dataChildren]?.length) {
          const result = this.getParentNode(target, arr[i][this.dataChildren])
          if (result.length && arr[i].visible) {
            res = [...result, arr[i]]
          }
        }
      }
      return res
    },
    getContiguousNodes(target, arr) {
      for (let i = 0; i < arr.length; i++) {
        if (arr[i][this.dataValue] === target[this.dataValue]) {
          return arr.reduce((pre, val) => {
            if (val.visible) {
              return [...pre, val]
            }
            return pre
          }, [])
        }

        if (arr[i][this.dataChildren]?.length) {
          const res = this.getContiguousNodes(target, arr[i][this.dataChildren])
          if (res.length) return res
        }
      }

      return []
    },
    allChecked(selectedList, arr) {
      for (let i = 0; i < arr.length; i++) {
        if (!selectedList.includes(arr[i][this.dataValue].toString())) {
          return false
        }
      }
      return true
    },
    getTruthNode(node) {
      const arr = [...this.treeData]

      while (arr.length) {
        const item = arr.shift()

        if (item[this.dataValue] === node[this.dataValue]) {
          return item
        }

        if (item[this.dataChildren]?.length) {
          arr.push(...item[this.dataChildren])
        }
      }

      return null
    },
    handleNodeClick(node) {
      node = this.getTruthNode(node)
      node.checked = !node.checked
      // 如果是单选不考虑其他情况
      if (!this.mutiple) {
        if (node.checked) {
          this.selectedList = [node[this.dataValue].toString()]
        } else {
          this.selectedList = []
        }
      } else {
        // 多选情况
        if (!this.linkage) {
          // 不需要联动
          let emitData = null
          if (node.checked) {
            emitData = Array.from(new Set([...this.selectedList, node[this.dataValue].toString()]))
          } else {
            emitData = this.selectedList.filter((id) => id !== node[this.dataValue].toString())
          }
          this.selectedList = emitData
        } else {
          // 需要联动
          let emitData = [...this.selectedList]
          let childrenVal = []
          if (node[this.dataChildren]?.length) {
            childrenVal = this.getChildren(node)
              .filter((item) => !item.disabled)
              .map((item) => item[this.dataValue].toString())
          }
          const contiguousNodes = this.getContiguousNodes(node, this.treeData).filter((item) => !item.disabled)
          const [_, ...parentNodes] = this.getParentNode(node, this.treeData)
          if (node.checked) {
            // 选中
            emitData = Array.from(new Set([...emitData, node[this.dataValue].toString()]))
            if (childrenVal.length) {
              // 选中全部子节点
              emitData = Array.from(new Set([...emitData, ...childrenVal]))
            }
            if (parentNodes.length && this.allChecked(emitData, contiguousNodes)) {
              // 有父元素 如果父元素下所有子元素全部选中，选中父元素
              while (parentNodes.length) {
                const item = parentNodes.shift()
                if (!item.disabled) {
                  const children = this.getChildren(item).filter((child) => !child.disabled)
                  if (this.allChecked(emitData, children)) {
                    emitData = Array.from(new Set([...emitData, item[this.dataValue].toString()]))
                  } else {
                    break
                  }
                }
              }
            }
          } else {
            // 取消选中
            emitData = emitData.filter((id) => id !== node[this.dataValue].toString())
            if (childrenVal.length) {
              // 取消选中全部子节点
              childrenVal.forEach((childVal) => {
                emitData = emitData.filter((id) => id !== childVal)
              })
            }
          }
          this.selectedList = emitData
        }
      }
    },
    handleHideChildren(node) {
      if (!node[this.dataChildren]?.length) return
      this.$set(node, 'showChildren', !node.showChildren)
    },
    getName(id) {
      const arr = [...this.treeData]

      while (arr.length) {
        const item = arr.shift()

        if (item[this.dataValue].toString() === id) {
          return item[this.dataLabel]
        }

        if (item[this.dataChildren]?.length) {
          arr.push(...item[this.dataChildren])
        }
      }

      return ''
    }
  }
}
</script>

<style lang="scss" scoped>
.custom-tree-select-content {
  

  .popup-content {
    background-color: #fff;
    border-top-left-radius: 20px;
    border-top-right-radius: 20px;
    display: flex;
    flex-direction: column;

    .title {
      padding: 8px 3rem;
      border-bottom: 1px solid $uni-border-color;
      display: flex;
      justify-content: space-between;
      position: relative;

      .left {
        position: absolute;
        left: 10px;
      }

      .center {
        flex: 1;
        text-align: center;
      }

      .right {
        position: absolute;
        right: 10px;
      }
    }

    .select-content {
      padding: 8px 10px;
      flex: 1;
      overflow-x: hidden;
      overflow-y: auto;
      position: relative;

      .search-box {
        margin-bottom: 10px;
        background-color: #fff;
        box-shadow: 0 0 10px #eee;
        position: sticky;
        top: 0;
        z-index: 1;
      }
    }
  }

  .no-data {
    width: 100%;
    color: #999;
    font-size: 12px;
  }

  .no-data[center=''],
  .no-data[align='center'] {
    text-align: center;
  }
}
</style>