<template>
<!-- 组件根元素 -->
<div id="stopwatch">
  <!-- 秒表表盘 -->
  <div id="stopwatch-dashes" ref="dashBox" :style="{'cursor': isGrabbing ? 'grabbing' : 'grab'}">
    <!-- 数字表盘 -->
    <div id="digital-dash" :style="digitalDashStyle">
      <span :style="{'fontSize': `${globalTimeFontSize}rem`}">
        {{ globalTimeFormated }}
      </span>
    </div>
    <!-- 模拟表盘 -->
    <div id="analog-dash" :style="analogDashStyle">
      <!-- 秒钟表盘（大）刻度数字 -->
      <ul class="large-dash-digits">
        <li 
            v-for="(digit, idx) in LARGE_DASH_DIGITS"
            :key="idx">
          {{ digit }}
        </li>
      </ul>
      <!-- 分钟表盘（小）刻度数字 -->
      <ul class="small-dash-digits">
        <li
            v-for="(digit, idx) in SMALL_DASH_DIGITS"
            :key="idx">
          {{ digit }}
      </li>
      </ul>
      <!-- 表盘刻度及指针 -->
      <svg
          :width="SIZE"
          :height="SIZE">
        <!-- 秒钟表盘（大） -->
        <circle
            class="large-dash-scales"
            v-for="(scaleCircle, name) in LARGE_DASH"
            :key="name"
            :cx="DASHES_OFFSET"
            :cy="DASHES_OFFSET"
            :r="radius(scaleCircle.diameter, scaleCircle.strokeWidth)"
            :fill="scaleCircle.fill"
            :stroke="scaleCircle.color"
            :stroke-width="scaleCircle.strokeWidth"
            :stroke-dasharray="dashArray(radius(scaleCircle.diameter, scaleCircle.strokeWidth), scaleCircle.unitNum, scaleCircle.spaceNum)"
            :style="{transformOrigin: '50% 50%', transform: `rotate(${-360 / 480 / 2}deg)`}">
        </circle>
        <!-- 分钟表盘（小） -->
        <circle
            class="small-dash-scales"
            v-for="(scaleCircle, name) in SMALL_DASH"
            :key="name"
            :cx="DASHES_OFFSET"
            :cy="DASHES_OFFSET - SMALL_DASH_Y"
            :r="radius(scaleCircle.diameter, scaleCircle.strokeWidth)"
            :fill="scaleCircle.fill"
            :stroke="scaleCircle.color"
            :stroke-width="scaleCircle.strokeWidth"
            :stroke-dasharray="dashArray(radius(scaleCircle.diameter, scaleCircle.strokeWidth), scaleCircle.unitNum, scaleCircle.spaceNum)"
            :style="{transformOrigin: `50% ${SIZE / 2 - SMALL_DASH_Y}px`, transform: `rotate(${90 - 360 / 180 / 2 }deg)`}">
        </circle>
        <!-- 分钟表盘（小）表针 -->
        <line
            class="small-dash-hand"
            :x1="SIZE / 2"
            :y1="SIZE / 2 - SMALL_DASH_D / 2 - SMALL_DASH_Y"
            :x2="SIZE / 2"
            :y2="SIZE / 2 - SMALL_DASH_Y"
            stroke="#ffa000"
            :stroke-width="HAND_WIDTH"
            :style="{transformOrigin: `50% ${SIZE / 2 - SMALL_DASH_Y}px`, transform: `rotate(${smallHandDegree}deg)`}">
        </line>
        <!-- 分钟表盘（小）转轴 -->
        <circle
            class="small-dash-axis"
            :cx="DASHES_OFFSET"
            :cy="DASHES_OFFSET - SMALL_DASH_Y"
            :r="radius(6, HAND_WIDTH)"
            fill="#ffa000"
            stroke="#ffa000"
            :stroke-width="HAND_WIDTH">
        </circle>
        <!-- 秒钟表盘（大）本计次表针 -->
        <line
            class="large-dash-hand-local"
            :x1="SIZE / 2"
            :y1="0"
            :x2="SIZE / 2"
            :y2="SIZE / 2 + 30"
            stroke="#00a0ff"
            :stroke-width="HAND_WIDTH"
            :style="{transformOrigin: `50% 50%`, transform: `rotate(${largeHandLocalDegree}deg)`}">
        </line>
        <!-- 秒钟表盘（大）全局表针 -->
        <line
            class="large-dash-hand-global"
            :x1="SIZE / 2"
            :y1="0"
            :x2="SIZE / 2"
            :y2="SIZE / 2 + 30"
            stroke="#ffa000"
            :stroke-width="HAND_WIDTH"
            :style="{transformOrigin: `50% 50%`, transform: `rotate(${largeHandGlobalDegree}deg)`}">
        </line>
        <!-- 秒钟表盘（大）转轴 -->
        <circle
            class="large-dash-axis"
            :cx="DASHES_OFFSET"
            :cy="DASHES_OFFSET"
            :r="radius(7, HAND_WIDTH)"
            fill="#000"
            stroke="#ffa000"
            :stroke-width="HAND_WIDTH">
        </circle>
      </svg>
      <!-- 全局时间数字表盘 -->
      <div class="small-digital-dash">
        {{ globalTimeFormated }}
      </div>
    </div>
  </div>

  <!-- 秒表按钮 -->
  <div id="stopwatch-buttons">
    <!-- 取消/计次按钮 -->
    <div class="round-buttons">
      <button class="left-button" @click="leftBtnClick" :class="state" :disabled="leftBtnDisabled">
        <span>{{ leftBtnContent }}</span>
      </button>
    </div>
    <!-- 表盘切换控制按钮 -->
    <div class="switch-button" ref="switcher">
      <div :class="{highlight: isDigitalDash}"></div>
      <div :class="{highlight: isAnalogDash}"></div>
    </div>
    <!-- 开始/停止按钮 -->
    <div class="round-buttons">
      <button class="right-button" @click="rightBtnClick" :class="state">
        <span>{{ rightBtnContent }}</span>
      </button>
    </div>
  </div>

  <!-- 秒表计次记录 -->
  <div id="stopwatch-laps">
    <ul>
      <li
          v-for="(time, idx) in laps"
          :key="idx"
          :class="{max: isMax(idx), min: isMin(idx)}">
        <span>计次 {{ laps.length - idx }}</span>
        <span>{{ time ? time.formatedTime : localTimeFormated }}</span>
      </li>
    </ul>
  </div>
