<template>
  <div
    :class="carouselClasses"
    @mouseenter.stop="handleMouseEnter"
    @mouseleave.stop="handleMouseLeave"
  >
    <!-- 箭头 -->
    <div
      class="el-carousel__container"
      :style="{ height: height }"
    >
      <!-- <transition
                v-if="arrowDisplay"
                name="carousel-arrow-left"
            >
                <button
                    type="button"
                    v-show="(arrow === 'always' || hover) && (loop || activeIndex > 0)"
                    @mouseenter="handleButtonEnter('left')"
                    @mouseleave="handleButtonLeave"
                    @click.stop="throttledArrowClick(activeIndex - 1)"
                    class="el-carousel__arrow el-carousel__arrow--left"
                >
                    <i class="el-icon-arrow-left"></i>
                </button>
            </transition>
            <transition
                v-if="arrowDisplay"
                name="carousel-arrow-right"
            >
                <button
                    type="button"
                    v-show="(arrow === 'always' || hover) && (loop || activeIndex < items.length - 1)"
                    @mouseenter="handleButtonEnter('right')"
                    @mouseleave="handleButtonLeave"
                    @click.stop="throttledArrowClick(activeIndex + 1)"
                    class="el-carousel__arrow el-carousel__arrow--right"
                >
                    <i class="el-icon-arrow-right"></i>
                </button>
            </transition> -->
      <slot></slot>
    </div>
    <!-- 下面一行的指示器 -->
    <!-- <ul
            v-if="indicatorPosition !== 'none'"
            :class="indicatorsClasses"
        >
            <li
                v-for="(item, index) in items"
                :key="index"
                :class="[
          'el-carousel__indicator',
          'el-carousel__indicator--' + direction,
          { 'is-active': index === activeIndex }]"
                @mouseenter="throttledIndicatorHover(index)"
                @click.stop="handleIndicatorClick(index)"
            >
                <button class="el-carousel__button">
                    <span v-if="hasLabel">{{ item.label }}</span>
                </button>
            </li>
        </ul> -->
  </div>
</template>

<script>
import throttle from "throttle-debounce/throttle";
import {
  addResizeListener,
  removeResizeListener
} from "element-ui/src/utils/resize-event";

