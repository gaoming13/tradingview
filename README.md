### TradingView 开发文档

- 版本：1.13 (internal id b235e44c @ 2019-03-20 03:56:17.207031)
- 下载地址：https://github.com/gaoming13/tradingview

#### 图表接收数据的2种方式

1. JS Api (使用推模型实时更新，例如websocket)
2. UDF - Universal Data Feed 通用数据饲料 (使用拉模型、脉冲(pulse)、刷新进行更新)

#### 特殊配置项修改

1. 修改k线图放大程度

```js
// 新版本
// charting_library/static/bundles/library.8f9f4b3bb78c78c71688.js
// 搜索 b=6 修改 k线宽度m_barSpacing
// 搜索 S=5 修改 k线右侧间距m_rightOffset

// 老版本
overrides: {
  'timeScale.m_barSpacing': 100,
  'timeScale.m_rightOffset': 5,
}
```

2. 完全去除Logo

```
// charting_library/static/bundles/library.8f9f4b3bb78c78c71688.js

// ow&&(this._loadImage(b,"tvLogo"),this._updateStrokeColor(),this._model.properties().paneProperties.background.subscribe(this,this._updateStrokeColor),
// 改为
// ow&&(this._model.properties().paneProperties.background.subscribe(this,this._updateStrokeColor),

// e.lineJoin="round",e.strokeText(this._txt,i,z?4:0)),t.save(),
// 改为
// e.lineJoin="round",e.strokeText("",i,z?4:0)),t.save(),

// e.fillStyle=this._fillColor,e.fillText(this._txt,i,z?4:0)),t.save(),
// 改为
// e.fillStyle=this._fillColor,e.fillText("",i,z?4:0)),t.save(),
```

#### Demo