</div>
</template>



<script setup>
import { ref, computed, onMounted } from 'vue';
import { range } from 'lodash';

// +----------------------------------------+
// |          绘制模拟表盘（非响应式）          |
// +----------------------------------------+
/**
 * radius 函数描述
 * @param {number} diameter 圆环外边直径
 * @param {number} strokeWidth 圆环宽度
 * @return {number} 圆环中环线半径
 */
const radius = (diameter, strokeWidth) => (diameter - strokeWidth) / 2;

/**
 * dashArray 函数描述
 * @param {number} radius 圆环中环线半径
 * @param {number} unitNum 圆周上[刻度线等宽单位]排布数量
 * @param {number} spaceNum 刻度线间隙宽度的[刻度线等宽单位]
 * @return {Array} stroke-dasharray 属性可接收的数组
 */
const dashArray = (radius, unitNum, spaceNum=1) => {
  const circumference = radius * 2 * Math.PI;
  const unitLength = circumference / unitNum;
  return spaceNum === 1 ? [unitLength] : [unitLength, unitLength * spaceNum];
};

const LARGE_DASH_DIGITS = range(5, 61, 5);              // 秒钟表盘（大）显示数码
const SMALL_DASH_DIGITS = range(5, 31, 5);              // 分钟表盘（小）显示数码
const SIZE = 300;                                       // 秒钟表盘（大）外边直径
const DASHES_OFFSET = SIZE / 2;                         // 表盘中点对齐坐标
const SMALL_DASH_D = 80;                                // 分钟表盘（小）外边直径
const SMALL_DASH_Y = 55;                                // 分钟表盘（小）中点 y 坐标再偏移补偿量
const HAND_WIDTH = dashArray(radius(SIZE, 8), 480)[0];  // 指针宽度

