# ECharts 知识点总结

## 1. 简介

ECharts 是百度开源的 JavaScript 可视化图表库，基于 Canvas，支持折线图、柱状图、饼图、散点图、地图等多种图表，交互丰富、可高度定制。

特点：
- 丰富的图表类型
- 支持大数据量（可配合 dataZoom）
- 响应式、可交互
- 支持主题定制

## 2. 安装与引入

```
# npm 安装
npm install echarts

# CDN 引入
<script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
```

```
// ES 模块按需引入
import * as echarts from 'echarts'
```

## 3. 初始化图表

```
// 准备一个容器 DOM（需指定宽高）
<div id="chart" style="width: 600px; height: 400px;"></div>

// 初始化实例
const chart = echarts.init(document.getElementById('chart'))

// 设置配置项
chart.setOption({
    title: { text: '示例图表' },
    xAxis: { data: ['A', 'B', 'C'] },
    yAxis: {},
    series: [{
        type: 'bar',
        data: [10, 20, 30]
    }]
})
```

## 4. 基础配置项

```
chart.setOption({
    title: {                       // 标题
        text: '主标题',
        subtext: '副标题'
    },
    tooltip: { trigger: 'axis' },  // 提示框
    legend: { data: ['销量'] },    // 图例
    grid: {                        // 绘图区域
        left: '10%',
        right: '10%',
        bottom: '15%'
    },
    xAxis: {},                     // x 轴
    yAxis: {},                     // y 轴
    series: []                     // 系列（图表数据）
})
```

## 5. 折线图 line

用于展示数据的变化趋势。

```
chart.setOption({
    xAxis: {
        type: 'category',              // 类目轴
        data: ['周一', '周二', '周三', '周四', '周五']
    },
    yAxis: {
        type: 'value'                  // 数值轴
    },
    series: [{
        type: 'line',                  // 折线图
        data: [120, 200, 150, 80, 70],
        smooth: true,                  // 平滑曲线
        areaStyle: {},                 // 填充面积
        lineStyle: { color: '#409eff' } // 线条颜色
    }]
})
```

## 6. 柱状图 bar

用于比较不同分类的数据。

```
chart.setOption({
    xAxis: {
        type: 'category',
        data: ['苹果', '香蕉', '橙子']
    },
    yAxis: { type: 'value' },
    series: [{
        type: 'bar',                   // 柱状图
        data: [50, 80, 60],
        barWidth: 30,                  // 柱宽
        itemStyle: { color: '#67C23A' },  // 柱子颜色
        label: { show: true, position: 'top' }  // 显示数值
    }]
})
```

```
// 横向柱状图：交换 xAxis/yAxis
chart.setOption({
    yAxis: { type: 'category', data: ['苹果', '香蕉'] },
    xAxis: { type: 'value' },
    series: [{ type: 'bar', data: [50, 80] }]
})
```

## 7. 饼图 pie

展示各部分占比。

```
chart.setOption({
    series: [{
        type: 'pie',                   // 饼图
        radius: '60%',                 // 半径
        data: [
            { value: 40, name: 'A' },
            { value: 30, name: 'B' },
            { value: 30, name: 'C' }
        ],
        label: {                       // 标签
            formatter: '{b}: {c} ({d}%)'
        }
    }]
})
```

```
// 环形图：内外半径不同
series: [{
    type: 'pie',
    radius: ['40%', '70%'],   // [内半径, 外半径]
    data: [...]
}]
```

## 8. 散点图 scatter

展示两个变量之间的关系。

```
chart.setOption({
    xAxis: { type: 'value' },
    yAxis: { type: 'value' },
    series: [{
        type: 'scatter',               // 散点图
        data: [
            [10, 20], [15, 25], [20, 18], [25, 30]
        ],
        symbolSize: 12                 // 点的大小
    }]
})
```

## 9. 雷达图 radar

多维指标的对比。

```
chart.setOption({
    radar: {                           // 雷达图坐标系
        indicator: [
            { name: '性能', max: 100 },
            { name: '外观', max: 100 },
            { name: '价格', max: 100 }
        ]
    },
    series: [{
        type: 'radar',                 // 雷达图
        data: [{
            value: [80, 70, 90],
            name: '产品A'
        }]
    }]
})
```

