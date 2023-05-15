<template>
  <view class="custom-tree-select-content">
    <uni-popup
      v-if="showPopup"
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
          <view
            class="left"
            :style="{ color: cancelTextColor }"
            @click="close()"
          >
            <text>{{ cancelText }}</text>
          </view>
          <view class="center">
            <text>{{ placeholder }}</text>
          </view>
          <view
            class="right"
            :style="{ color: confirmTextColor }"
            @click="done"
          >
            <text>{{ confirmText }}</text>
          </view>
        </view>
        <view v-if="search" class="search-box">
          <uni-easyinput
            :maxlength="-1"
            prefixIcon="search"
            placeholder="搜索"
            v-model="searchStr"
            confirm-type="search"
            @confirm="handleSearch"
          ></uni-easyinput>
          <button
            type="primary"
            size="mini"
            class="search-btn"
            @click="handleSearch"
          >
            搜索
          </button>
        </view>
        <view v-if="treeData.length" class="select-content">
          <scroll-view
            class="scroll-view-box"
            :scroll-top="scrollTop"
            scroll-y="true"
            @touchmove.stop
          >
            <view v-if="!filterTreeData.length" class="no-data center">
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
              :border="border"
            ></data-select-item>
            <view class="sentry" />
          </scroll-view>
        </view>
        <view v-else class="no-data center">
          <text>暂无数据</text>
        </view>
      </view>
    </uni-popup>
  </view>
</template>