const LARGE_DASH = {
  lowScale: {
    diameter: SIZE,
    strokeWidth: 8,
    unitNum: 480,
    spaceNum: 1,
    color: "#555",
    fill: "transparent",
  },
  highScale: {
    diameter: SIZE,
    strokeWidth: 14,
    unitNum: 480,
    spaceNum: 7,
    color: "#555",
    fill: "transparent",
  },
  highLight: {
    diameter: SIZE,
    strokeWidth: 14,
    unitNum: 480,
    spaceNum: 39,
    color: "#fff",
    fill: "transparent",
  },
};

const SMALL_DASH = {
  lowScale: {
    diameter: SMALL_DASH_D,
    strokeWidth: 5,
    unitNum: 180,
    spaceNum: 2,
    color: "#555",
    fill: "transparent",
  },
  highScale: {
    diameter: SMALL_DASH_D,
    strokeWidth: 8,
    unitNum: 180,
    spaceNum: 5,
    color: "#555",
    fill: "transparent",
  },
  highLight: {
    diameter: SMALL_DASH_D,
    strokeWidth: 8,
    unitNum: 180,
    spaceNum: 29,
    color: "#fff",
    fill: "transparent",
  },
};

// +----------------------------------------+
// |                秒表计时器                |
// +----------------------------------------+
const timer = ref(null);  // 计时器
const globalTime = ref(0);  // 全局总时间（毫秒）
const localTime = ref(0);  // 本计次总时间（毫秒）

/**
 * getMilliSeconds 函数描述
 * @param {number} time 毫秒数
 * @return 毫秒位显示值
 */
const getMilliSeconds = (time) => Math.floor((time % 1000) / 10).toString().padStart(2, 0);

/**
 * getSeconds 函数描述
 * @param {number} time 毫秒数
 * @return 秒钟位显示值
 */
const getSeconds = (time) => Math.floor((time % 60000) / 1000).toString().padStart(2, 0);

/**
 * getMinutes 函数描述
 * @param {number} time 毫秒数
 * @return 分钟位显示值
 */
const getMinutes = (time) => Math.floor((time % 3600000) / 60000).toString().padStart(2, 0);

/**
 * getHours 函数描述
 * @param {number} time 毫秒数
 * @return 小时位显示值
 */
const getHours = (time) => Math.floor((time % 86400000) / 3600000).toString();

const globalMilliSeconds = computed(() => getMilliSeconds(globalTime.value));            // 全局时间毫秒位
const globalSeconds = computed(() => getSeconds(globalTime.value));                      // 全局时间秒钟位
const globalMinutes = computed(() => getMinutes(globalTime.value));                      // 全局时间分钟位
const globalHours = computed(() => getHours(globalTime.value));                          // 全局时间小时位
const hasGlobalHours = computed(() => globalHours.value !== '0');                        // 全局时间是否含有小时位
const globalTimeFontSize = computed(() => globalHours.value === '0' ? 5 : (globalHours.value.length === 1 ? 4.4 : 3.8));
const globalTimeFormated = computed(() => (hasGlobalHours.value ? `${globalHours.value}:` : '') + `${globalMinutes.value}:${globalSeconds.value}.${globalMilliSeconds.value}`);
const localMilliSeconds = computed(() => getMilliSeconds(localTime.value));              // 本计次时间毫秒位
const localSeconds = computed(() => getSeconds(localTime.value));                        // 本计次时间秒钟位
const localMinutes = computed(() => getMinutes(localTime.value));                        // 本计次时间分钟位
const localHours = computed(() => getHours(localTime.value));                            // 本计次时间小时位
const hasLocalHours = computed(() => localHours.value !== '0');                          // 本计次时间是否含有小时位
const localTimeFormated = computed(() => (hasLocalHours.value ? `${localHours.value}:` : '') + `${localMinutes.value}:${localSeconds.value}.${localMilliSeconds.value}`);
const largeHandGlobalDegree = computed(() => (globalTime.value % 60000) * 360 / 60000);  // 秒钟表盘（大）全局时间指针角度
const largeHandLocalDegree = computed(() => (localTime.value % 60000) * 360 / 60000);    // 秒钟表盘（大）本计次时间指针角度
const smallHandDegree = computed(() => (globalTime.value % 1800000) * 360 / 1800000);    // 分钟表盘（小）全局时间指针角度

