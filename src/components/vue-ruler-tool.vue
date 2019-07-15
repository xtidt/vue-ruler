<template>
  <div>
    <div v-show="rulerToggle" id="rulerTool" :style="{width : windowWidth + 'px',height : windowHeight + 'px',position:position}" class="ScaleBox" onselectstart="return false;">
      <div id="levelRuler" class="ScaleRuler_h" @mousedown.stop="horizontalDragRuler">
        <span v-for="(item,index) in xScale" :key="index" :style="{left:index * 50 + 2 + 'px'}" class="n">{{ item.id }}</span>
      </div>
      <div id="verticalRuler" class="ScaleRuler_v" @mousedown.stop="verticalDragRuler">
        <span v-for="(item,index) in yScale" :key="index" :style="{top:index * 50 + 2 + 'px'}" class="n">{{ item.id }}</span>
      </div>
      <div id="levelDottedLine" :style="{top:verticalDottedTop + 'px'}" class="RefDot_h"/>
      <div id="verticalDottedLine" :style="{left:levelDottedLeft + 'px'}" class="RefDot_v"/>
      <!-- 水平参考线 -->
      <div v-for="item in levelLineList" :id="item.id" :title="item.title" :style="{top:item.top+ 'px'}" :key="item.id" class="RefLine_h" @mousedown="dragLevelLine(item.id)"/>
      <!-- 垂直参考线 -->
      <div v-for="item in verticalLineList" :id="item.id" :title="item.title" :style="{left:item.left+ 'px'}" :key="item.id" class="RefLine_v" @mousedown="dragVerticalLine(item.id)"/>
      <div id="content" :style="{left: contentLayout.left + 'px',top: contentLayout.top + 'px'}" style="padding: 18px;">
        <!-- 父级内容区域 -->
        <slot />
      </div>
    </div>
    <!-- tool -->
    <div class="tool" style="border:1px solid black; position:absolute; bottom:0; right: 0; min-width: 100px;">
      <p>工具栏:</p>
      <button type="button" @click="handleToggle">{{rulerToggle? '关闭':'打开'}}</button>
    </div>
  </div>
</template>

<script>
import './ruler-tool.scss';

