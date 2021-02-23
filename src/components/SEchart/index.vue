<template>
  <div class="s-chart-container"></div>
</template>

<script>
import echarts from 'echarts/lib/echarts'
import { _debounce } from '@/utils'
let elementResizeDetectorMaker = require('element-resize-detector')

// 事件
const EVENTS = [
  'legendselectchanged', // 切换图例的选中状态
  'legendselected', // 选中图例
  'legendunselected', // 取消选中图例
  'legendscroll', // 图例滚动事件
  'legendselectall', // 图例全选后的事件
  'legendinverseselect', // 图例反选后的事件
  'legendscroll', // 图例滚动事件
  'datazoom', // 数据区域缩放后的事件
  'datarangeselected', // 视觉映射组件中，range 值改变后触发的事件
  'timelinechanged', // 时间轴中的时间点改变后的事件
  'timelineplaychanged', // 时间轴中播放状态的切换事件
  'restore', // 重置 option 事件
  'dataviewchanged', // 工具栏中数据视图的修改事件
  'magictypechanged', // 工具栏中动态类型切换的切换事件
  'geoselectchanged', // geo 中地图区域切换选中状态的事件
  'geoselected', // geo 中地图区域选中后的事件
  'geounselected', // geo 中地图区域取消选中后的事件
  'selectchanged', // 原pieselectchanged、mapselectchanged 在数据选中状态发生变化时触发的事件
  'selected', // 原pieselected、mapselected 选中
  'unselected', // 原pieunselected、mapunselected 取消选中
  'pieselectchanged',
  'pieselected',
  'pieunselected',
  'mapselectchanged',
  'mapselected',
  'mapunselected',
  'axisareaselected', // 平行坐标轴 (Parallel)范围选取事件
  'focusnodeadjacency',
  'unfocusnodeadjacency',
  'brush', // “选框正在添加”事件
  'brushEnd', // “选框添加完毕”事件
  'brushselected', // 对外通知当前选中了什么
  'rendered', // 渲染结束事件
  'finished', // 渲染完成事件
  'click', // 点击事件
  'dblclick', // 双击事件
  'mouseover', // 鼠标移入元素
  'mouseout', // 鼠标离开
  'mousemove', // 鼠标移动
  'mousedown', // 鼠标按下
  'mouseup', // 鼠标抬起
  'globalout', //
  'contextmenu', // 鼠标右击
  'highlight', // 高亮事件
  'downplay', // 取消高亮事件
  'globalcursortaken', // brush 组件捕获鼠标 cursor 时触发
  'rendered', // 渲染结束事件
  'finished' // 渲染完成事件
]
// 监听“空白处”的事件
const ZR_EVENTS = [
  'click',
  'mousedown',
  'mouseup',
  'mousewheel',
  'dblclick',
  'contextmenu'
]

const INIT_TRIGGERS = ['theme', 'initOptions', 'autoresize']
const REWATCH_TRIGGERS = ['manualUpdate']