## 10. 仪表盘 gauge

展示单个指标的进度。

```
chart.setOption({
    series: [{
        type: 'gauge',                 // 仪表盘
        min: 0,
        max: 100,
        detail: { formatter: '{value}%' },  // 中心数字
        data: [{ value: 75, name: '完成率' }]
    }]
})
```

## 11. 漏斗图 funnel

展示数据逐步过滤的过程。

```
chart.setOption({
    series: [{
        type: 'funnel',                // 漏斗图
        data: [
            { value: 100, name: '访问' },
            { value: 80, name: '点击' },
            { value: 50, name: '下单' },
            { value: 20, name: '支付' }
        ]
    }]
})
```

## 12. 热力图 heatmap

用颜色深浅表示数值大小。

```
chart.setOption({
    xAxis: { type: 'category', data: ['A', 'B', 'C'] },
    yAxis: { type: 'category', data: ['周一', '周二'] },
    visualMap: {                       // 视觉映射
        min: 0,
        max: 10,
        inRange: { color: ['#fff', '#409eff'] }
    },
    series: [{
        type: 'heatmap',               // 热力图
        data: [
            [0, 0, 5],    // [x 索引, y 索引, 值]
            [1, 0, 8],
            [2, 0, 3]
        ]
    }]
})
```

## 13. 数据区域缩放 dataZoom

大数据量时可拖动缩放、滚动查看。

```
chart.setOption({
    dataZoom: [
        {
            type: 'slider',            // 滑块式缩放条
            start: 0,                  // 起始显示比例
            end: 50                    // 结束显示比例
        },
        {
            type: 'inside'             // 鼠标滚轮缩放
        }
    ],
    // ... 其余配置
})
```

## 14. 坐标轴配置

```
chart.setOption({
    xAxis: {
        type: 'category',
        data: [...],
        axisLabel: {                 // 轴标签
            rotate: 30,              // 旋转角度
            interval: 0              // 全部显示
        },
        name: '月份'                 // 轴名称
    },
    yAxis: {
        type: 'value',
        min: 0,                      // 最小值
        max: 100,                    // 最大值
        splitLine: {                 // 网格线
            lineStyle: { type: 'dashed' }
        }
    }
})
```

## 15. 多系列与双 y 轴

```
chart.setOption({
    xAxis: { type: 'category', data: ['1月', '2月', '3月'] },
    yAxis: [
        { type: 'value', name: '销量' },      // 左轴
        { type: 'value', name: '增长率' }     // 右轴
    ],
    series: [
        {
            name: '销量',
            type: 'bar',
            data: [100, 150, 120]
        },
        {
            name: '增长率',
            type: 'line',
            yAxisIndex: 1,          // 使用右轴
            data: [10, 20, 15]
        }
    ],
    legend: { data: ['销量', '增长率'] }
})
```

## 16. tooltip 提示框配置

```
tooltip: {
    trigger: 'axis',          // 触发方式：axis/item
    // 自定义内容
    formatter: function (params) {
        return params[0].name + ': ' + params[0].value
    }
}
```

```
// 饼图用 item 触发
tooltip: {
    trigger: 'item',
    formatter: '{b}: {c} ({d}%)'   // 名称: 值 (百分比)
}
```

## 17. 主题 theme

```
// 使用内置主题
const chart = echarts.init(dom, 'dark')   // 暗色主题

// 自定义主题
const chart = echarts.init(dom, {
    color: ['#409eff', '#67C23A', '#E6A23C']
})
```

## 18. 数据更新 setOption

```
// 首次设置
chart.setOption(option)

// 更新数据（会合并配置）
chart.setOption({
    series: [{ data: [1, 2, 3, 4] }]
})

// 不合并，完全替换
chart.setOption(newOption, true)   // 第二个参数 notMerge
```

## 19. 响应式 resize

窗口大小变化时调整图表尺寸。