<script>
import { paging } from './utils'
import dataSelectItem from './data-select-item.vue'
export default {
  name: 'custom-tree-select',
  components: {
    dataSelectItem
  },
  model: {
    prop: 'value',
    event: 'input'
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
    },
    showChildren: {
      type: Boolean,
      default: true
    },
    border: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      contentHeight: '500px',
      treeData: [],
      filterTreeData: [],
      clearTimerList: [],
      showPopup: false,
      clickOpenTimer: null,
      scrollTop: 0,
      searchStr: '',
      selectedList: []
    }
  },
  watch: {
    listData: {
      deep: true,
      immediate: true,
      handler(newVal) {
        if (newVal) {
          this.treeData = this.initData(newVal)
          if (this.showPopup) {
            this.resetClearTimerList()
            this.renderTree(this.treeData)
          }
        }
      }
    },
    selectedList: {
      deep: true,
      handler(newVal) {
        const ids = newVal
          ? Array.isArray(newVal)
            ? newVal
            : newVal.split(',')
          : []
        this.changeStatus(this.treeData, ids, true)
        this.changeStatus(this.filterTreeData, ids, true)
      }
    }
  },
  mounted() {
    this.getContentHeight(uni.getSystemInfoSync())
  },
  methods: {
    // 搜索完成返回顶部
    goTop() {
      this.scrollTop = 10
      this.$nextTick(() => {
        this.scrollTop = 0
      })
    },
    done() {
      this.$emit('done', this.selectedList)
      this.close()
    },
    // 处理搜索
    handleSearch() {
      this.resetClearTimerList()
      this.renderTree(this.searchValue(this.searchStr, this.treeData))
      this.goTop()
      uni.hideKeyboard()
    },
    // 具体搜索方法
    searchValue(str, arr) {
      const res = []
      arr.forEach((item) => {
        if (item.visible) {
          if (
            item[this.dataLabel].toLowerCase().indexOf(str.toLowerCase()) > -1
          ) {
            res.push(item)
          } else {
            if (item[this.dataChildren]?.length) {
              const data = this.searchValue(str, item[this.dataChildren])
              if (data?.length) {
                if (
                  str &&
                  !item.showChildren &&
                  item[this.dataChildren]?.length
                ) {
                  item.showChildren = true
                }
                res.push({
                  ...item,
                  [this.dataChildren]: data
                })
              }
            }
          }
        }
      })
      return res
    },
    // 更新试图 搜索的数据如果是父子联动有时候不能正常显示
    updateTreeData(arr) {
      if (!arr.length) return
      for (let i = 0; i < arr.length; i++) {
        const truthNode = this.getTruthNode(arr[i])
        if (truthNode) {
          arr[i].checked = truthNode.checked
          if (arr[i][this.dataChildren]?.length) {
            this.updateTreeData(arr[i][this.dataChildren])
          }
        }
      }
    },
    getContentHeight({ screenHeight }) {
      this.contentHeight = `${Math.floor(screenHeight * 0.7)}px`
    },
    // 懒加载
    renderTree(arr) {
      const pagingArr = paging(arr)
      this.filterTreeData.splice(
        0,
        this.filterTreeData.length,
        ...(pagingArr?.[0] || [])
      )
      this.lazyRenderList(pagingArr, 1)
    },
    // 懒加载具体逻辑
    lazyRenderList(arr, startIndex) {
      for (let i = startIndex; i < arr.length; i++) {
        let timer = null
        timer = setTimeout(() => {
          this.filterTreeData.push(...arr[i])
        }, i * 500)
        this.clearTimerList.push(() => clearTimeout(timer))
      }
    },
    // 打开弹窗
    open() {
      this.showPopup = true
      this.$nextTick(() => {
        this.$refs.popup.open()
        this.renderTree(this.treeData)
      })
    },
    // 关闭弹窗
    close() {
      this.$refs.popup.close()
    },
    // 弹窗状态变化 包括点击回显框和遮罩
    change(data) {
      if (data.show) {
        uni.$on('custom-tree-popup-node-click', this.handleNodeClick)
        uni.$on('custom-tree-popup-name-click', this.handleHideChildren)
      } else {
        uni.$off('custom-tree-popup-node-click', this.handleNodeClick)
        uni.$off('custom-tree-popup-name-click', this.handleHideChildren)
        this.resetClearTimerList()
        this.selectedList.splice(0, this.selectedList.length)
        this.searchStr = ''
        if (this.animation) {
          setTimeout(() => {
            this.showPopup = false
          }, 200)
        } else {
          this.showPopup = false
        }
      }
      this.$emit('change', data)
    },
    // 中断懒加载
    resetClearTimerList() {
      const list = [...this.clearTimerList]
      this.clearTimerList.splice(0, this.clearTimerList.length)
      list.forEach((fn) => fn())
    },
    // 点击遮罩
    maskClick() {
      this.$emit('maskClick')
    },
    // 初始化数据
    initData(arr, parentVisible) {
      if (!Array.isArray(arr)) return []

      const res = []

      for (let i = 0; i < arr.length; i++) {
        const obj = {
          [this.dataLabel]: arr[i][this.dataLabel],
          [this.dataValue]: arr[i][this.dataValue]
        }

        obj.checked = this.selectedList.includes(
          arr[i][this.dataValue].toString()
        )

        obj.disabled = Boolean(arr[i].disabled)

        const parentVisibleState =
          parentVisible === undefined ? true : parentVisible
        const curVisibleState =
          arr[i].visible === undefined ? true : Boolean(arr[i].visible)
        if (parentVisibleState === curVisibleState) {
          obj.visible = parentVisibleState
        } else if (!parentVisibleState || !curVisibleState) {
          obj.visible = false
        } else {
          obj.visible = true
        }

        obj.showChildren =
          'showChildren' in arr[i] && arr[i].showChildren != undefined
            ? arr[i].showChildren
            : this.showChildren

        if (arr[i][this.dataChildren]?.length) {
          obj[this.dataChildren] = this.initData(
            arr[i][this.dataChildren],
            obj.visible
          )
        }

        res.push(obj)
      }
      return res
    },
    // 获取某个节点所有子元素
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
    // 获取某个节点所有父元素
    getParentNode(target, arr) {
      let res = []

      for (let i = 0; i < arr.length; i++) {
        if (arr[i][this.dataValue] === target[this.dataValue]) {
          return true
        }

        if (arr[i][this.dataChildren]?.length) {
          const childRes = this.getParentNode(target, arr[i][this.dataChildren])
          if (typeof childRes === 'boolean' && childRes) {
            res = [arr[i]]
          } else if (Array.isArray(childRes) && childRes.length) {
            res = [...childRes, arr[i]]
          }
        }
      }

      return res
    },
    // 获取某个节点所有兄弟元素
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
    // 获取treeData中对应数据
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
    getFilterTreeNode(node) {
      const arr = [...this.filterTreeData]
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
    // 点击checkbox
    handleNodeClick(data) {
      const node = this.getTruthNode(data)
      node.checked = !node.checked
      // 如果是单选不考虑其他情况
      if (!this.mutiple) {
        let emitData = []
        if (node.checked) {
          emitData = [node[this.dataValue].toString()]
        }
        this.selectedList = emitData
      } else {
        // 多选情况
        if (!this.linkage) {
          // 不需要联动
          let emitData = null
          if (node.checked) {
            emitData = Array.from(
              new Set([...this.selectedList, node[this.dataValue].toString()])
            )
          } else {
            emitData = this.selectedList.filter(
              (id) => id !== node[this.dataValue].toString()
            )
          }
          this.selectedList = emitData
        } else {
          // 需要联动
          let emitData = [...this.selectedList]
          const parentNodes = this.getParentNode(node, this.treeData)
          const childrenVal = this.getChildren(node).filter(
            (item) => !item.disabled
          )
          if (node.checked) {
            // 选中
            emitData = Array.from(
              new Set([...emitData, node[this.dataValue].toString()])
            )
            if (childrenVal.length) {
              emitData = Array.from(
                new Set([
                  ...emitData,
                  ...childrenVal.map((item) => item[this.dataValue].toString())
                ])
              )
            }
            if (parentNodes.length) {
              // 有父元素 如果父元素下所有子元素全部选中，选中父元素
              while (parentNodes.length) {
                const item = parentNodes.shift()
                if (!item.disabled) {
                  const allChecked = item[this.dataChildren]
                    .filter((node) => node.visible)
                    .every((node) => node.checked)
                  if (allChecked) {
                    item.checked = true
                    emitData = Array.from(
                      new Set([...emitData, item[this.dataValue].toString()])
                    )
                  } else {
                    break
                  }
                }
              }
            }
          } else {
            // 取消选中
            emitData = emitData.filter(
              (id) => id !== node[this.dataValue].toString()
            )
            if (parentNodes.length) {
              parentNodes.forEach((parentNode) => {
                emitData = emitData.filter(
                  (id) => id !== parentNode[this.dataValue].toString()
                )
              })
            }
            if (childrenVal.length) {
              // 取消选中全部子节点
              childrenVal.forEach((childNode) => {
                emitData = emitData.filter(
                  (id) => id !== childNode[this.dataValue].toString()
                )
              })
            }
          }
          this.selectedList = emitData
        }
      }
    },
    // 点击名称折叠或展开
    handleHideChildren(node) {
      const status = !node.showChildren
      this.getTruthNode(node).showChildren = status
      this.getFilterTreeNode(node).showChildren = status
    },
    // 根据 dataValue 找节点
    changeStatus(list, ids, checkedState) {
      const arr = [...list]
      const data =
        typeof ids === 'string'
          ? ids.split(',')
          : ids.map((item) => item.toString())

      while (arr.length) {
        const item = arr.shift()

        if (data.includes(item[this.dataValue].toString())) {
          this.$set(item, 'checked', checkedState)
        } else {
          this.$set(item, 'checked', !checkedState)
        }

        if (!item.checked) {
          this.isSelectedAll = false
        }

        if (item[this.dataChildren]?.length) {
          arr.push(...item[this.dataChildren])
        }
      }

      return null
    }
  }
}
</script>

<style lang="scss" scoped>
.custom-tree-select-content {
  .popup-content {
    flex: 1;
    background-color: #fff;
    border-top-left-radius: 20px;
    border-top-right-radius: 20px;
    display: flex;
    flex-direction: column;

    .title {
      padding: 8px 3rem;
      border-bottom: 1px solid $uni-border-color;
      font-size: 14px;
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

    .search-box {
      margin: 8px 10px 0;
      background-color: #fff;
      display: flex;
      align-items: center;

      .search-btn {
        margin-left: 10px;
        height: 35px;
        line-height: 35px;
      }
    }

    .select-content {
      margin: 8px 10px;
      flex: 1;
      overflow: hidden;
      position: relative;
    }

    .scroll-view-box {
      touch-action: none;
      flex: 1;
      position: absolute;
      top: 0;
      right: 0;
      bottom: 0;
      left: 0;
    }
    .sentry {
      height: 48px;
    }
  }

  .no-data {
    width: auto;
    color: #999;
    font-size: 12px;
  }

  .no-data.center {
    text-align: center;
  }
}
</style>