// +----------------------------------------+
// |            （计时操作）按钮逻辑            |
// +----------------------------------------+
// 使用 timer 和 globalTime 作为秒表状态判断依据
const state = computed(() => timer.value !== null ? 'started' : (globalTime.value > 0 ? 'stopped' : 'reset'));
const leftBtnContent = computed(() => state.value === 'stopped' ? '复位' : '计次');
const rightBtnContent = computed(() => state.value === 'started' ? '停止' : '启动');
const leftBtnDisabled = computed(() => state.value === 'reset');
const leftBtnClick = () => {
  if (state.value === 'started') {
    // 秒表已启动，点击以计次
    laps.value[0] = {
      milliSeconds: localTime.value,
      formatedTime: localTimeFormated.value,
    };
    localTime.value = 0;  // 置0本计次时间
    if (laps.value[maxLapIdx.value].milliSeconds > laps.value[0].milliSeconds) {
      maxLapIdx.value++;
    } else {
      maxLapIdx.value = 1;
    }
    if (laps.value[minLapIdx.value].milliSeconds < laps.value[0].milliSeconds) {
      minLapIdx.value++;
    } else {
      minLapIdx.value = 1;
    }
    laps.value.unshift(null)  // 向 laps 首位插入空时间占位符（html 中解析为响应式本计次时间）
  } else if (state.value === 'stopped') {
    // 秒表已停止，点击以复位
    clearInterval(timer.value);
    globalTime.value = 0;
    localTime.value = 0;
    timer.value = null;  // 先停止计时器，再置0 globalTime，最后清空计时器，防止状态跳变
    laps.value = [];  // 计次列表清空
    maxLapIdx.value = 0;
    minLapIdx.value = 0;
  }
};
const rightBtnClick = () => {
  const deltaTime = 10;
  if (state.value === 'reset') {
    // 秒表已复位，点击以启动秒表
    clearInterval(timer.value);  // （保险）清除计时器
    globalTime.value = 0;  // （保险）复位全局时间
    localTime.value = 0;  // （保险）复位本计次时间
    laps.value.unshift(null);  // 向 laps 首位插入空时间占位符（html 中解析为响应式本计次时间）
    timer.value = setInterval(() => {
      globalTime.value += deltaTime;
      localTime.value += deltaTime;
    }, deltaTime);
  } else if (state.value === 'started') {
    // 秒表已启动，点击以停止秒表
    clearInterval(timer.value);
    timer.value = null;
  } else if (state.value === 'stopped'){
    // 秒表已停止，点击以重新启动秒表
    clearInterval(timer.value);  // （保险）清除计时器
    timer.value = setInterval(() => {
      globalTime.value += deltaTime;
      localTime.value += deltaTime;
    }, deltaTime);
  }
};

// +----------------------------------------+
// |               秒表计次记录               |
// +----------------------------------------+
const laps = ref([]);
const maxLapIdx = ref(0);
const minLapIdx = ref(0);
const isMax = (idx) => laps.value.length > 2 && idx === maxLapIdx.value;
const isMin = (idx) => laps.value.length > 2 && idx === minLapIdx.value;