export default {
  name: "ElCarousel",

  props: {
    initialIndex: {
      type: Number,
      default: 0
    },
    height: String,
    trigger: {
      type: String,
      default: "hover"
    },
    autoplay: {
      type: Boolean,
      default: true
    },
    interval: {
      type: Number,
      default: 3000
    },
    indicatorPosition: String,
    indicator: {
      type: Boolean,
      default: true
    },
    arrow: {
      type: String,
      default: "hover"
    },
    type: String,
    loop: {
      type: Boolean,
      default: true
    },
    direction: {
      type: String,
      default: "horizontal",
      validator(val) {
        return ["horizontal", "vertical"].indexOf(val) !== -1;
      }
    }
  },

  data() {
    return {
      items: [],
      activeIndex: -1,
      containerWidth: 0,
      timer: null,
      hover: false
    };
  },

  computed: {
    arrowDisplay() {
      return this.arrow !== "never" && this.direction !== "vertical";
    },

    hasLabel() {
      return this.items.some(item => item.label.toString().length > 0);
    },

    /**
     * 根据direction、type，动态生成类名 el-carousel el-carousel--card
     */
    carouselClasses() {
      const classes = ["el-carousel", "el-carousel--" + this.direction];
      if (this.type === "card") {
        classes.push("el-carousel--card");
      }
      return classes;
    },

    indicatorsClasses() {
      const classes = [
        "el-carousel__indicators",
        "el-carousel__indicators--" + this.direction
      ];
      if (this.hasLabel) {
        classes.push("el-carousel__indicators--labels");
      }
      if (this.indicatorPosition === "outside" || this.type === "card") {
        classes.push("el-carousel__indicators--outside");
      }
      return classes;
    }
  },

  watch: {
    /**
     * items发生改变，就将外层传入的initialIndex设置为ActiveItem
     */
    items(val) {
      if (val.length > 0) this.setActiveItem(this.initialIndex);
    },

    /**
     * 如果activeIndex的值发生改变，就会调用resetItemPosition，传入旧值，调整所有carousel-item位置
     */
    activeIndex(val, oldVal) {
      this.resetItemPosition(oldVal);
      if (oldVal > -1) {
        this.$emit("change", val, oldVal);
      }
    },

    /**
     * 根据是否自动播放的值，来决定开启、关闭定时器
     */
    autoplay(val) {
      val ? this.startTimer() : this.pauseTimer();
    },

    loop() {
      this.setActiveItem(this.activeIndex);
    }
  },

  methods: {
    /**
     * 鼠标移动到轮播图上时，不转动
     */
    handleMouseEnter() {
      this.hover = true;
      this.pauseTimer();
    },
    handleMouseLeave() {
      this.hover = false;
      this.startTimer();
    },

    /**
     * 根据是否在边界上，返回'left||right||false'
     */
    itemInStage(item, index) {
      const length = this.items.length;
      if (
        (index === length - 1 && item.inStage && this.items[0].active) ||
        (item.inStage && this.items[index + 1] && this.items[index + 1].active)
      ) {
        return "left";
      } else if (
        (index === 0 && item.inStage && this.items[length - 1].active) ||
        (item.inStage && this.items[index - 1] && this.items[index - 1].active)
      ) {
        return "right";
      }
      return false;
    },

    /**
     * 鼠标是否悬浮在arrow上，arrow = 'left||right'
     */
    handleButtonEnter(arrow) {
      if (this.direction === "vertical") return;
      this.items.forEach((item, index) => {
        if (arrow === this.itemInStage(item, index)) {
          item.hover = true;
        }
      });
    },

    handleButtonLeave() {
      if (this.direction === "vertical") return;
      this.items.forEach(item => {
        item.hover = false;
      });
    },

    /**
     * el-carousel内除了el-carousel-item可能还有其他子组件，这里做一层过滤
     */
    updateItems() {
      this.items = this.$children.filter(
        child => child.$options.name === "ElCarouselItem"
      );
    },

    /**
     * 根据传入的参数oldIndex，遍历调整所有的carousel-item的位置
     */
    resetItemPosition(oldIndex) {
      this.items.forEach((item, index) => {
        item.translateItem(index, this.activeIndex, oldIndex);
      });
    },

    /**
     * 轮播（activeIndex+1||=0）
     */
    playSlides() {
      if (this.activeIndex < this.items.length - 1) {
        this.activeIndex++;
      } else if (this.loop) {
        this.activeIndex = 0;
      }
    },

    pauseTimer() {
      if (this.timer) {
        clearInterval(this.timer);
        this.timer = null;
      }
    },

    /**
     * 定时器，每隔this.interval就滚一次
     */
    startTimer() {
      if (this.interval <= 0 || !this.autoplay || this.timer) return;
      this.timer = setInterval(this.playSlides, this.interval);
    },

    /**
     * 手动切换幻灯片（根据name属性/索引）=> 是否需要修改activeIndex => 触发resetItemPosition，通知item位置变化
     */
    setActiveItem(index) {
      // name属性 => 索引
      if (typeof index === "string") {
        const filteredItems = this.items.filter(item => item.name === index);
        if (filteredItems.length > 0) {
          index = this.items.indexOf(filteredItems[0]);
        }
      }
      // 索引
      index = Number(index);
      if (isNaN(index) || index !== Math.floor(index)) {
        console.warn("[Element Warn][Carousel]index must be an integer.");
        return;
      }
      let length = this.items.length;
      const oldIndex = this.activeIndex;
      // 如果传入的index超出列表边界，就根据“是否循环播放”来设置activeIndex值，会自动触发resetItemPosition
      if (index < 0) {
        this.activeIndex = this.loop ? length - 1 : 0;
      } else if (index >= length) {
        this.activeIndex = this.loop ? 0 : length - 1;
      } else {
        this.activeIndex = index;
      }
      // 手动触发resetItemPosition
      if (oldIndex === this.activeIndex) {
        this.resetItemPosition(oldIndex);
      }
    },

    prev() {
      this.setActiveItem(this.activeIndex - 1);
    },

    next() {
      this.setActiveItem(this.activeIndex + 1);
    },

    handleIndicatorClick(index) {
      this.activeIndex = index;
    },

    handleIndicatorHover(index) {
      if (this.trigger === "hover" && index !== this.activeIndex) {
        this.activeIndex = index;
      }
    }
  },

  created() {
    this.throttledArrowClick = throttle(300, true, index => {
      this.setActiveItem(index);
    });
    this.throttledIndicatorHover = throttle(300, index => {
      this.handleIndicatorHover(index);
    });
  },

  /**
   * 过滤掉非carouser-item元素，在nextTick中，添加resize监听器，初始化activeIndex后，开启定时器
   */
  mounted() {
    this.updateItems();
    this.$nextTick(() => {
      addResizeListener(this.$el, this.resetItemPosition);
      if (this.initialIndex < this.items.length && this.initialIndex >= 0) {
        this.activeIndex = this.initialIndex;
      }
      this.startTimer();
    });
  },

  beforeDestroy() {
    if (this.$el) removeResizeListener(this.$el, this.resetItemPosition);
    this.pauseTimer();
  }
};
</script>