```html
<template>
  <div id="id-kline-chart">
    <!-- 图表 -->
    <div class="chart">
      <div class="TVChartContainer" id="tv_chart_container" />
    </div>
    <!-- 图标蒙层 -->
    <div @click="chartMaskIsShow = false" class="chart-mask" :class="{ 'on': chartMaskIsShow} "></div>
  </div>
</template>

<script>
import { widget } from '../../data/tradingView/charting_library/charting_library.min'
import moment from 'moment-timezone'
import stompClient from '../../utils/stompClient'
import { mapState } from 'vuex'
import bigDecimal from 'js-big-decimal'

// 所有k线数据点[[时间戳], [[数据点]]]
let kLineData = [[], []]
// 定时器:重新计算最高点与最低点标记位定时器
let timerCalcHighLow = null
let shapeHigh = null
let shapeLow = null

export default {
  name: 'TVChartContainer',
  props: ['symbolId', 'symbolBaseUnit', 'symbolQuoteUnit', 'quoteUnitDecimal'],
  watch: {
    symbolId () {
      kLineData = [[], []]
      this.tvWidget.chart().setSymbol(this.symbolId)
    },
  },
  data: () => {
    return {
      // 遮罩是否显示
      chartMaskIsShow: true,
      // 支持图标类型
      chartList: [
        // 分时
        // { resolution: '1',  chartType: 3, name: 'L0901'},
        // 15min
        { resolution: '15',  chartType: 1, name: 'L0903'},
        // 30min
        { resolution: '30',  chartType: 1, name: 'L0904'},
        // 1hour
        { resolution: '60',  chartType: 1, name: 'L0905'},
        // 4hour
        { resolution: '240',  chartType: 1, name: 'L0906'},
        // 1day
        { resolution: '1D',  chartType: 1, name: 'L0902'},
        // 1week
        { resolution: 'W',  chartType: 1, name: 'L0907'},
      ],
      // tvWidget
      tvWidget: null,
      // 当前粒度
      resolution: '60',
      // 当前类型
      chartType: 1,
      // 主题配色
      theme: {
        up: '#00BD9A',
        down: '#FF6960',
        bg: '#02071e',
        grid: '#1e1c2a',
        cross: '#9194A3',
        border: '#4e5b85',
        text: '#5F5D6C',
        areatop: 'rgba(122, 152, 247, .1)',
        areadown: 'rgba(122, 152, 247, .02'
      },
      // 当前订阅
      nowSubscribeId: '',
    }
  },
  computed: {
    ...mapState({
      // 语言
      lang: state => state.lang,
    })
  },
  methods: {
    // 修改图标类型
    changeChart (resolution, chartType) {
      let self = this
      self.resolution = resolution
      self.chartType = chartType
      kLineData = [[], []]
      self.tvWidget.chart().setResolution(resolution, function() {
        self.tvWidget.chart().setChartType(chartType)
      })
    },
    // 获取tradeview粒度转项目粒度
    getResolutionVal (resolution) {
      let interval = ''
      switch (resolution) {
        case '1': interval = '1m'; break;
        case '5': interval = '5m'; break;
        case '15': interval = '15m'; break;
        case '30': interval = '30m'; break;
        case '60': interval = '1h'; break;
        case '240': interval = '4h'; break;
        case 'D': interval = '1d'; break;
        case '1D': interval = '1d'; break;
        case '5D': interval = '5d'; break;
        case 'W': interval = '1w'; break;
        case '1W': interval = '1w'; break;
        case 'M': interval = '1M'; break;
        case '1M': interval = '1M'; break;
      }
      return interval
    },
    // body点击隐藏图表遮罩
    hideChartMask (e) {
      let self = this
      let domKlineChart = document.getElementById('id-kline-chart')
      if (! domKlineChart.contains(e.target)) {
        self.chartMaskIsShow = true
      }
    },
    // 新增k线数据处理(时间戳,最低,最高)
    kLineDataDeal (t, l, h) {
      let index = kLineData[0].indexOf(t)
      if (index > -1) {
        kLineData[1][index] = {l, h}
      } else {
        kLineData[0].push(t)
        kLineData[1].push({l, h})
      }
    },
    // 重新计算最低点最高点
    reCalcHighLow () {
      let self = this
      clearTimeout(timerCalcHighLow)
      timerCalcHighLow = setTimeout(() => {
        let range = self.tvWidget.chart().getVisibleRange()
        let timezoneOffset = new Date().getTimezoneOffset() * 60000
        range.from = range.from * 1000 + timezoneOffset
        range.to = ((range.to <= 0 ? 100000000000 : range.to) * 1000) + timezoneOffset
        let pointMin = { t: 0, l: 100000000000 }
        let pointMax = { t: 0, h: 0 }
        for (let i = 0; i < kLineData[0].length; i++) {
          let t = kLineData[0][i]
          if (t >= range.from && t <= range.to) {
            let v = kLineData[1][i]
            if (v.l < pointMin.l) {
              pointMin.t = t
              pointMin.l = v.l
            }
            if (v.h > pointMax.h) {
              pointMax.t = t
              pointMax.h = v.h
            }
          }
        }
        if (shapeLow !== null) {
          self.tvWidget.chart().removeEntity(shapeLow)
          shapeLow = null
        }
        if (shapeHigh !== null) {
          self.tvWidget.chart().removeEntity(shapeHigh)
          shapeHigh = null
        }
        if (pointMin.l !== pointMax.h) {
          // 创建最低点
          if (pointMin.t > 0) {
            shapeLow = self.tvWidget.chart().createShape({
              time: pointMin.t / 1000,
              channel: 'low'
            }, {
              shape: 'arrow_up',
              lock: true,
              disableSelection: true,
              disableSave: true,
              disableUndo: true,
              text: bigDecimal.round(pointMin.l, self.quoteUnitDecimal, bigDecimal.RoundingModes.FLOOR),
              overrides: {
                color: '#fff',
                fontsize: 10,
              }
            })
          }
          // 创建最高点
          if (pointMax.t > 0) {
            shapeHigh = self.tvWidget.chart().createShape({
              time: pointMax.t / 1000,
              channel: 'high'
            }, {
              shape: 'arrow_down',
              lock: true,
              disableSelection: true,
              disableSave: true,
              disableUndo: true,
              text: bigDecimal.round(pointMax.h, self.quoteUnitDecimal, bigDecimal.RoundingModes.FLOOR),
              overrides: {
                color: '#fff',
                fontsize: 10,
              }
            })
          }
        }
      }, 200)
    },
  },
  mounted() {
    let self = this

    // 远程获取配置数据改为本地写死
    window.Datafeeds.UDFCompatibleDatafeed.prototype._requestConfiguration = function () {
      return new Promise((resolve) => {
        resolve({
          // 如果 datafeed 支持商品查询和人商品解析逻辑
          supports_search: true,
          // 如果 datafeed 只提供所有商品集合的完整信息，并且无法进行商品搜索或单个商品解析
          supports_group_request: false,
          // 是否在K线上显示标记
          supports_marks: false,
          // 是否支持时间刻度标记
          supports_timescale_marks: false,
          // 是否支持服务器时间(unix时间)
          supports_time: true,
          // 交易所数组
          exchanges: [],
          // 商品类型过滤器数组
          symbols_types: [],
          // 服务器支持的周期数组
          supported_resolutions: ['1','5','15','60','120','240','D','2D','3D','1W','W','3W','M','6M']
        })
      });
    }

    // 通过商品名称解析商品信息
    window.Datafeeds.UDFCompatibleDatafeed.prototype.resolveSymbol = function (symbolName, onSymbolResolvedCallback, onResolveErrorCallback) {
      setTimeout(() => {
        let name = (self.symbolBaseUnit + self.symbolQuoteUnit).toUpperCase()
        let description = (self.symbolBaseUnit + '/' + self.symbolQuoteUnit).toUpperCase()
        onSymbolResolvedCallback({
          // 商品名称
          name: name,
          // 标识符
          ticker: self.symbolId,
          // 商品说明
          description: description,
          // 商品交易时间
          session: '24x7',
          // 交易所时区
          timezone: 'Asia/Shanghai',
          // 仪表类型
          type: 'crypto',
          // 商品是否有成交量数据
          has_no_volume: false,
          // 是否具有日内(分钟)历史数据
          has_intraday: true,
          // 是否具有日为单位的历史数据
          has_daily: true,
          // 是否具有以W和M为单位的历史数据
          has_weekly_and_monthly: true,
          // 服务端支持的粒度
          intraday_multipliers: ['1', '5', '15', '30', '60', '240', '1D', '5D', 'W', '1W', '1M'],
          // 客户端支持的粒度(如果服务端不支持大粒度，则用小粒度生成)
          supported_resolutions: ['1', '5', '15', '30', '60', '240', '1D', '5D', 'W', '1W', '1M'],
          // 价格精度
          pricescale: Math.pow(10, self.quoteUnitDecimal),
          minmov: 1,
          minmove2: 0,
        })
      }, 0)
    }

    // 当图表需要知道服务器时间时(如果配置标志supports_time设置为true，则调用此函数)
    window.Datafeeds.UDFCompatibleDatafeed.prototype.getServerTime = function (cb) {
      self.$http.post('/api/market-server-time', {}, {
        timeout: 10000
      }).then(res => {
        if (res.body.code === 0) {
          cb(parseInt(res.body.data.unixTime/1000))
        }
      }, (e) => {
      })
    }

    // 通过日期范围获取历史K线数据
    window.Datafeeds.UDFCompatibleDatafeed.prototype.getBars = function (symbolInfo, resolution, rangeStartDate, rangeEndDate, onResult, onError) {
      let interval = self.getResolutionVal(resolution)
      self.$http.post('/api/market-kline-data', {
        resolution: interval,
        symbol: symbolInfo.ticker,
        from: rangeStartDate * 1000,
        end: rangeEndDate * 1000,
      }, {
        timeout: 10000
      }).then(res => {
        if (res.data.code === 0) {
          let lines = res.data.data
          let bars = []
          lines.forEach(v => {
            bars.push({
              time: v.t,
              open: v.o,
              close: v.c,
              low: v.l,
              high: v.h,
              volume: v.v,
            })
            // 新增k线数据处理
            self.kLineDataDeal(v.t, v.l, v.h)
            // 重新计算最低点最高点
            self.reCalcHighLow()
          })
          onResult(bars, { noData: bars.length ? false : true })
        } else {
          onError('errr')
        }
      }, (e) => {
        onError('errr')
        if (e.status === 0) {
          self.loadingNum--
          // 网络错误
          alert(self.$t('L2009'))
        } else {
          // http-code错误
          alert(self.$t('L2010'))
        }
      })
    }

    // 订阅K线数据
    window.Datafeeds.UDFCompatibleDatafeed.prototype.subscribeBars = function(symbolInfo, resolution, onRealtimeCallback, subscribeUID, onResetCacheNeededCallback){
      let interval = self.getResolutionVal(resolution)
      // 连接socket
      stompClient.connect(() => {
        self.nowSubscribeId = '/topic/'+interval+'Kline_'+symbolInfo.ticker
        // 订阅K线实时数据
        stompClient.subscribe(self.nowSubscribeId, (res) => {
          let data = JSON.parse(res.body)
          let t = Number(moment(data.timeline).format('x'))
          let o = parseFloat(data.open)
          let c = parseFloat(data.last)
          let l = parseFloat(data.low)
          let h = parseFloat(data.high)
          let v = parseFloat(data.volume)
          onRealtimeCallback({
            time: t,
            open: o,
            close: c,
            low: l,
            high: h,
            volume: v,
          })
          // 最新1笔买还是卖
          let trend = parseInt(data.trend)
          // 新增k线数据处理
          self.kLineDataDeal(t, l, h)
          // 重新计算最低点最高点
          self.reCalcHighLow()
        })
      })
    }

    // 取消订阅K线数据
    window.Datafeeds.UDFCompatibleDatafeed.prototype.unsubscribeBars = function(subscriberUID){
      if (self.nowSubscribeId !== '') {
        // 取消订阅K线实时数据
        stompClient.unsubscribe(self.nowSubscribeId)
      }
    }

    let lang = 'en'
    if (self.lang.now.id === 'cn') {
      lang = 'zh'
    }
    const widgetOptions = {
      // 初始商品和间隔
      symbol: self.symbolId,
      // widget的DOM元素id
      container_id: 'tv_chart_container',
      // JavaScript对象的实现接口 JS API 以反馈图表及数据
      datafeed: new window.Datafeeds.UDFCompatibleDatafeed('https://demo_feed.tradingview.com'),
      // 图表的初始时区
      timezone: 'Asia/Shanghai',
      // debug
      debug: false,
      // static文件夹的路径
      library_path: '/data/tradingView/charting_library/',
      // 显示图表是否占用窗口中所有可用的空间
      fullscreen: false,
      // 显示图表是否应使用容器中的所有可用空间，并在调整容器本身大小时自动调整大小
      autosize: true,
      // 图表库的本地化处理
      locale: lang,
      // 间隔
      interval: '60',
      // 定制加载进度条
      loading_screen: {
        backgroundColor: '#040b1d'
      },
      // 主题颜色
      themeName: 'dark',
      custom_css_url: '/data/tradingView/charting_library/static/bundles/mobile.css',
      studies_overrides: {
        // 交易量副图上涨颜色
        'volume.volume.color.0': '#C23027',
        // 交易量副图下跌颜色
        'volume.volume.color.1': '#0D7D47',
      },
      overrides: {
        'volumePaneSize': 'medium', // 交易量副图大小(large, medium, small, tiny)
        'scalesProperties.lineColor': this.theme.text, // 横轴纵轴颜色
        'scalesProperties.textColor': this.theme.text, // 横州纵轴字体颜色
        'paneProperties.background': this.theme.bg, // 面板背景色
        'paneProperties.vertGridProperties.color': this.theme.grid, // 纵轴栅栏颜色
        'paneProperties.horzGridProperties.color': this.theme.grid, // 横轴栅栏颜色
        'paneProperties.crossHairProperties.color': this.theme.cross, // 鼠标十字线颜色
        'paneProperties.legendProperties.showStudyArguments': true,
        'paneProperties.legendProperties.showStudyTitles': true,
        'paneProperties.legendProperties.showStudyValues': true,
        'paneProperties.legendProperties.showSeriesTitle': false,
        'paneProperties.legendProperties.showSeriesOHLC': true,
        'paneProperties.topMargin': 20, // 面板头部间距
        'paneProperties.bottomMargin': 12, // 面板底部间距
        'mainSeriesProperties.candleStyle.upColor': this.theme.up, // K线上涨颜色
        'mainSeriesProperties.candleStyle.downColor': this.theme.down, // K线下跌颜色
        'mainSeriesProperties.candleStyle.drawWick': true, // 是否显示蜡烛心
        'mainSeriesProperties.candleStyle.borderColor': this.theme.border,
        'mainSeriesProperties.candleStyle.borderUpColor': this.theme.up,
        'mainSeriesProperties.candleStyle.drawBorder': true, // 是否显示边框
        'mainSeriesProperties.candleStyle.wickUpColor': this.theme.up,
        'mainSeriesProperties.candleStyle.wickDownColor': this.theme.down,
        'mainSeriesProperties.candleStyle.barColorsOnPrevClose': false,
        'timeScale.m_barSpacing': 100,
        'timeScale.m_rightOffset': 5,
        /*
        timeScale:
        m_barSpacing: 50
        m_rightOffset: 5
        */
      },
      // 禁用功能
      disabled_features: [
        'header_widget',
        'header_widget_dom_node', // 隐藏头部小部件DOM元素
        'compare_symbol', // 上下文菜单.比较/覆盖
        'display_market_status', // 休市
        'header_chart_type', // 图标类型
        'header_compare',
        'header_interval_dialog_button',
        'header_resolutions', // 粒度选择
        'header_screenshot', // 屏幕快照
        'header_symbol_search', // 符号搜索
        'header_undo_redo', // 撤销按钮
        'left_toolbar', // 左侧工具栏
        'legend_context_menu', // 头部上下文菜单
        'show_hide_button_in_legend',
        'show_interval_dialog_on_key_press',
        'snapshot_trading_drawings', // 包含屏幕截图中的订单/位置/执行信号
        'symbol_info', // 商品信息对话框
        'timeframes_toolbar', // 底部工具栏
        'use_localstorage_for_settings', // 允许将用户设置保存到localstorage
        'volume_force_overlay', // 在主数据列的同一窗格上放置成交量指示器
        'header_saveload', // 它不是header_widget的一部分
        'go_to_date', // 允许您使用'Go to'对话框跳转到任意K线
        'edit_buttons_in_legend',
        'symbol_search_hot_key',
        'border_around_the_chart',
        'control_bar',
        'context_menus',
        'chart_zoom', // 允许图表缩放
        // 'chart_scroll', // 允许图表滚动
        'show_chart_property_page', // 关闭此功能会禁用属性
        'countdown', // 在价格标尺上显示倒计时标签
        'study_market_minimized',
        'main_series_scale_menu',
        'display_market_status',
        'remove_library_container_border',
        'chart_property_page_style',
        'property_pages',
        'chart_property_page_scales',
        'chart_property_page_background',
        'source_selection_markers',
      ],
      // 启用功能
      enabled_features: [
        // 'dont_show_boolean_study_arguments', // 是否隐藏指标参数
        'hide_last_na_study_output', // 隐藏最后一次指标输出
        'move_logo_to_main_pane', // 将标志放在主数据列窗格上，而不是底部窗格
        // 'same_data_requery', // 允许您使用相同的商品调用setSymbol来刷新数据
        // 'side_toolbar_in_fullscreen_mode', // 使用此功能，您可以在全屏模式下启用绘图工具栏
        // 'disable_resolution_rebuild', // 显示的时间与DataFeed提供的时间完全一致，而不进行对齐。 如果您希望图表构建一些分辨率，则不建议使用此方法
      ],
    }
    self.tvWidget = new widget(widgetOptions)
    self.tvWidget.onChartReady(function() {
      // 订阅视图切换事件(更新最高点与最低点)
      self.tvWidget.chart().onVisibleRangeChanged().subscribe(null, () => {
        self.reCalcHighLow()
      })
    })

    // 点击body隐藏遮罩
    document.addEventListener('click', self.hideChartMask)
    document.addEventListener('touchstart', self.hideChartMask)
  },
  beforeDestroy () {
    let self = this
    // 取消订阅
    stompClient.unsubscribe(self.nowSubscribeId) // 当前K线图频道

    document.removeEventListener('click', self.hideChartMask)
    document.removeEventListener('touchstart', self.hideChartMask)
  },
}
</script>

<style lang="scss" scoped>
.page03 {
  & > .chart {
    border-radius: 0 0 5px 5px;
    background: #040b1d;
    height: 410px;
    padding-bottom: 5px;
  }
}
.TVChartContainer {
  height: 100%;
}
.chart-mask {
  position: absolute;
  left: 0;
  top: 40px;
  width: 100%;
  height: 410px;
  z-index: 2;
  display: none;
  &.on {
    display: block;
  }
}
</style>
```