export default {
  props: {
    options: Object, // 实例数据
    theme: [String, Object], // 主题
    initOptions: Object, // 初始化附加参数
    group: String, // 图表的分组，用于联动
    autoresize: Boolean, // 大小是否随着父元素大小改变而改变
    // watchShallow: Boolean, // 是否深度监听
    manualUpdate: Boolean // 在性能敏感（数据量很大）的场景下，我们最好对于 option prop 绕过 Vue 的响应式系统。当将 manual-update prop 指定为 true 且不传入 option prop 时，数据将不会被监听。然后，需要用 ref 获取组件实例以后手动调用 setOption 方法来更新图表
  },
  data() {
    return {
      lastArea: 0,
      chart: null
    }
  },
  watch: {
    group(group) {
      this.chart.group = group
    }
  },
  created() {
    this.initOptionsWatcher()
    INIT_TRIGGERS.forEach(prop => {
      this.$watch(prop, () => {
        this.refresh()
      }, { deep: true })
    })

    REWATCH_TRIGGERS.forEach(prop => {
      this.$watch(prop, () => {
        this.initOptionsWatcher()
        this.refresh()
      })
    })
  },
  mounted() {
    if (this.options) {
      this.init()
    }
  },
  activated() {
    if (this.autoresize) {
      this.chart && this.chart.resize()
    }
  },
  methods: {
    // 初始化
    init(options) {
      if (this.chart) {
        return
      }
      this.chart = echarts.init(this.$el, this.theme, this.initOptions)
      if (this.group) {
        this.chart.group = this.group
      }
      this.chart.setOption(options || this.manualOptions || this.options || {}, true)
      EVENTS.forEach(event => {
        this.chart.on(event, params => {
          this.$emit(event, params)
        })
      })
      ZR_EVENTS.forEach(event => {
        this.chart.getZr().on(event, params => {
          this.$emit(`zr:${event}`, params)
        })
      })
      try {
        if (this.autoresize) {
          let myObserver = new ResizeObserver(entries => {
            entries.forEach(entry => {
              _debounce(() => {
                this.resize()
              }, 100)
            })
          })
          myObserver.observe(this.$el)
          this.$once('hook:beforeDestroy', () => {
            myObserver.unobserve(this.$el)
          })
        }
      } catch (err) {
        this.resizeWatch()
      }
    },
    setOption(options, notMerge, lazyUpdate) {
      if (this.manualUpdate) {
        this.manualOptions = options
      }
      if (!this.chart) {
        this.init(options)
      } else {
        this.delegateMethod('setOption', options, notMerge, lazyUpdate)
      }
    },
    // 初始化选项观察程序
    initOptionsWatcher() {
      if (this.__unwatchOptions) {
        this.__unwatchOptions()
        this.__unwatchOptions = null
      }
      if (!this.manualUpdate) {
        this.__unwatchOptions = this.$watch('options', (val, oldVal) => {
          if (!this.chart && val) {
            this.init()
          } else {
            // 改变“选项”将导致合并 用新的引用替换它将导致不合并
            this.chart.setOption(val, val !== oldVal)
          }
        }, { deep: true })
      }
    },
    // 在大数据量（百万以上）的渲染场景，分片加载数据和增量渲染
    appendData(params) {
      this.delegateMethod('appendData', params)
    },
    // 大小改变调用
    resize(options) {
      this.delegateMethod('resize', options)
    },
    // 触发图表行为
    dispatchAction(payload) {
      this.delegateMethod('dispatchAction', payload)
    },
    // 转换坐标系上的点到像素坐标值
    convertToPixel(finder, value) {
      return this.delegateMethod('convertToPixel', finder, value)
    },
    // 转换像素坐标值到逻辑坐标系上的点
    convertFromPixel(finder, value) {
      return this.delegateMethod('convertFromPixel', finder, value)
    },
    // 判断给定的点是否在指定的坐标系或者系列上
    containPixel(finder, value) {
      return this.delegateMethod('containPixel', finder, value)
    },
    // 显示加载动画效果
    showLoading(type, options) {
      this.delegateMethod('showLoading', type, options)
    },
    // 隐藏加载动画效果
    hideLoading() {
      this.delegateMethod('hideLoading')
    },
    // 导出图表图片，返回一个 base64 的 URL，可以设置为Image的src
    getDataURL(options) {
      return this.delegateMethod('getDataURL', options)
    },
    // 导出联动的图表图片，返回一个 base64 的 url，可以设置为Image的src。导出图片中每个图表的相对位置跟容器的相对位置有关。
    getConnectedDataURL(options) {
      return this.delegateMethod('getConnectedDataURL', options)
    },
    // 清空当前实例，会移除实例中所有的组件和图表
    clear() {
      this.delegateMethod('clear')
    },
    // 销毁实例，实例销毁后无法再被使用。
    dispose() {
      this.delegateMethod('dispose')
    },
    // 委托代理方法
    delegateMethod(name, ...args) {
      if (!this.chart) {
        this.init()
      }
      return this.chart[name](...args)
    },
    // 销毁
    destroy() {
      this.dispose()
      this.chart = null
    },
    // 刷新
    refresh() {
      if (this.chart) {
        this.destroy()
        this.init()
      }
    },
    // 监听父元素大小改变
    resizeWatch() {
      let erd = elementResizeDetectorMaker()
      erd.listenTo(this.$el, () => {
        if (this.chart) {
          this.resize()
        }
      })
      this.$once('hook:beforeDestroy', () => {
        erd.uninstall(this.$el)
      })
    }
  },
  destroyed() {
    if (this.chart) {
      this.destroy()
    }
  },
  // 多个图表实例实现联动
  connect(group) {
    if (typeof group !== 'string') {
      group = group.map(chart => chart.chart)
    }
    echarts.connect(group)
  },
  // 解除图表实例的联动，如果只需要移除单个实例，可以将通过将该图表实例 group 设为空
  disconnect(group) {
    echarts.disConnect(group)
  },
  // 注册主题，用于初始化实例的时候指定。
  registerMap(mapName, geoJSON, specialAreas) {
    echarts.registerMap(mapName, geoJSON, specialAreas)
  },
  registerTheme(name, theme) {
    echarts.registerTheme(name, theme)
  },
  graphic: echarts.graphic
}
</script>

<style lang="scss" scoped>
.s-chart-container {
  width: 100%;
  height: 100%;
}
</style>