export default {
  name: 'VueRulerTool',
  components: {},
  props: {
    position: {
      type: String,
      default: 'relative',
      validator: function(val) {
        return ['absolute', 'fixed', 'relative', 'static', 'inherit'].indexOf(val) !== -1
      }
    }, // 规定元素的定位类型
    isHotKey: {
      type: Boolean, default: false
    }, // 热键开关
    isScaleRevise: {
      type: Boolean, default: false
    }, // 刻度修正(根据content进行刻度重置)
    presetLine: {
      type: Array,
      default: () => {
        return [] // { type: 'l', site: 50 }, { type: 'v', site: 180 }
      }
    }, // 预置参考线
    contentLayout: {
      type: Object,
      default: () => {
        return { top: 0, left: 0 }
      }
    }, // 内容部分布局
    parent: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    // 标尺尺寸信息
    rulerStyle() {
      return {
        w: 18,
        h: 18
      }
    }
  },
  data() {
    return {
      windowWidth: 0, // 窗口宽度
      windowHeight: 0, // 窗口高度
      xScale: [], // 水平刻度
      yScale: [], // 垂直刻度
      topSpacing: 0, // 标尺与窗口上间距
      leftSpacing: 0, //  标尺与窗口左间距
      isDrag: false,
      dragFlag: '', // 拖动开始标记，可能值x(从水平标尺开始拖动),y(从垂直标尺开始拖动)
      levelLineList: [], // 生成的水平线列表
      verticalLineList: [], // 生成的垂直线列表
      levelDottedLeft: -999, // 水平虚线位置
      verticalDottedTop: -999, // 垂直虚线位置
      rulerWidth: 0, // 垂直标尺的宽度
      rulerHeight: 0, // 水平标尺的高度
      dragLineId: '', // 被移动线的ID
      keyCode: {
        r: 82
      }, // 快捷键参数
      rulerToggle: true // 标尺辅助线显示开关
    }
  },
  watch: {
  },
  mounted() {
    this.addListeners() // 添加事件监听
    this.init()
    this.quickGeneration(this.presetLine) // 生成预置参考线
    const self = this // 绑定窗口调整大小onresize事件
    window.onresize = function() { // 如果直接使用this,this指向的不是vue实例
      self.xScale = []
      self.yScale = []
      self.init()
    }
  },
  beforeDestroy: function() {
    document.documentElement.removeEventListener('mousemove', this.dottedLineMove, true)
    document.documentElement.removeEventListener('mouseup', this.dottedLineUp, true)
    document.documentElement.removeEventListener('keyup', this.keyboard, true)
    window.onresize = null;
  },
  methods: {
    addListeners() {
      document.documentElement.addEventListener('mousemove', this.dottedLineMove, true)
      document.documentElement.addEventListener('mouseup', this.dottedLineUp, true)
      document.documentElement.addEventListener('keyup', this.keyboard, true)
    },
    init() {
      this.box()
      this.scaleCalc()
    },
    box() {
      if (this.isScaleRevise) { // 根据内容部分进行刻度修正
        const content = document.getElementById('content')
        const contentLeft = content.offsetLeft
        const contentTop = content.offsetTop
        for (let i = 0; i < contentLeft; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentLeft) {
            this.xScale.push({ id: i })
          }
        }
        for (let i = 0; i < contentTop; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentTop) {
            this.yScale.push({ id: i })
          }
        }
      }
      if(this.parent){
        const style = window.getComputedStyle(this.$el.parentNode,null)
        this.windowWidth = parseInt(style.getPropertyValue('width'), 10)
        this.windowHeight = parseInt(style.getPropertyValue('height'), 10)
      }else {
        this.windowWidth = document.documentElement.clientWidth - this.leftSpacing
        this.windowHeight = document.documentElement.clientHeight - this.topSpacing
      }
      this.rulerWidth = document.getElementById('verticalRuler').clientWidth
      this.rulerHeight = document.getElementById('levelRuler').clientHeight
      this.topSpacing = document.getElementById('levelRuler').getBoundingClientRect().y //.offsetParent.offsetTop
      this.leftSpacing =document.getElementById('verticalRuler').getBoundingClientRect().x// .offsetParent.offsetLeft
    }, // 获取窗口宽与高
    scaleCalc() {
      for (let i = 0; i < this.windowWidth; i += 1) {
        if (i % 50 === 0) {
          this.xScale.push({ id: i })
        }
      }
      for (let i = 0; i < this.windowHeight; i += 1) {
        if (i % 50 === 0) {
          this.yScale.push({ id: i })
        }
      }
    }, // 计算刻度并渲染
    newHorizontalLevelLine() {
      this.isDrag = true
      this.dragFlag = 'x'
    }, // 生成一个水平参考线
    newVerticalLine() {
      this.isDrag = true
      this.dragFlag = 'y'
    }, // 生成一个垂直参考线
    dottedLineMove($event) {
      switch (this.dragFlag) {
        case 'x':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'y':
          if (this.isDrag) {
            this.levelDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        case 'l':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'v':
          if (this.isDrag) {
            this.levelDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        default:
          break
      }
    }, // 虚线移动
    dottedLineUp($event) {
      if (this.isDrag) {
        this.isDrag = false
        switch (this.dragFlag) {
          case 'x':
            this.levelLineList.push(
              {
                id: 'levelLine' + this.levelLineList.length + 1,
                title: $event.pageY + 1 - this.topSpacing - this.rulerStyle.h + 'px',
                top: $event.pageY - this.topSpacing + 1
              }
            )
            break
          case 'y':
            this.verticalLineList.push(
              {
                id: 'verticalLine' + this.verticalLineList.length + 1,
                title: $event.pageX + 1 - this.leftSpacing - this.rulerStyle.w + 'px',
                left: $event.pageX - this.leftSpacing + 1
              }
            )
            break
          case 'l':
            if ($event.pageY - this.topSpacing < this.rulerHeight) {
              let Index, id
              this.levelLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.levelLineList.splice(Index, 1, {
                id: id,
                title: -600 + 'px',
                top: -600
              })
            } else {
              let Index, id
              this.levelLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.levelLineList.splice(Index, 1, {
                id: id,
                title: $event.pageY + 1 - this.topSpacing - this.rulerStyle.h + 'px',
                top: $event.pageY - this.topSpacing + 1
              })
            }
            break
          case 'v':
            if ($event.pageX - this.leftSpacing < this.rulerWidth) {
              let Index, id
              this.verticalLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.verticalLineList.splice(Index, 1, {
                id: id,
                title: -600 + 'px',
                left: -600
              })
            } else {
              let Index, id
              this.verticalLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.verticalLineList.splice(Index, 1, {
                id: id,
                title: $event.pageX + 2 - this.leftSpacing - this.rulerStyle.w + 'px',
                left: $event.pageX - this.leftSpacing + 1
              })
            }
            break
          default:
            break
        }
        this.verticalDottedTop = this.levelDottedLeft = -10
      }
    }, // 虚线松开
    horizontalDragRuler() {
      this.newHorizontalLevelLine()
    }, // 水平标尺处按下鼠标
    verticalDragRuler() {
      this.newVerticalLine()
    }, // 垂直标尺处按下鼠标
    dragLevelLine(id) {
      this.isDrag = true
      this.dragFlag = 'l'
      this.dragLineId = id
    }, // 水平线处按下鼠标
    dragVerticalLine(id) {
      this.isDrag = true
      this.dragFlag = 'v'
      this.dragLineId = id
    }, // 垂直线处按下鼠标
    keyboard($event) {
      if (this.isHotKey) {
        switch ($event.keyCode) {
          case this.keyCode.r:
            this.rulerToggle = !this.rulerToggle
            break
        }
      }
    }, // 键盘事件
    quickGeneration(params) {
      if (params !== []) {
        params.forEach(item => {
          switch (item.type) {
            case 'l':
              this.levelLineList.push({
                id: 'levelLine' + this.levelLineList.length + 1,
                title: item.site + 'px',
                top: item.site
              })
              break
            case 'v':
              this.verticalLineList.push({
                id: 'verticalLine' + this.verticalLineList.length + 1,
                title: item.site + 'px',
                left: item.site
              })
              break
            default:
              break
          }
        })
      }
    }, // 快速生成预置参考线
    handleToggle() {
      this.rulerToggle = !this.rulerToggle
    }
  }
}
</script>