// +----------------------------------------+
// |               表盘拖动切换               |
// +----------------------------------------+
// 获取表盘 DOM 元素
const dashBox = ref(null);
// 定义表盘及其位移属性（基准，拖动边界，切换点）
const dashWidth = 330;
const dashes = {
  digital: {
    baseX: 0,
    deltaX: {
      max: 0,
      min: -dashWidth,
      switchPnt: -(dashWidth / 2),
    },
  },
  analog: {
    baseX: -dashWidth,
    deltaX: {
      max: dashWidth,
      min: 0,
      switchPnt: dashWidth / 2,
    },
  },
};
// 计算各表盘位置及动画
const currentDash = ref('digital');                                               // 当前表盘
const mouseDownX = ref(0);                                                        // 鼠标左键按下的 x 坐标
const mouseMoveX = ref(0);                                                        // 鼠标左键按下后移动的 x 坐标
const deltaX = computed(() => {                                                   // 表盘 x 坐标移动量
  const delta = mouseMoveX.value - mouseDownX.value;
  const maxDelta = dashes[currentDash.value].deltaX.max;
  const minDelta = dashes[currentDash.value].deltaX.min;
  // 超出 min、max 的部分做减小位移量处理
  if (delta > maxDelta) {
    return maxDelta + Math.pow(delta - maxDelta, 0.8);
  } else if (delta < minDelta) {
    return minDelta - Math.pow(minDelta - delta, 0.8);
  } else {
    return delta;
  }
});
const digitalX = computed(() => dashes[currentDash.value].baseX + deltaX.value);  // 数字表盘 x 轴坐标
const analogX = computed(() => digitalX.value + dashWidth);                       // 模拟表盘 x 轴坐标
const isGrabbing = computed(() => mouseDownX.value !== 0);                        // 是否正在拖动
const digitalDashStyle = computed(() => {                                         // 数字表盘样式
  const style = {transform: `translateX(${digitalX.value}px)`};
  if (!isGrabbing.value) style['transition'] = 'transform .2s ease-out'
  return style;
});
const analogDashStyle = computed(() => {                                          // 模拟表盘样式
  const style = {transform: `translateX(${analogX.value}px)`};
  if (!isGrabbing.value) style['transition'] = 'transform .2s ease-out'
  return style;
});
// 监听鼠标行为反馈表盘位置，判断切换条件
onMounted(() => {
  // 鼠标移动事件监听处理函数，单拎出来方便取消监听
  const grabbingHandler = (event) => {mouseMoveX.value = event.x};
  // 监听鼠标左键按下事件
  dashBox.value.addEventListener('mousedown', (event) => {
    mouseDownX.value = event.x;
    mouseMoveX.value = event.x;
    // 左键按下后，才监听鼠标移动，降低资源消耗
    dashBox.value.addEventListener('mousemove', grabbingHandler);
  });
  // 监听鼠标左键抬起事件
  dashBox.value.addEventListener('mouseup', (event) => {
    dashBox.value.removeEventListener('mousemove', grabbingHandler);
    // 判断表盘位移量是否达到对应表盘切换点，需要切换表盘
    if (currentDash.value === 'digital' && deltaX.value < dashes['digital'].deltaX.switchPnt) {
      currentDash.value = 'analog';
    } else if (currentDash.value === 'analog' && deltaX.value > dashes['analog'].deltaX.switchPnt) {
      currentDash.value = 'digital';
    }
    // 置0鼠标位移，拖动状态变为 false
    mouseDownX.value = 0;
    mouseMoveX.value = 0;
  });
});
// +----------------------------------------+
// |               表盘点击切换               |
// +----------------------------------------+
const switchToDigital = () => currentDash.value = 'digital';
const switchToAnalog = () => currentDash.value = 'analog';
const isDigitalDash = computed(() => currentDash.value === 'digital');
const isAnalogDash = computed(() => currentDash.value === 'analog');
const switcher = ref(null);
const halfWindowX = ref(window.innerWidth / 2);
const switcherMouseX = ref(0);
const switcherJudgment = computed(() => {
  if (switcherMouseX.value === 0) return currentDash.value;
  if (switcherMouseX.value <= halfWindowX.value) {
    return 'digital';
  } else {
    return 'analog';
  }
});
onMounted(() => {
  // 响应式获取 window 的中线坐标
  window.addEventListener("resize", () => halfWindowX.value = window.innerWidth / 2);
  // 鼠标移动事件监听处理函数，单拎出来方便取消监听
  const movingHandler = (event) => {
    switcherMouseX.value = event.x;
    currentDash.value = switcherJudgment.value;
  };
  
  switcher.value.addEventListener('mousedown', (event) => {
    switcherMouseX.value = event.x;
    currentDash.value = switcherJudgment.value;
    switcher.value.addEventListener('mousemove', movingHandler);
  });

  switcher.value.addEventListener("mouseup", (event) => {
    switcher.value.removeEventListener('mousemove', movingHandler);
    switcherMouseX.value = 0;
  });
});
</script>