#### 参考配置项
```js
TradingView.defaultProperties = {
  chartproperties: {
    timezone: e,
    dataWindowProperties: {
      background: "rgba( 255, 254, 206, 0.2)",
      border: "rgba( 96, 96, 144, 1)",
      font: "Verdana",
      fontBold: !1,
      fontItalic: !1,
      fontSize: 10,
      transparency: 80,
      visible: !0
    },
    paneProperties: {
      background: "#ffffff",
      gridProperties: {
        color: "#e1ecf2",
        style: CanvasEx.LINESTYLE_SOLID
      },
      vertGridProperties: {
        color: "#e1ecf2",
        style: CanvasEx.LINESTYLE_SOLID
      },
      horzGridProperties: {
        color: "#e1ecf2",
        style: CanvasEx.LINESTYLE_SOLID
      },
      crossHairProperties: {
        color: "rgba( 152, 152, 152, 1)",
        style: CanvasEx.LINESTYLE_DASHED,
        transparency: 0,
        width: 1
      },
      topMargin: 5,
      bottomMargin: 5,
      leftAxisProperties: {
        autoScale: !0,
        autoScaleDisabled: !1,
        lockScale: !1,
        percentage: !1,
        percentageDisabled: !1,
        log: !1,
        logDisabled: !1,
        alignLabels: !0
      },
      rightAxisProperties: {
        autoScale: !0,
        autoScaleDisabled: !1,
        lockScale: !1,
        percentage: !1,
        percentageDisabled: !1,
        log: !1,
        logDisabled: !1,
        alignLabels: !0
      },
      legendProperties: {
        showStudyArguments: !0,
        showStudyTitles: !0,
        showStudyValues: !0,
        showSeriesTitle: !0,
        showSeriesOHLC: !0,
        showLegend: !0
      }
    },
    scalesProperties: {
      showLeftScale: !1,
      showRightScale: !0,
      backgroundColor: "#ffffff",
      lineColor: "#555",
      textColor: "#555",
      fontSize: 11,
      scaleSeriesOnly: !1,
      showSeriesLastValue: !0,
      showSeriesPrevCloseValue: !1,
      showStudyLastValue: !1,
      showSymbolLabels: !1,
      showStudyPlotLabels: !1
    },
    mainSeriesProperties: {
      style: c.STYLE_CANDLES,
      esdShowDividends: !0,
      esdShowSplits: !0,
      esdShowEarnings: !0,
      esdShowBreaks: !1,
      esdBreaksStyle: {
        color: "rgba( 235, 77, 92, 1)",
        style: CanvasEx.LINESTYLE_DASHED,
        width: 1
      },
      esdFlagSize: 2,
      showCountdown: !0,
      showInDataWindow: !0,
      visible: !0,
      silentIntervalChange: !1,
      showPriceLine: !0,
      priceLineWidth: 1,
      priceLineColor: "",
      showPrevClosePriceLine: !1,
      prevClosePriceLineWidth: 1,
      prevClosePriceLineColor: "rgba( 85, 85, 85, 1)",
      minTick: "default",
      extendedHours: !1,
      sessVis: !1,
      statusViewStyle: {
        fontSize: 17,
        showExchange: !0,
        showInterval: !0,
        showSymbolAsDescription: !1
      },
      candleStyle: {
        upColor: "#53b987",
        downColor: "#eb4d5c",
        drawWick: !0,
        drawBorder: !0,
        borderColor: "#378658",
        borderUpColor: "#53b987",
        borderDownColor: "#eb4d5c",
        wickColor: "#737375",
        wickUpColor: "#a9cdd3",
        wickDownColor: "#f5a6ae",
        barColorsOnPrevClose: !1
      },
      hollowCandleStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        drawWick: !0,
        drawBorder: !0,
        borderColor: "rgba( 55, 134, 88, 1)",
        borderUpColor: "rgba( 83, 185, 135, 1)",
        borderDownColor: "rgba( 255, 77, 92, 1)",
        wickColor: "rgba( 115, 115, 117, 1)",
        wickUpColor: "rgba( 169, 220, 195, 1)",
        wickDownColor: "rgba( 245, 166, 174, 1)"
      },
      haStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        drawWick: !0,
        drawBorder: !0,
        borderColor: "rgba( 55, 134, 88, 1)",
        borderUpColor: "rgba( 83, 185, 135, 1)",
        borderDownColor: "rgba( 255, 77, 92, 1)",
        wickColor: "rgba( 115, 115, 117, 1)",
        wickUpColor: "rgba( 83, 185, 135, 1)",
        wickDownColor: "rgba( 255, 77, 92, 1)",
        showRealLastPrice: !1,
        barColorsOnPrevClose: !1,
        inputs: {},
        inputInfo: {}
      },
      barStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        barColorsOnPrevClose: !1,
        dontDrawOpen: !1
      },
      lineStyle: {
        color: "rgba( 60, 120, 216, 1)",
        linestyle: CanvasEx.LINESTYLE_SOLID,
        linewidth: 1,
        priceSource: "close",
        styleType: c.STYLE_LINE_TYPE_SIMPLE
      },
      areaStyle: {
        color1: "rgba( 96, 96, 144, 0.5)",
        color2: "rgba( 1, 246, 245, 0.5)",
        linecolor: "rgba( 0, 148, 255, 1)",
        linestyle: CanvasEx.LINESTYLE_SOLID,
        linewidth: 1,
        priceSource: "close",
        transparency: 50
      },
      priceAxisProperties: {
        autoScale: !0,
        autoScaleDisabled: !1,
        lockScale: !1,
        percentage: !1,
        percentageDisabled: !1,
        log: !1,
        logDisabled: !1
      },
      renkoStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        borderUpColor: "rgba( 83, 185, 135, 1)",
        borderDownColor: "rgba( 255, 77, 92, 1)",
        upColorProjection: "rgba( 169, 220, 195, 1)",
        downColorProjection: "rgba( 245, 166, 174, 1)",
        borderUpColorProjection: "rgba( 169, 220, 195, 1)",
        borderDownColorProjection: "rgba( 245, 166, 174, 1)",
        wickUpColor: "rgba( 83, 185, 135, 1)",
        wickDownColor: "rgba( 255, 77, 92, 1)",
        inputs: {
          source: "close",
          boxSize: 3,
          style: "ATR",
          atrLength: 14,
          wicks: !0
        },
        inputInfo: {
          source: {
            name: "Source"
          },
          boxSize: {
            name: "Box size"
          },
          style: {
            name: "Style"
          },
          atrLength: {
            name: "ATR Length"
          },
          wicks: {
            name: "Wicks"
          }
        }
      },
      pbStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        borderUpColor: "rgba( 83, 185, 135, 1)",
        borderDownColor: "rgba( 255, 77, 92, 1)",
        upColorProjection: "rgba( 169, 220, 195, 1)",
        downColorProjection: "rgba( 245, 166, 174, 1)",
        borderUpColorProjection: "rgba( 169, 220, 195, 1)",
        borderDownColorProjection: "rgba( 245, 166, 174, 1)",
        inputs: {
          source: "close",
          lb: 3
        },
        inputInfo: {
          source: {
            name: "Source"
          },
          lb: {
            name: "Number of line"
          }
        }
      },
      kagiStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        upColorProjection: "rgba( 169, 220, 195, 1)",
        downColorProjection: "rgba( 245, 166, 174, 1)",
        inputs: {
          source: "close",
          style: "ATR",
          atrLength: 14,
          reversalAmount: 1
        },
        inputInfo: {
          source: {
            name: "Source"
          },
          style: {
            name: "Style"
          },
          atrLength: {
            name: "ATR Length"
          },
          reversalAmount: {
            name: "Reversal amount"
          }
        }
      },
      pnfStyle: {
        upColor: "rgba( 83, 185, 135, 1)",
        downColor: "rgba( 255, 77, 92, 1)",
        upColorProjection: "rgba( 169, 220, 195, 1)",
        downColorProjection: "rgba( 245, 166, 174, 1)",
        inputs: {
          sources: "Close",
          reversalAmount: 3,
          boxSize: 1,
          style: "ATR",
          atrLength: 14
        },
        inputInfo: {
          sources: {
            name: "Source"
          },
          boxSize: {
            name: "Box size"
          },
          reversalAmount: {
            name: "Reversal amount"
          },
          style: {
            name: "Style"
          },
          atrLength: {
            name: "ATR Length"
          }
        }
      },
      baselineStyle: {
        baselineColor: "rgba( 117, 134, 150, 1)",
        topFillColor1: "rgba( 83, 185, 135, 0.1)",
        topFillColor2: "rgba( 83, 185, 135, 0.1)",
        bottomFillColor1: "rgba( 235, 77, 92, 0.1)",
        bottomFillColor2: "rgba( 235, 77, 92, 0.1)",
        topLineColor: "rgba( 83, 185, 135, 1)",
        bottomLineColor: "rgba( 235, 77, 92, 1)",
        topLineWidth: 1,
        bottomLineWidth: 1,
        priceSource: "close",
        transparency: 50,
        baseLevelPercentage: 50
      }
    },
    symbolWatermarkProperties: {
      color: "rgba( 85, 85, 85, 0)",
      transparency: 100
    },
    chartEventsSourceProperties: {
      visible: !0,
      futureOnly: !0,
      breaks: {
        color: "rgba(85, 85, 85, 1)",
        visible: !1,
        style: CanvasEx.LINESTYLE_DASHED,
        width: 1
      }
    },
    tradingProperties: {
      showPositions: !0,
      showOrders: !0,
      showExecutions: !0,
      extendLeft: !0,
      lineLength: 5,
      lineWidth: 1,
      lineStyle: CanvasEx.LINESTYLE_DASHED
    },
    alertsProperties: {
      labels: {
        visible: !0,
        color: "rgba( 215, 84, 66, 1)",
        highlightColor: "rgba( 255, 255, 51, 1)",
        hoverColor: "rgba( 245, 227, 135, 1)",
        line: {
          visible: !0,
          style: CanvasEx.LINESTYLE_DASHED,
          width: 1
        }
      },
      fakeLabels: {
        visible: !0,
        color: "rgba( 119, 119, 119, 1)",
        line: {
          visible: !0,
          style: CanvasEx.LINESTYLE_DASHED,
          width: 1
        }
      },
      drawingIcon: {
        color: "rgba( 170, 170, 170, 1)"
      }
    },
    editorFontsList: ["Verdana", "Courier New", "Times New Roman", "Arial"],
    volumePaneSize: "large"
  },
  drawings: {
    magnet: !1,
    stayInDrawingMode: !1,
    drawOnAllCharts: !0,
    crossHairColor: "rgba( 183, 183, 183, 1)",
    crossHairStyle: CanvasEx.LINESTYLE_DASHED,
    crossHairWidth: 1
  },
  linetoolorder: {
    singleChartOnly: !0,
    extendLeft: "inherit",
    lineLength: "inherit",
    lineColor: "rgba( 255, 0, 0, 1)",
    lineTransparency: 0,
    lineStyle: "inherit",
    lineWidth: "inherit",
    bodyBorderColor: "rgba( 255, 0, 0, 0)",
    bodyBorderTransparency: 0,
    bodyBackgroundColor: "rgba( 255, 255, 255, 0.75)",
    bodyBackgroundTransparency: 25,
    bodyTextColor: "rgba( 255, 0, 0, 0)",
    bodyTextTransparency: 0,
    bodyFontFamily: "Verdana",
    bodyFontSize: 7,
    bodyFontBold: !0,
    bodyFontItalic: !1,
    quantityBorderColor: "rgba( 255, 0, 0, 0)",
    quantityBorderTransparency: 0,
    quantityBackgroundColor: "rgba( 255, 0, 0, 0.75)",
    quantityBackgroundTransparency: 25,
    quantityTextColor: "rgba( 255, 255, 255, 1)",
    quantityTextTransparency: 0,
    quantityFontFamily: "Verdana",
    quantityFontSize: 7,
    quantityFontBold: !0,
    quantityFontItalic: !1,
    cancelButtonBorderColor: "rgba( 255, 0, 0, 1)",
    cancelButtonBorderTransparency: 0,
    cancelButtonBackgroundColor: "rgba( 255, 255, 255, 0.75)",
    cancelButtonBackgroundTransparency: 25,
    cancelButtonIconColor: "rgba( 255, 0, 0, 1)",
    cancelButtonIconTransparency: 0,
    tooltip: ""
  },
  linetoolposition: {
    singleChartOnly: !0,
    extendLeft: "inherit",
    lineLength: "inherit",
    lineColor: "rgba( 0, 113, 224, 1)",
    lineTransparency: 0,
    lineStyle: "inherit",
    lineWidth: "inherit",
    bodyBorderColor: "rgba( 0, 113, 224, 1)",
    bodyBorderTransparency: 0,
    bodyBackgroundColor: "rgba( 255, 255, 255, 0.75)",
    bodyBackgroundTransparency: 25,
    bodyTextColor: "rgba( 0, 113, 224, 1)",
    bodyTextTransparency: 0,
    bodyFontFamily: "Verdana",
    bodyFontSize: 7,
    bodyFontBold: !0,
    bodyFontItalic: !1,
    quantityBorderColor: "rgba( 0, 113, 224, 1)",
    quantityBorderTransparency: 0,
    quantityBackgroundColor: "rgba( 0, 113, 224, 0.75)",
    quantityBackgroundTransparency: 25,
    quantityTextColor: "rgba( 255, 255, 255, 1)",
    quantityTextTransparency: 0,
    quantityFontFamily: "Verdana",
    quantityFontSize: 7,
    quantityFontBold: !0,
    quantityFontItalic: !1,
    reverseButtonBorderColor: "rgba( 0, 113, 224, 1)",
    reverseButtonBorderTransparency: 0,
    reverseButtonBackgroundColor: "rgba( 255, 255, 255, 0.75)",
    reverseButtonBackgroundTransparency: 25,
    reverseButtonIconColor: "rgba( 0, 113, 224, 1)",
    reverseButtonIconTransparency: 0,
    closeButtonBorderColor: "rgba( 0, 113, 224, 1)",
    closeButtonBorderTransparency: 0,
    closeButtonBackgroundColor: "rgba( 255, 255, 255, 0.75)",
    closeButtonBackgroundTransparency: 25,
    closeButtonIconColor: "rgba( 0, 113, 224, 1)",
    closeButtonIconTransparency: 0,
    tooltip: ""
  },
  linetoolexecution: {
    singleChartOnly: !0,
    direction: "buy",
    arrowHeight: 8,
    arrowSpacing: 1,
    arrowColor: "rgba( 0, 0, 255, 1)",
    arrowTransparency: 0,
    text: "",
    textColor: "rgba( 0, 0, 0, 1)",
    textTransparency: 0,
    fontFamily: "Verdana",
    fontSize: 8,
    fontBold: !1,
    fontItalic: !1,
    tooltip: ""
  },
  linetoolicon: {
    singleChartOnly: !0,
    clonable: !0,
    color: "rgba( 61, 133, 198, 1)",
    size: 40,
    icon: 61536,
    angle: .5 * Math.PI,
    scale: 1
  },
  linetoolbezierquadro: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    fillBackground: !1,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    transparency: 50,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Normal
  },
  linetoolbeziercubic: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    fillBackground: !1,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    transparency: 50,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Normal
  },
  linetooltrendline: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Normal,
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    snapTo45Degrees: !0,
    alwaysShowStats: !1,
    showMiddlePoint: !1,
    showPriceRange: !1,
    showBarsRange: !1,
    showDateTimeRange: !1,
    showDistance: !1,
    showAngle: !1
  },
  linetooltimecycles: {
    clonable: !0,
    linecolor: "rgba(21, 153, 128, 1)",
    linewidth: 1,
    fillBackground: !0,
    backgroundColor: "rgba(106, 168, 79, 0.5)",
    transparency: 50,
    linestyle: CanvasEx.LINESTYLE_SOLID
  },
  linetoolsineline: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID
  },
  linetooltrendangle: {
    singleChartOnly: !0,
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    snapTo45Degrees: !0,
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !0,
    italic: !1,
    alwaysShowStats: !1,
    showMiddlePoint: !1,
    showPriceRange: !1,
    showBarsRange: !1,
    extendRight: !1,
    extendLeft: !1
  },
  linetooldisjointangle: {
    clonable: !0,
    linecolor: "rgba( 18, 159, 92, 1)",
    linewidth: 2,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    fillBackground: !0,
    backgroundColor: "rgba( 106, 168, 79, 0.5)",
    transparency: 50,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Normal,
    font: "Verdana",
    textcolor: "rgba( 18, 159, 92, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    showPrices: !1,
    showPriceRange: !1,
    showDateTimeRange: !1,
    showBarsRange: !1
  },
  linetoolflatbottom: {
    clonable: !0,
    linecolor: "rgba( 73, 133, 231, 1)",
    linewidth: 2,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    fillBackground: !0,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    transparency: 50,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Normal,
    font: "Verdana",
    textcolor: "rgba( 73, 133, 231, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    showPrices: !1,
    showPriceRange: !1,
    showDateTimeRange: !1,
    showBarsRange: !1
  },
  linetoolfibspiral: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID
  },
  linetooldaterange: {
    clonable: !0,
    linecolor: "rgba( 88, 88, 88, 1)",
    linewidth: 1,
    font: "Verdana",
    textcolor: "rgba( 255, 255, 255, 1)",
    fontsize: 12,
    fillLabelBackground: !0,
    labelBackgroundColor: "rgba( 91, 133, 191, 0.9)",
    labelBackgroundTransparency: 30,
    fillBackground: !0,
    backgroundColor: "rgba( 186, 218, 255, 0.4)",
    backgroundTransparency: 60,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    extendTop: !1,
    extendBottom: !1
  },
  linetoolpricerange: {
    clonable: !0,
    linecolor: "rgba( 88, 88, 88, 1)",
    linewidth: 1,
    font: "Verdana",
    textcolor: "rgba( 255, 255, 255, 1)",
    fontsize: 12,
    fillLabelBackground: !0,
    labelBackgroundColor: "rgba( 91, 133, 191, 0.9)",
    labelBackgroundTransparency: 30,
    fillBackground: !0,
    backgroundColor: "rgba( 186, 218, 255, 0.4)",
    backgroundTransparency: 60,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    extendLeft: !1,
    extendRight: !1
  },
  linetooldateandpricerange: {
    clonable: !0,
    linecolor: "rgba( 88, 88, 88, 1)",
    linewidth: 1,
    font: "Verdana",
    textcolor: "rgba( 255, 255, 255, 1)",
    fontsize: 12,
    fillLabelBackground: !0,
    labelBackgroundColor: "rgba( 91, 133, 191, 0.9)",
    labelBackgroundTransparency: 30,
    fillBackground: !0,
    backgroundColor: "rgba( 186, 218, 255, 0.4)",
    backgroundTransparency: 60,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)"
  },
  linetoolriskrewardshort: {
    isShort: !0,
    clonable: !0,
    linecolor: "rgba( 88, 88, 88, 1)",
    linewidth: 1,
    font: "Verdana",
    textcolor: "rgba(255, 255, 255, 1)",
    fontsize: 12,
    fillLabelBackground: !0,
    labelBackgroundColor: "rgba( 88, 88, 88, 1)",
    labelBackgroundTransparency: 0,
    fillBackground: !0,
    stopBackground: "rgba( 255, 0, 0, 0.2)",
    profitBackground: "rgba( 0, 160, 0, 0.2)",
    stopBackgroundTransparency: 80,
    profitBackgroundTransparency: 80,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    compact: !1,
    riskDisplayMode: "percents",
    accountSize: 1e3,
    risk: 25
  },
  linetoolriskrewardlong: {
    isShort: !1,
    clonable: !0,
    linecolor: "rgba( 88, 88, 88, 1)",
    linewidth: 1,
    font: "Verdana",
    textcolor: "rgba(255, 255, 255, 1)",
    fontsize: 12,
    fillLabelBackground: !0,
    labelBackgroundColor: "rgba( 88, 88, 88, 1)",
    labelBackgroundTransparency: 0,
    fillBackground: !0,
    stopBackground: "rgba( 255, 0, 0, 0.2)",
    profitBackground: "rgba( 0, 160, 0, 0.2)",
    stopBackgroundTransparency: 80,
    profitBackgroundTransparency: 80,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    compact: !1,
    riskDisplayMode: "percents",
    accountSize: 1e3,
    risk: 25
  },
  linetoolarrow: {
    clonable: !0,
    linecolor: "rgba( 111, 136, 198, 1)",
    linewidth: 2,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !1,
    leftEnd: _.Normal,
    rightEnd: _.Arrow,
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    alwaysShowStats: !1,
    showMiddlePoint: !1,
    showPriceRange: !1,
    showBarsRange: !1,
    showDateTimeRange: !1,
    showDistance: !1,
    showAngle: !1
  },
  linetoolray: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !0,
    leftEnd: _.Normal,
    rightEnd: _.Normal,
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    alwaysShowStats: !1,
    showMiddlePoint: !1,
    showPriceRange: !1,
    showBarsRange: !1,
    showDateTimeRange: !1,
    showDistance: !1,
    showAngle: !1
  },
  linetoolextended: {
    clonable: !0,
    linecolor: "rgba( 21, 153, 128, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !0,
    extendRight: !0,
    leftEnd: _.Normal,
    rightEnd: _.Normal,
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    alwaysShowStats: !1,
    showMiddlePoint: !1,
    showPriceRange: !1,
    showBarsRange: !1,
    showDateTimeRange: !1,
    showDistance: !1,
    showAngle: !1
  },
  linetoolhorzline: {
    clonable: !0,
    linecolor: "rgba( 128, 204, 219, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    showPrice: !0,
    showLabel: !1,
    text: "",
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    horzLabelsAlign: "center",
    vertLabelsAlign: "top"
  },
  linetoolhorzray: {
    clonable: !0,
    linecolor: "rgba( 128, 204, 219, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    showPrice: !0,
    showLabel: !1,
    text: "",
    font: "Verdana",
    textcolor: "rgba( 21, 119, 96, 1)",
    fontsize: 12,
    bold: !1,
    italic: !1,
    horzLabelsAlign: "center",
    vertLabelsAlign: "top"
  },
  linetoolvertline: {
    clonable: !0,
    linecolor: "rgba( 128, 204, 219, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    showTime: !0
  },
  linetoolcirclelines: {
    clonable: !0,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    linecolor: "rgba( 128, 204, 219, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID
  },
  linetoolfibtimezone: {
    horzLabelsAlign: "right",
    vertLabelsAlign: "bottom",
    clonable: !0,
    baselinecolor: "rgba( 128, 128, 128, 1)",
    linecolor: "rgba( 0, 85, 219, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    showLabels: !0,
    font: "Verdana",
    fillBackground: !1,
    transparency: 80,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    level1: m.c(0, "rgba( 128, 128, 128, 1)", !0),
    level2: m.c(1, "rgba( 0, 85, 219, 1)", !0),
    level3: m.c(2, "rgba( 0, 85, 219, 1)", !0),
    level4: m.c(3, "rgba( 0, 85, 219, 1)", !0),
    level5: m.c(5, "rgba( 0, 85, 219, 1)", !0),
    level6: m.c(8, "rgba( 0, 85, 219, 1)", !0),
    level7: m.c(13, "rgba( 0, 85, 219, 1)", !0),
    level8: m.c(21, "rgba( 0, 85, 219, 1)", !0),
    level9: m.c(34, "rgba( 0, 85, 219, 1)", !0),
    level10: m.c(55, "rgba( 0, 85, 219, 1)", !0),
    level11: m.c(89, "rgba( 0, 85, 219, 1)", !0),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetooltext: {
    clonable: !0,
    color: "rgba( 102, 123, 139, 1)",
    text: $.t("Text"),
    font: "Verdana",
    fontsize: 20,
    fillBackground: !1,
    backgroundColor: "rgba( 91, 133, 191, 0.9)",
    backgroundTransparency: 70,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    bold: !1,
    italic: !1,
    locked: !1,
    fixedSize: !0,
    wordWrap: !1,
    wordWrapWidth: 400
  },
  linetooltextabsolute: {
    singleChartOnly: !0,
    clonable: !0,
    color: "rgba( 102, 123, 139, 1)",
    text: $.t("Text"),
    font: "Verdana",
    fontsize: 20,
    fillBackground: !1,
    backgroundColor: "rgba( 155, 190, 213, 0.3)",
    backgroundTransparency: 70,
    drawBorder: !1,
    borderColor: "rgba( 102, 123, 139, 1)",
    bold: !1,
    italic: !1,
    locked: !0,
    wordWrap: !1,
    wordWrapWidth: 400
  },
  linetoolballoon: {
    clonable: !0,
    color: "rgba( 102, 123, 139, 1)",
    backgroundColor: "rgba( 255, 254, 206, 0.7)",
    borderColor: "rgba( 140, 140, 140, 1)",
    fontWeight: "bold",
    fontsize: 12,
    font: "Arial",
    transparency: 30,
    text: $.t("Comment")
  },
  linetoolbrush: {
    clonable: !0,
    linecolor: "rgba( 53, 53, 53, 1)",
    linewidth: 2,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    smooth: 5,
    fillBackground: !1,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    transparency: 50,
    leftEnd: _.Normal,
    rightEnd: _.Normal
  },
  linetoolpolyline: {
    clonable: !0,
    linecolor: "rgba( 53, 53, 53, 1)",
    linewidth: 2,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    fillBackground: !0,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    transparency: 50,
    filled: !1
  },
  linetoolarrowmark: {
    clonable: !0,
    color: "rgba( 120, 120, 120, 1)",
    text: "",
    fontsize: 20,
    font: "Verdana"
  },
  linetoolarrowmarkleft: {
    clonable: !0,
    color: "rgba( 120, 120, 120, 1)",
    text: "",
    fontsize: 20,
    font: "Verdana"
  },
  linetoolarrowmarkup: {
    clonable: !0,
    color: "rgba( 120, 120, 120, 1)",
    text: "",
    fontsize: 20,
    font: "Verdana"
  },
  linetoolarrowmarkright: {
    clonable: !0,
    color: "rgba( 120, 120, 120, 1)",
    text: "",
    fontsize: 20,
    font: "Verdana"
  },
  linetoolarrowmarkdown: {
    clonable: !0,
    color: "rgba( 120, 120, 120, 1)",
    text: "",
    fontsize: 20,
    font: "Verdana"
  },
  linetoolflagmark: {
    clonable: !0,
    color: "rgba( 255, 0, 0, 1)"
  },
  linetoolnote: {
    clonable: !0,
    markerColor: "rgba( 46, 102, 255, 1)",
    textColor: "rgba( 0, 0, 0, 1)",
    backgroundColor: "rgba( 255, 255, 255, 1)",
    backgroundTransparency: 0,
    text: $.t("Text"),
    font: "Arial",
    fontSize: 12,
    bold: !1,
    italic: !1,
    locked: !1,
    fixedSize: !0
  },
  linetoolnoteabsolute: {
    singleChartOnly: !0,
    clonable: !0,
    markerColor: "rgba( 46, 102, 255, 1)",
    textColor: "rgba( 0, 0, 0, 1)",
    backgroundColor: "rgba( 255, 255, 255, 1)",
    backgroundTransparency: 0,
    text: $.t("Text"),
    font: "Arial",
    fontSize: 12,
    bold: !1,
    italic: !1,
    locked: !0,
    fixedSize: !0
  },
  linetoolthumbup: {
    clonable: !0,
    color: "rgba( 0, 128, 0, 1)"
  },
  linetoolthumbdown: {
    clonable: !0,
    color: "rgba( 255, 0, 0, 1)"
  },
  linetoolpricelabel: {
    clonable: !0,
    color: "rgba( 102, 123, 139, 1)",
    backgroundColor: "rgba( 255, 255, 255, 0.7)",
    borderColor: "rgba( 140, 140, 140, 1)",
    fontWeight: "bold",
    fontsize: 11,
    font: "Arial",
    transparency: 30
  },
  linetoolrectangle: {
    clonable: !0,
    color: "rgba( 21, 56, 153, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 21, 56, 153, 0.5)",
    linewidth: 1,
    snapTo45Degrees: !0,
    transparency: 50
  },
  linetoolrotatedrectangle: {
    clonable: !0,
    color: "rgba( 152, 0, 255, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 142, 124, 195, 0.5)",
    transparency: 50,
    linewidth: 1,
    snapTo45Degrees: !0
  },
  linetoolellipse: {
    clonable: !0,
    color: "rgba( 153, 153, 21, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 153, 153, 21, 0.5)",
    transparency: 50,
    linewidth: 1
  },
  linetoolarc: {
    clonable: !0,
    color: "rgba( 153, 153, 21, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 153, 153, 21, 0.5)",
    transparency: 50,
    linewidth: 1
  },
  linetoolprediction: {
    singleChartOnly: !0,
    linecolor: "rgba( 28, 115, 219, 1)",
    linewidth: 2,
    sourceBackColor: "rgba( 241, 241, 241, 1)",
    sourceTextColor: "rgba( 110, 110, 110, 1)",
    sourceStrokeColor: "rgba( 110, 110, 110, 1)",
    targetStrokeColor: "rgba( 47, 168, 255, 1)",
    targetBackColor: "rgba( 11, 111, 222, 1)",
    targetTextColor: "rgba( 255, 255, 255, 1)",
    successBackground: "rgba( 54, 160, 42, 0.9)",
    successTextColor: "rgba( 255, 255, 255, 1)",
    failureBackground: "rgba( 231, 69, 69, 0.5)",
    failureTextColor: "rgba( 255, 255, 255, 1)",
    intermediateBackColor: "rgba( 234, 210, 137, 1)",
    intermediateTextColor: "rgba( 109, 77, 34, 1)",
    transparency: 10,
    centersColor: "rgba( 32, 32, 32, 1)"
  },
  linetooltriangle: {
    clonable: !0,
    color: "rgba( 153, 21, 21, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 153, 21, 21, 0.5)",
    transparency: 50,
    linewidth: 1
  },
  linetoolcallout: {
    clonable: !0,
    color: "rgba( 255, 255, 255, 1)",
    backgroundColor: "rgba( 153, 21, 21, 0.5)",
    transparency: 50,
    linewidth: 2,
    fontsize: 12,
    font: "Verdana",
    text: $.t("Text"),
    bordercolor: "rgba( 153, 21, 21, 1)",
    bold: !1,
    italic: !1,
    wordWrap: !1,
    wordWrapWidth: 400
  },
  linetoolparallelchannel: {
    clonable: !0,
    linecolor: "rgba( 119, 52, 153, 1)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    extendLeft: !1,
    extendRight: !1,
    fillBackground: !0,
    backgroundColor: "rgba( 180, 167, 214, 0.5)",
    transparency: 50,
    showMidline: !1,
    midlinecolor: "rgba( 119, 52, 153, 1)",
    midlinewidth: 1,
    midlinestyle: CanvasEx.LINESTYLE_DASHED
  },
  linetoolelliottimpulse: {
    degree: 7,
    clonable: !0,
    showWave: !0,
    color: "rgba( 61, 133, 198, 1)",
    linewidth: 1
  },
  linetoolelliotttriangle: {
    degree: 7,
    clonable: !0,
    showWave: !0,
    color: "rgba( 255, 152, 0, 1)",
    linewidth: 1
  },
  linetoolelliotttriplecombo: {
    degree: 7,
    clonable: !0,
    showWave: !0,
    color: "rgba( 106, 168, 79, 1)",
    linewidth: 1
  },
  linetoolelliottcorrection: {
    degree: 7,
    clonable: !0,
    showWave: !0,
    color: "rgba( 61, 133, 198, 1)",
    linewidth: 1
  },
  linetoolelliottdoublecombo: {
    degree: 7,
    clonable: !0,
    showWave: !0,
    color: "rgba( 106, 168, 79, 1)",
    linewidth: 1
  },
  linetoolbarspattern: {
    singleChartOnly: !0,
    color: "rgba( 80, 145, 204, 1)",
    clonable: !0,
    mode: u.Bars,
    mirrored: !1,
    flipped: !1
  },
  linetoolghostfeed: {
    singleChartOnly: !0,
    clonable: !0,
    averageHL: 20,
    variance: 50,
    candleStyle: {
      upColor: "#6ba583",
      downColor: "#d75442",
      drawWick: !0,
      drawBorder: !0,
      borderColor: "#378658",
      borderUpColor: "#225437",
      borderDownColor: "#5b1a13",
      wickColor: "#737375"
    },
    transparency: 50
  },
  study: {
    inputs: {},
    styles: {},
    palettes: {},
    bands: {},
    area: {},
    graphics: {},
    showInDataWindow: !0,
    visible: !0,
    showStudyArguments: !0,
    precision: "default"
  },
  linetoolpitchfork: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    style: h.Original,
    median: {
      visible: !0,
      color: "rgba( 165, 0, 0, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level0: m.c(.25, "rgba( 160, 107, 0, 1)", !1),
    level1: m.c(.382, "rgba( 105, 158, 0, 1)", !1),
    level2: m.c(.5, "rgba( 0, 155, 0, 1)", !0),
    level3: m.c(.618, "rgba( 0, 153, 101, 1)", !1),
    level4: m.c(.75, "rgba( 0, 101, 153, 1)", !1),
    level5: m.c(1, "rgba( 0, 0, 153, 1)", !0),
    level6: m.c(1.5, "rgba( 102, 0, 153, 1)", !1),
    level7: m.c(1.75, "rgba( 153, 0, 102, 1)", !1),
    level8: m.c(2, "rgba( 165, 0, 0, 1)", !1),
    __collectibleLines: ["median", "level0", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8"]
  },
  linetoolpitchfan: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    median: {
      visible: !0,
      color: "rgba( 165, 0, 0, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level0: m.c(.25, "rgba( 160, 107, 0, 1)", !1),
    level1: m.c(.382, "rgba( 105, 158, 0, 1)", !1),
    level2: m.c(.5, "rgba( 0, 155, 0, 1)", !0),
    level3: m.c(.618, "rgba( 0, 153, 101, 1)", !1),
    level4: m.c(.75, "rgba( 0, 101, 153, 1)", !1),
    level5: m.c(1, "rgba( 0, 0, 153, 1)", !0),
    level6: m.c(1.5, "rgba( 102, 0, 153, 1)", !1),
    level7: m.c(1.75, "rgba( 153, 0, 102, 1)", !1),
    level8: m.c(2, "rgba( 165, 0, 0, 1)", !1),
    __collectibleLines: ["median", "level0", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8"]
  },
  linetoolgannfan: {
    clonable: !0,
    showLabels: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    level1: m.f(1, 8, "rgba( 160, 107, 0, 1)", !0),
    level2: m.f(1, 4, "rgba( 105, 158, 0, 1)", !0),
    level3: m.f(1, 3, "rgba( 0, 155, 0, 1)", !0),
    level4: m.f(1, 2, "rgba( 0, 153, 101, 1)", !0),
    level5: m.f(1, 1, "rgba( 128, 128, 128, 1)", !0),
    level6: m.f(2, 1, "rgba( 0, 101, 153, 1)", !0),
    level7: m.f(3, 1, "rgba( 0, 0, 153, 1)", !0),
    level8: m.f(4, 1, "rgba( 102, 0, 153, 1)", !0),
    level9: m.f(8, 1, "rgba( 165, 0, 0, 1)", !0),
    __collectibleLines: ["level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetoolganncomplex: {
    clonable: !0,
    fillBackground: !1,
    arcsBackground: {
      fillBackground: !0,
      transparency: 80
    },
    reverse: !1,
    scaleRatio: "",
    showLabels: !0,
    labelsStyle: {
      font: "Verdana",
      fontSize: 12,
      bold: !1,
      italic: !1
    },
    levels: [m.d("rgba( 128, 128, 128, 1)", !0, 1), m.d("rgba( 160, 107, 0, 1)", !0, 1), m.d("rgba( 105, 158, 0, 1)", !0, 1), m.d("rgba( 0, 155, 0, 1)", !0, 1), m.d("rgba( 0, 153, 101, 1)", !0, 1), m.d("rgba( 128, 128, 128, 1)", !0, 1)],
    fanlines: [m.e("rgba( 165, 0, 255, 1)", !1, 1, 8, 1), m.e("rgba( 165, 0, 0, 1)", !1, 1, 5, 1), m.e("rgba( 128, 128, 128, 1)", !1, 1, 4, 1), m.e("rgba( 160, 107, 0, 1)", !1, 1, 3, 1), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 1), m.e("rgba( 0, 155, 0, 1)", !0, 1, 1, 1), m.e("rgba( 0, 153, 101, 1)", !0, 1, 1, 2), m.e("rgba( 0, 153, 101, 1)", !1, 1, 1, 3), m.e("rgba( 0, 0, 153, 1)", !1, 1, 1, 4), m.e("rgba( 102, 0, 153, 1)", !1, 1, 1, 5), m.e("rgba( 165, 0, 255, 1)", !1, 1, 1, 8)],
    arcs: [m.e("rgba( 160, 107, 0, 1)", !0, 1, 1, 0), m.e("rgba( 160, 107, 0, 1)", !0, 1, 1, 1), m.e("rgba( 160, 107, 0, 1)", !0, 1, 1.5, 0), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 0), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 1), m.e("rgba( 0, 155, 0, 1)", !0, 1, 3, 0), m.e("rgba( 0, 155, 0, 1)", !0, 1, 3, 1), m.e("rgba( 0, 153, 101, 1)", !0, 1, 4, 0), m.e("rgba( 0, 153, 101, 1)", !0, 1, 4, 1), m.e("rgba( 0, 0, 153, 1)", !0, 1, 5, 0), m.e("rgba( 0, 0, 153, 1)", !0, 1, 5, 1)],
    __collectibleLines: ["trendline", "levels.0", "levels.1", "levels.2", "levels.3", "levels.4", "levels.5", "fanlines.0", "fanlines.1", "fanlines.2", "fanlines.3", "fanlines.4", "fanlines.5", "fanlines.6", "fanlines.7", "fanlines.8", "fanlines.9", "fanlines.10", "arcs.0", "arcs.1", "arcs.2", "arcs.3", "arcs.4", "arcs.5", "arcs.6", "arcs.7", "arcs.8", "arcs.9", "arcs.10"]
  },
  linetoolgannfixed: {
    clonable: !0,
    fillBackground: !1,
    arcsBackground: {
      fillBackground: !0,
      transparency: 80
    },
    reverse: !1,
    levels: [m.d("rgba( 128, 128, 128, 1)", !0, 1), m.d("rgba( 160, 107, 0, 1)", !0, 1), m.d("rgba( 105, 158, 0, 1)", !0, 1), m.d("rgba( 0, 155, 0, 1)", !0, 1), m.d("rgba( 0, 153, 101, 1)", !0, 1), m.d("rgba( 128, 128, 128, 1)", !0, 1)],
    fanlines: [m.e("rgba( 165, 0, 255, 1)", !1, 1, 8, 1), m.e("rgba( 165, 0, 0, 1)", !1, 1, 5, 1), m.e("rgba( 128, 128, 128, 1)", !1, 1, 4, 1), m.e("rgba( 160, 107, 0, 1)", !1, 1, 3, 1), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 1), m.e("rgba( 0, 155, 0, 1)", !0, 1, 1, 1), m.e("rgba( 0, 153, 101, 1)", !0, 1, 1, 2), m.e("rgba( 0, 153, 101, 1)", !1, 1, 1, 3), m.e("rgba( 0, 0, 153, 1)", !1, 1, 1, 4), m.e("rgba( 102, 0, 153, 1)", !1, 1, 1, 5), m.e("rgba( 165, 0, 255, 1)", !1, 1, 1, 8)],
    arcs: [m.e("rgba( 160, 107, 0, 1)", !0, 1, 1, 0), m.e("rgba( 160, 107, 0, 1)", !0, 1, 1, 1), m.e("rgba( 160, 107, 0, 1)", !0, 1, 1.5, 0), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 0), m.e("rgba( 105, 158, 0, 1)", !0, 1, 2, 1), m.e("rgba( 0, 155, 0, 1)", !0, 1, 3, 0), m.e("rgba( 0, 155, 0, 1)", !0, 1, 3, 1), m.e("rgba( 0, 153, 101, 1)", !0, 1, 4, 0), m.e("rgba( 0, 153, 101, 1)", !0, 1, 4, 1), m.e("rgba( 0, 0, 153, 1)", !0, 1, 5, 0), m.e("rgba( 0, 0, 153, 1)", !0, 1, 5, 1)],
    __collectibleLines: ["trendline", "levels.0", "levels.1", "levels.2", "levels.3", "levels.4", "levels.5", "fanlines.0", "fanlines.1", "fanlines.2", "fanlines.3", "fanlines.4", "fanlines.5", "fanlines.6", "fanlines.7", "fanlines.8", "fanlines.9", "fanlines.10", "arcs.0", "arcs.1", "arcs.2", "arcs.3", "arcs.4", "arcs.5", "arcs.6", "arcs.7", "arcs.8", "arcs.9", "arcs.10"]
  },
  linetoolgannsquare: {
    clonable: !0,
    color: "rgba( 21, 56, 153, 0.8)",
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    font: "Verdana",
    showTopLabels: !0,
    showBottomLabels: !0,
    showLeftLabels: !0,
    showRightLabels: !0,
    fillHorzBackground: !0,
    horzTransparency: 80,
    fillVertBackground: !0,
    vertTransparency: 80,
    reverse: !1,
    fans: m.a("rgba( 128, 128, 128, 1)", !1),
    hlevel1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    hlevel2: m.b(.25, "rgba( 160, 107, 0, 1)", !0),
    hlevel3: m.b(.382, "rgba( 105, 158, 0, 1)", !0),
    hlevel4: m.b(.5, "rgba( 0, 155, 0, 1)", !0),
    hlevel5: m.b(.618, "rgba( 0, 153, 101, 1)", !0),
    hlevel6: m.b(.75, "rgba( 0, 101, 153, 1)", !0),
    hlevel7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    vlevel1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    vlevel2: m.b(.25, "rgba( 160, 107, 0, 1)", !0),
    vlevel3: m.b(.382, "rgba( 105, 158, 0, 1)", !0),
    vlevel4: m.b(.5, "rgba( 0, 155, 0, 1)", !0),
    vlevel5: m.b(.618, "rgba( 0, 153, 101, 1)", !0),
    vlevel6: m.b(.75, "rgba( 0, 101, 153, 1)", !0),
    vlevel7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    __collectibleLines: ["fans", "hlevel1", "hlevel2", "hlevel3", "hlevel4", "hlevel5", "hlevel6", "hlevel7", "vlevel1", "vlevel2", "vlevel3", "vlevel4", "vlevel5", "vlevel6", "vlevel7"]
  },
  linetoolfibspeedresistancefan: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    grid: {
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID,
      visible: !0
    },
    linewidth: 1,
    linestyle: CanvasEx.LINESTYLE_SOLID,
    font: "Verdana",
    showTopLabels: !0,
    showBottomLabels: !0,
    showLeftLabels: !0,
    showRightLabels: !0,
    snapTo45Degrees: !0,
    hlevel1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    hlevel2: m.b(.25, "rgba( 160, 107, 0, 1)", !0),
    hlevel3: m.b(.382, "rgba( 105, 158, 0, 1)", !0),
    hlevel4: m.b(.5, "rgba( 0, 155, 0, 1)", !0),
    hlevel5: m.b(.618, "rgba( 0, 153, 101, 1)", !0),
    hlevel6: m.b(.75, "rgba( 0, 101, 153, 1)", !0),
    hlevel7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    vlevel1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    vlevel2: m.b(.25, "rgba( 160, 107, 0, 1)", !0),
    vlevel3: m.b(.382, "rgba( 105, 158, 0, 1)", !0),
    vlevel4: m.b(.5, "rgba( 0, 155, 0, 1)", !0),
    vlevel5: m.b(.618, "rgba( 0, 153, 101, 1)", !0),
    vlevel6: m.b(.75, "rgba( 0, 101, 153, 1)", !0),
    vlevel7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    __collectibleLines: ["trendline", "hlevel1", "hlevel2", "hlevel3", "hlevel4", "hlevel5", "hlevel6", "hlevel7", "vlevel1", "vlevel2", "vlevel3", "vlevel4", "vlevel5", "vlevel6", "vlevel7"]
  },
  linetoolfibretracement: {
    clonable: !0,
    showCoeffs: !0,
    showPrices: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    extendLines: !1,
    horzLabelsAlign: "left",
    vertLabelsAlign: "middle",
    reverse: !1,
    coeffsAsPercents: !1,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    levelsStyle: {
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    level2: m.b(.236, "rgba( 204, 40, 40, 1)", !0),
    level3: m.b(.382, "rgba( 149, 204, 40, 1)", !0),
    level4: m.b(.5, "rgba( 40, 204, 40, 1)", !0),
    level5: m.b(.618, "rgba( 40, 204, 149, 1)", !0),
    level6: m.b(.786, "rgba( 40, 149, 204, 1)", !0),
    level7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    level8: m.b(1.618, "rgba( 40, 40, 204, 1)", !0),
    level9: m.b(2.618, "rgba( 204, 40, 40, 1)", !0),
    level10: m.b(3.618, "rgba( 149, 40, 204, 1)", !0),
    level11: m.b(4.236, "rgba( 204, 40, 149, 1)", !0),
    level12: m.b(1.272, "rgba( 149, 204, 40, 1)", !1),
    level13: m.b(1.414, "rgba( 204, 40, 40, 1)", !1),
    level16: m.b(2, "rgba( 40, 204, 149, 1)", !1),
    level14: m.b(2.272, "rgba( 149, 204, 40, 1)", !1),
    level15: m.b(2.414, "rgba( 40, 204, 40, 1)", !1),
    level17: m.b(3, "rgba( 40, 149, 204, 1)", !1),
    level18: m.b(3.272, "rgba( 128, 128, 128, 1)", !1),
    level19: m.b(3.414, "rgba( 40, 40, 204, 1)", !1),
    level20: m.b(4, "rgba( 204, 40, 40, 1)", !1),
    level21: m.b(4.272, "rgba( 149, 40, 204, 1)", !1),
    level22: m.b(4.414, "rgba( 204, 40, 149, 1)", !1),
    level23: m.b(4.618, "rgba( 149, 204, 40, 1)", !1),
    level24: m.b(4.764, "rgba( 40, 204, 149, 1)", !1),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11", "level12", "level13", "level14", "level15", "level16", "level17", "level18", "level19", "level20", "level21", "level22", "level23", "level24"]
  },
  linetoolfibchannel: {
    clonable: !0,
    showCoeffs: !0,
    showPrices: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    extendLeft: !1,
    extendRight: !1,
    horzLabelsAlign: "left",
    vertLabelsAlign: "middle",
    coeffsAsPercents: !1,
    levelsStyle: {
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    level2: m.b(.236, "rgba( 204, 40, 40, 1)", !0),
    level3: m.b(.382, "rgba( 149, 204, 40, 1)", !0),
    level4: m.b(.5, "rgba( 40, 204, 40, 1)", !0),
    level5: m.b(.618, "rgba( 40, 204, 149, 1)", !0),
    level6: m.b(.786, "rgba( 40, 149, 204, 1)", !0),
    level7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    level8: m.b(1.618, "rgba( 40, 40, 204, 1)", !0),
    level9: m.b(2.618, "rgba( 204, 40, 40, 1)", !0),
    level10: m.b(3.618, "rgba( 149, 40, 204, 1)", !0),
    level11: m.b(4.236, "rgba( 204, 40, 149, 1)", !0),
    level12: m.b(1.272, "rgba( 149, 204, 40, 1)", !1),
    level13: m.b(1.414, "rgba( 204, 40, 40, 1)", !1),
    level16: m.b(2, "rgba( 40, 204, 149, 1)", !1),
    level14: m.b(2.272, "rgba( 149, 204, 40, 1)", !1),
    level15: m.b(2.414, "rgba( 40, 204, 40, 1)", !1),
    level17: m.b(3, "rgba( 40, 149, 204, 1)", !1),
    level18: m.b(3.272, "rgba( 128, 128, 128, 1)", !1),
    level19: m.b(3.414, "rgba( 40, 40, 204, 1)", !1),
    level20: m.b(4, "rgba( 204, 40, 40, 1)", !1),
    level21: m.b(4.272, "rgba( 149, 40, 204, 1)", !1),
    level22: m.b(4.414, "rgba( 204, 40, 149, 1)", !1),
    level23: m.b(4.618, "rgba( 149, 204, 40, 1)", !1),
    level24: m.b(4.764, "rgba( 40, 204, 149, 1)", !1),
    __collectibleLines: ["level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11", "level12", "level13", "level14", "level15", "level16", "level17", "level18", "level19", "level20", "level21", "level22", "level23", "level24"]
  },
  linetoolprojection: {
    clonable: !0,
    showCoeffs: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    color1: "rgba( 0, 128, 0, 0.2)",
    color2: "rgba( 255, 0, 0, 0.2)",
    linewidth: 1,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level1: m.c(1, "rgba( 128, 128, 128, 1)", !0)
  },
  linetool5pointspattern: {
    clonable: !0,
    color: "rgba( 204, 40, 149, 1)",
    textcolor: "rgba( 255, 255, 255, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 204, 40, 149, 0.5)",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    transparency: 50,
    linewidth: 1
  },
  linetoolcypherpattern: {
    clonable: !0,
    color: "#CC2895",
    textcolor: "#FFFFFF",
    fillBackground: !0,
    backgroundColor: "#CC2895",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    transparency: 50,
    linewidth: 1
  },
  linetooltrianglepattern: {
    clonable: !0,
    color: "rgba( 149, 40, 255, 1)",
    textcolor: "rgba( 255, 255, 255, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 149, 40, 204, 0.5)",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    transparency: 50,
    linewidth: 1
  },
  linetoolabcd: {
    clonable: !0,
    color: "rgba( 0, 155, 0, 1)",
    textcolor: "rgba( 255, 255, 255, 1)",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    linewidth: 2
  },
  linetoolthreedrivers: {
    clonable: !0,
    color: "rgba( 149, 40, 255, 1)",
    textcolor: "rgba( 255, 255, 255, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 149, 40, 204, 0.5)",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    transparency: 50,
    linewidth: 2
  },
  linetoolheadandshoulders: {
    clonable: !0,
    color: "rgba( 69, 104, 47, 1)",
    textcolor: "rgba( 255, 255, 255, 1)",
    fillBackground: !0,
    backgroundColor: "rgba( 69, 168, 47, 0.5)",
    font: "Verdana",
    fontsize: 12,
    bold: !1,
    italic: !1,
    transparency: 50,
    linewidth: 2
  },
  linetoolfibwedge: {
    singleChartOnly: !0,
    clonable: !0,
    showCoeffs: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level1: m.c(.236, "rgba( 204, 40, 40, 1)", !0),
    level2: m.c(.382, "rgba( 149, 204, 40, 1)", !0),
    level3: m.c(.5, "rgba( 40, 204, 40, 1)", !0),
    level4: m.c(.618, "rgba( 40, 204, 149, 1)", !0),
    level5: m.c(.786, "rgba( 40, 149, 204, 1)", !0),
    level6: m.c(1, "rgba( 128, 128, 128, 1)", !0),
    level7: m.c(1.618, "rgba( 40, 40, 204, 1)", !1),
    level8: m.c(2.618, "rgba( 204, 40, 40, 1)", !1),
    level9: m.c(3.618, "rgba( 149, 40, 204, 1)", !1),
    level10: m.c(4.236, "rgba( 204, 40, 149, 1)", !1),
    level11: m.c(4.618, "rgba( 204, 40, 149, 1)", !1),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetoolfibcircles: {
    clonable: !0,
    showCoeffs: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    snapTo45Degrees: !0,
    coeffsAsPercents: !1,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    level1: m.c(.236, "rgba( 204, 40, 40, 1)", !0),
    level2: m.c(.382, "rgba( 149, 204, 40, 1)", !0),
    level3: m.c(.5, "rgba( 40, 204, 40, 1)", !0),
    level4: m.c(.618, "rgba( 40, 204, 149, 1)", !0),
    level5: m.c(.786, "rgba( 40, 149, 204, 1)", !0),
    level6: m.c(1, "rgba( 128, 128, 128, 1)", !0),
    level7: m.c(1.618, "rgba( 40, 40, 204, 1)", !0),
    level8: m.c(2.618, "rgba( 204, 40, 40, 1)", !0),
    level9: m.c(3.618, "rgba( 149, 40, 204, 1)", !0),
    level10: m.c(4.236, "rgba( 204, 40, 149, 1)", !0),
    level11: m.c(4.618, "rgba( 204, 40, 149, 1)", !0),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetoolfibspeedresistancearcs: {
    clonable: !0,
    showCoeffs: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    fullCircles: !1,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    level1: m.c(.236, "rgba( 204, 40, 40, 1)", !0),
    level2: m.c(.382, "rgba( 149, 204, 40, 1)", !0),
    level3: m.c(.5, "rgba( 40, 204, 40, 1)", !0),
    level4: m.c(.618, "rgba( 40, 204, 149, 1)", !0),
    level5: m.c(.786, "rgba( 40, 149, 204, 1)", !0),
    level6: m.c(1, "rgba( 128, 128, 128, 1)", !0),
    level7: m.c(1.618, "rgba( 40, 40, 204, 1)", !0),
    level8: m.c(2.618, "rgba( 204, 40, 40, 1)", !0),
    level9: m.c(3.618, "rgba( 149, 40, 204, 1)", !0),
    level10: m.c(4.236, "rgba( 204, 40, 149, 1)", !0),
    level11: m.c(4.618, "rgba( 204, 40, 149, 1)", !0),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetooltrendbasedfibextension: {
    clonable: !0,
    showCoeffs: !0,
    showPrices: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    extendLines: !1,
    horzLabelsAlign: "left",
    vertLabelsAlign: "middle",
    reverse: !1,
    coeffsAsPercents: !1,
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    levelsStyle: {
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level1: m.b(0, "rgba( 128, 128, 128, 1)", !0),
    level2: m.b(.236, "rgba( 204, 40, 40, 1)", !0),
    level3: m.b(.382, "rgba( 149, 204, 40, 1)", !0),
    level4: m.b(.5, "rgba( 40, 204, 40, 1)", !0),
    level5: m.b(.618, "rgba( 40, 204, 149, 1)", !0),
    level6: m.b(.786, "rgba( 40, 149, 204, 1)", !0),
    level7: m.b(1, "rgba( 128, 128, 128, 1)", !0),
    level8: m.b(1.618, "rgba( 40, 40, 204, 1)", !0),
    level9: m.b(2.618, "rgba( 204, 40, 40, 1)", !0),
    level10: m.b(3.618, "rgba( 149, 40, 204, 1)", !0),
    level11: m.b(4.236, "rgba( 204, 40, 149, 1)", !0),
    level12: m.b(1.272, "rgba( 149, 204, 40, 1)", !1),
    level13: m.b(1.414, "rgba( 204, 40, 40, 1)", !1),
    level16: m.b(2, "rgba( 40, 204, 149, 1)", !1),
    level14: m.b(2.272, "rgba( 149, 204, 40, 1)", !1),
    level15: m.b(2.414, "rgba( 40, 204, 40, 1)", !1),
    level17: m.b(3, "rgba( 40, 149, 204, 1)", !1),
    level18: m.b(3.272, "rgba( 128, 128, 128, 1)", !1),
    level19: m.b(3.414, "rgba( 40, 40, 204, 1)", !1),
    level20: m.b(4, "rgba( 204, 40, 40, 1)", !1),
    level21: m.b(4.272, "rgba( 149, 40, 204, 1)", !1),
    level22: m.b(4.414, "rgba( 204, 40, 149, 1)", !1),
    level23: m.b(4.618, "rgba( 149, 204, 40, 1)", !1),
    level24: m.b(4.764, "rgba( 40, 204, 149, 1)", !1),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11", "level12", "level13", "level14", "level15", "level16", "level17", "level18", "level19", "level20", "level21", "level22", "level23", "level24"]
  },
  linetooltrendbasedfibtime: {
    clonable: !0,
    showCoeffs: !0,
    font: "Verdana",
    fillBackground: !0,
    transparency: 80,
    horzLabelsAlign: "right",
    vertLabelsAlign: "bottom",
    trendline: {
      visible: !0,
      color: "rgba( 128, 128, 128, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_DASHED
    },
    level1: m.c(0, "rgba( 128, 128, 128, 1)", !0),
    level2: m.c(.382, "rgba( 204, 40, 40, 1)", !0),
    level3: m.c(.5, "rgba( 149, 204, 40, 1)", !1),
    level4: m.c(.618, "rgba( 40, 204, 40, 1)", !0),
    level5: m.c(1, "rgba( 40, 204, 149, 1)", !0),
    level6: m.c(1.382, "rgba( 40, 149, 204, 1)", !0),
    level7: m.c(1.618, "rgba( 128, 128, 128, 1)", !0),
    level8: m.c(2, "rgba( 40, 40, 204, 1)", !0),
    level9: m.c(2.382, "rgba( 204, 40, 40, 1)", !0),
    level10: m.c(2.618, "rgba( 149, 40, 204, 1)", !0),
    level11: m.c(3, "rgba( 204, 40, 149, 1)", !0),
    __collectibleLines: ["trendline", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8", "level9", "level10", "level11"]
  },
  linetoolschiffpitchfork: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    style: h.Schiff,
    median: {
      visible: !0,
      color: "rgba( 165, 0, 0, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level0: m.c(.25, "rgba( 160, 107, 0, 1)", !1),
    level1: m.c(.382, "rgba( 105, 158, 0, 1)", !1),
    level2: m.c(.5, "rgba( 0, 155, 0, 1)", !0),
    level3: m.c(.618, "rgba( 0, 153, 101, 1)", !1),
    level4: m.c(.75, "rgba( 0, 101, 153, 1)", !1),
    level5: m.c(1, "rgba( 0, 0, 153, 1)", !0),
    level6: m.c(1.5, "rgba( 102, 0, 153, 1)", !1),
    level7: m.c(1.75, "rgba( 153, 0, 102, 1)", !1),
    level8: m.c(2, "rgba( 165, 0, 0, 1)", !1),
    __collectibleLines: ["median", "level0", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8"]
  },
  linetoolschiffpitchfork2: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    style: h.Schiff2,
    median: {
      visible: !0,
      color: "rgba( 165, 0, 0, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level0: m.c(.25, "rgba( 160, 107, 0, 1)", !1),
    level1: m.c(.382, "rgba( 105, 158, 0, 1)", !1),
    level2: m.c(.5, "rgba( 0, 155, 0, 1)", !0),
    level3: m.c(.618, "rgba( 0, 153, 101, 1)", !1),
    level4: m.c(.75, "rgba( 0, 101, 153, 1)", !1),
    level5: m.c(1, "rgba( 0, 0, 153, 1)", !0),
    level6: m.c(1.5, "rgba( 102, 0, 153, 1)", !1),
    level7: m.c(1.75, "rgba( 153, 0, 102, 1)", !1),
    level8: m.c(2, "rgba( 165, 0, 0, 1)", !1),
    __collectibleLines: ["median", "level0", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8"]
  },
  linetoolinsidepitchfork: {
    clonable: !0,
    fillBackground: !0,
    transparency: 80,
    style: h.Inside,
    median: {
      visible: !0,
      color: "rgba( 165, 0, 0, 1)",
      linewidth: 1,
      linestyle: CanvasEx.LINESTYLE_SOLID
    },
    level0: m.c(.25, "rgba( 160, 107, 0, 1)", !1),
    level1: m.c(.382, "rgba( 105, 158, 0, 1)", !1),
    level2: m.c(.5, "rgba( 0, 155, 0, 1)", !0),
    level3: m.c(.618, "rgba( 0, 153, 101, 1)", !1),
    level4: m.c(.75, "rgba( 0, 101, 153, 1)", !1),
    level5: m.c(1, "rgba( 0, 0, 153, 1)", !0),
    level6: m.c(1.5, "rgba( 102, 0, 153, 1)", !1),
    level7: m.c(1.75, "rgba( 153, 0, 102, 1)", !1),
    level8: m.c(2, "rgba( 165, 0, 0, 1)", !1),
    __collectibleLines: ["median", "level0", "level1", "level2", "level3", "level4", "level5", "level6", "level7", "level8"]
  },
  linetool: {
    frozen: !1,
    visible: !0
  },
  linetoolvisibilities: {
    intervalsVisibilities: {
      seconds: !0,
      secondsFrom: 1,
      secondsTo: 59,
      minutes: !0,
      minutesFrom: 1,
      minutesTo: 59,
      hours: !0,
      hoursFrom: 1,
      hoursTo: 24,
      days: !0,
      daysFrom: 1,
      daysTo: 366,
      weeks: !0,
      months: !0
    }
  }
}
```
