# custom-tree-popup 使用指南

**提示：**使用该插件前确保你已经导入 `uni-popup` `uni-icons` `uni-easyinput` 插件。

当前插件主要作为弹窗来使用，每次打开弹窗都会初始化状态，如果只是作为表单选择器组件此时推荐使用 [`custom-tree-select`](https://ext.dcloud.net.cn/plugin?id=10295) 组件。 

**如果在微信小程序中使用，在 `main.js` 文件中添加以下代码**

```js
// #ifdef MP-WEIXIN
  Vue.prototype.$bus = new Vue()
// #endif
```

## 优势

**💪**：基于 `uni-popup` `uni-icons` `uni-easyinput`插件进行开发，默认样式与 `uni-easyinput` 样式对标。

**⚙**  ：提供更多配置项。

**📦**：开箱即用。

## Props

|        属性名         |  类型   |     默认值      |                 说明                 |
| :-------------------: | :-----: | :-------------: | :----------------------------------: |
|       animation       | Boolean |      ture       |           是否开启弹窗动画           |
|     is-mask-click     | Boolean |      true       |           点击遮罩关闭弹窗           |
| mask-background-color | String  | rgba(0,0,0,0.4) |    蒙版颜色，建议使用 rgba 颜色值    |
|   background-color    | String  |      none       |             主窗口背景色             |
|       safe-area       | Boolean |      true       |          是否适配底部安全区          |
|      choseParent      | Boolean |      true       |            父节点是否可选            |
|        linkage        | Boolean |      false      |           父子节点是否联动           |
|      placeholder      | String  |        -        |       空状态信息提示、弹窗标题       |
|      confirmText      | String  |      完成       |             确定按钮文字             |
|   confirmTextColor    | String  |     #007aff     |           确定按钮文字颜色           |
|      cancelText       | String  |      取消       |             取消按钮文字             |
|    cancelTextColor    | String  |      #333       |           取消按钮文字颜色           |
|       listData        |  Array  |        -        |              展示的数据              |
|       dataLabel       | String  |      name       |       listData中对应数据的key        |
|       dataValue       | String  |       id        |      listData中对应数据的value       |
|     dataChildren      | String  |    children     |  listData中对应数据的children的key   |
|        mutiple        | Boolean |      false      |             是否可以多选             |
|        search         | Boolean |      false      | 是否可以搜索（常用于数据较多的情况） |

## Events

| 事件名称  | 说明                     | 返回值                              |
| --------- | ------------------------ | ----------------------------------- |
| change    | 弹窗组件状态发生变化触发 | e={show: true｜false,type:当前模式} |
| maskClick | 点击遮罩层触发           |                                     |
| done      | 点击完成按钮触发         | 选择的数据组成的数组                |

## 基础使用示例

```vue
<template>
  <!--/pages/index/index-->
	<button @click="$refs.treePopup.open()">打开弹窗</button>
  <custom-tree-popup ref="treePopup" :listData="listData" @done="done"  />
</template>

<script>
export default {
  data() {
    return {
      listData: [
        {
          id: 1,
          name: '城市1',
          children: [
            {
              id: 3,
              name: '街道1',
              children: [
                {
                  id: 4,
                  name: '小区1'
                },
                {
                  id: 5,
                  name: '小区2'
                }
              ]
            }
          ]
        },
        {
          id: 2,
          name: '城市2',
          children: [
            {
              id: 6,
              name: '街道2'
            }
          ]
        },
        {
          id: 7,
          name: '城市3',
          children: [
            {
              id: 8,
              name: '街道1'
            },
            {
              id: 9,
              name: '街道2'
            },
            {
              id: 10,
              name: '街道10'
            }
          ]
        }
      ]
    }
  },
  methods: {
    done(data) {
      console.log(data)
    }
  }
}
</script>
```