<style lang="scss">
@use "sass:math";

@font-face {
  font-family: "SF Pro Text";
  font-weight: 400;
  src: url("../assets/fonts/SF-Pro-Text-Regular.otf") format("opentype");
}
@font-face{
  font-family: "SF Pro Text";
  font-weight: 300;
  src: url("../assets/fonts/SF-Pro-Text-Light.otf") format("opentype");
}
@font-face {
  font-family: "SF Pro Text";
  font-weight: 200;
  src: url("../assets/fonts/SF-Pro-Text-Thin.otf") format("opentype");
}

$component-top: 30px;
$dash-size: 300px;
$component-width: $component-top + $dash-size;

// 全局样式设定
:root {
  font-family: "SF Pro Text";
  font-weight: 400;
  font-size: 13px;  // 给 rem 单位设定基准
  font-variant-numeric: tabular-nums;  // 使得数字等宽
  color: #fff;
}

body {
  padding: 0;
  margin: 0;
  background-color: #000;
  min-width: $component-width;
  overflow: hidden;  // 阻止组件除[计次记录]外随鼠标滚轮滚动
}

#stopwatch {
  margin: 0 auto;
  padding-top: $component-top;
  width: $component-width;
  height: calc(100vh - #{$component-top});
  position: relative;
}

// 秒表表盘样式
#stopwatch-dashes {
  width: 100%;
  height: $dash-size;
  position: relative;
  user-select: none;
  overflow: hidden;

  @mixin dash-base() {
    padding: 0 math.div($component-top, 2);
    box-sizing: border-box;
    width: 100%;
    height: $dash-size;
    position: absolute;
    top: 0;
    left: 0;
  }
  
  #digital-dash {
    @include dash-base();
    display: flex;
    justify-content: center;
    align-items: center;

    span {
      font-weight: 200;
    }
  }

  #analog-dash {
    @include dash-base();

    @mixin dash-digits($num, $radius, $yOffset:0) {
      margin: 0;
      padding: 0;
      position: absolute;
      left: 50%;
      top: 50%;
      transform: translate(-50%, -50%);
      list-style-type: none;

      $x: 0;
      $y: 0 - $yOffset;
      $unitDegree: math.div(360, $num);

      @for $i from 1 to $num + 1 {
        li:nth-child(#{$i}) {
          position: absolute;
          left: 50%;
          top: 50%;
          transform:
            translate(-50%, -50%)
            translateX($x + $radius * math.cos(math.div(($unitDegree * $i - 90)* math.$pi, 180)))
            translateY($y + $radius * math.sin(math.div(($unitDegree * $i - 90) * math.$pi, 180)));
        }
      }
    }

    .large-dash-digits {
      @include dash-digits(12, 118px, 1px);
      font-size: 1.8rem;
      line-height: 1.8rem;
    }

    .small-dash-digits {
      @include dash-digits(6, 23px, 56px);
      font-size: .9rem;
      line-height: .9rem;

    }

    .small-digital-dash {
      position: absolute;
      top: 69%;
      left: 50%;
      transform: translate(-50%, -50%);
      font-size: 1.4rem;
    }

    svg {
      position: relative;
      z-index: 10;
    }
  }
}