```
// 监听窗口变化
window.addEventListener('resize', () => {
    chart.resize()
})

// 指定尺寸
chart.resize({ width: 800, height: 600 })
```

## 20. 事件交互

```
// 点击事件
chart.on('click', function (params) {
    console.log('点击了', params.name, params.value)
})

// 鼠标悬停
chart.on('mouseover', function (params) {
    console.log('悬停', params.name)
})

// 图例切换事件
chart.on('legendselectchanged', function (params) {
    console.log('图例切换', params.name)
})

// 解绑事件
chart.off('click')
```

## 21. 动画配置

```
chart.setOption({
    animation: true,               // 是否开启动画
    animationDuration: 1000,       // 动画时长（ms）
    animationEasing: 'cubicOut',   // 缓动函数
    series: [{
        type: 'bar',
        data: [...],
        animationDelay: function (idx) {
            return idx * 100       // 柱状图逐个延迟出现
        }
    }]
})
```

## 22. 定时刷新

```
// 定时更新数据
setInterval(() => {
    chart.setOption({
        series: [{ data: getNewData() }]
    })
}, 3000)   // 每 3 秒刷新一次
```

## 23. 导出图片

```
// 获取 base64 图片
const img = chart.getDataURL({
    type: 'png',
    pixelRatio: 2,       // 高清
    backgroundColor: '#fff'
})

// 触发下载
const a = document.createElement('a')
a.href = img
a.download = 'chart.png'
a.click()
```

## 24. 自适应容器大小

```
// 容器需有明确的宽高
<div id="chart" style="width: 100%; height: 400px;"></div>

const chart = echarts.init(document.getElementById('chart'))

// 配合 ResizeObserver 自动监听容器变化
new ResizeObserver(() => chart.resize()).observe(
    document.getElementById('chart')
)
```

## 25. 常用颜色与配色

```
// 全局调色板
chart.setOption({
    color: ['#409eff', '#67C23A', '#E6A23C', '#F56C6C', '#909399']
})

// 单个系列颜色
series: [{
    type: 'bar',
    data: [
        { value: 50, itemStyle: { color: '#409eff' } },
        { value: 80, itemStyle: { color: '#67C23A' } }
    ]
}]
```

## 26. 地图

需引入地图数据（echarts/map 或 GeoJSON）。

```
import china from './china.json'
echarts.registerMap('china', china)   // 注册地图

chart.setOption({
    series: [{
        type: 'map',
        map: 'china',
        data: [
            { name: '北京', value: 100 },
            { name: '上海', value: 200 }
        ],
        label: { show: true }         // 显示地名
    }]
})
```

## 27. 在 Vue 中使用

```
// 原生集成
import * as echarts from 'echarts'
import { ref, onMounted } from 'vue'

const chartRef = ref(null)
let chart = null

onMounted(() => {
    chart = echarts.init(chartRef.value)
    chart.setOption({
        xAxis: { data: ['A', 'B', 'C'] },
        yAxis: {},
        series: [{ type: 'bar', data: [1, 2, 3] }]
    })
})
```

```
// 推荐封装成组件，监听数据变化自动更新
// 或使用 vue-echarts 库
import VChart from 'vue-echarts'
```

## 28. 图表选择指南

| 图表类型 | 适用场景 |
|----------|----------|
| 折线图 line | 趋势变化、时间序列 |
| 柱状图 bar | 分类对比 |
| 饼图 pie | 占比构成 |
| 散点图 scatter | 变量关系、分布 |
| 雷达图 radar | 多维指标对比 |
| 仪表盘 gauge | 单一进度指标 |
| 漏斗图 funnel | 转化率分析 |
| 热力图 heatmap | 密度/频率分布 |
| 地图 map | 地理分布 |

## 29. 常用配置速查

| 配置项 | 作用 |
|--------|------|
| title | 标题 |
| tooltip | 提示框 |
| legend | 图例 |
| grid | 绘图区域边距 |
| xAxis / yAxis | 坐标轴 |
| series | 数据系列 |
| dataZoom | 区域缩放 |
| visualMap | 视觉映射 |
| toolbox | 工具栏 |