// 秒表按钮样式
#stopwatch-buttons {
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: relative;
  top: -25px;
  z-index: 20;

  .round-buttons {
    background-color: #000;
    border-radius: 50%;

    button {
      display: inline-block;
      position: relative;
      width: 70px;
      height: 70px;
      white-space: nowrap;  // 使得按钮内的文本不自动换行
      background-clip: padding-box;  // 背景色只沿伸到内边距框，从而使同色双线边框露出中间的透明
      border-radius: 50%;  // 按钮变成圆形
      border-style: double;
      border-width: 6px;
      cursor: pointer;
      font-size: 1rem;
      font-weight: 300;

      span {
        position: absolute;  // 绝对定位，隐试增加 display: inline-block 属性
        top: 50%;  // 将上边定位到容纳块（按钮）中间
        left: 50%;  // 将左边定位到容纳块（按钮）中间
        transform: translate(-50%, -50%);  // 将文本向上、左移动半个自身宽、长，继而将文本中点对齐按钮中点
      }

      @mixin buttonBgc($color) {
        background-color: $color;
        border-color: $color;
      }
      
      @mixin buttonColor($baseColor) {
        color: $baseColor;
        @include buttonBgc(rgba($baseColor, .2));

        // 鼠标事件，按钮文本不变色，按钮背景变色
        &:hover {
          @include buttonBgc(rgba($baseColor, .25));
        }

        &:active {
          @include buttonBgc(rgba($baseColor, .1));
        }

        // 按钮禁用，其文本变色，背景不变色
        &:disabled {
          color: rgba($baseColor, .3);
        }
      }

      &.right-button.stopped, &.right-button.reset {
        @include buttonColor(#40cc64);
      }

      &.right-button.started {
        @include buttonColor(#cc4064);
      }

      &.left-button {
        @include buttonColor(#fff);
      }
    }
  }

  .switch-button {
    height: 20px;
    width: 45px;
    border-radius: 20px;
    box-sizing: border-box;
    padding: 0 4px;
    display: flex;
    justify-content: space-evenly;
    align-items: center;
    background-color: transparent;

    &:hover {
      background-color: #333;
    }

    div {
      $diameter: 6px;

      height: $diameter;
      width: $diameter;
      border-radius: $diameter;
      background-color: #888;
      transition: background-color .1s ease-in-out;

      &.highlight {
        background-color: #fff;
        transition: background-color 0s;
      }
    }
  }
}

// 秒表计次记录样式
#stopwatch-laps {
  $bleeding: 10px;
  $lineHeight: 2.4rem;
  $lineColor: #555;
  $offsetY: 70px;

  position: absolute;  // 绝对定位以确保高度响应式变化
  top: $component-width + $offsetY;
  bottom: 0;
  left: 0;
  right: 0;
  border-top: 1px solid $lineColor;
  

  ul {
    list-style-type: none;
    box-sizing: border-box;
    margin: 0;
    margin-left: - $bleeding;  // 向左移动1倍出血宽，居中框体
    padding: 0;
    padding: 0 $bleeding;
    height: 100%;
    width: calc(100% + #{$bleeding * 2});  // 沿伸2倍出血宽，给滚动条留位置
    overflow-y: scroll;
    background-image: repeating-linear-gradient(180deg,
      transparent 0,
      transparent calc(#{$lineHeight} - 1px),
      $lineColor calc(#{$lineHeight} - 1px),
      $lineColor $lineHeight
    );
    background-clip: content-box;
    background-attachment: local;
    
    li {
      display: flex;
      justify-content: space-between;
      line-height: $lineHeight;

      &.max {
        color: #cc4064;
      }

      &.min {
        color: #40cc64;
      }

      span:first-child {
        font-variant-numeric: normal;
      }
    }
  }
}
</style>
