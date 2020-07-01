<template>
  <div
    v-show="ready"
    class="el-carousel__item"
    :class="{
      'is-active': active,
      'el-carousel__item--card': $parent.type === 'card',
      'is-in-stage': inStage,
      'is-hover': hover,ElCarouselItem
      'is-animating': animating
    }"
    @click="handleItemClick"
    :style="itemStyle"
  >
    <!-- 卡片化才有 -->
    <!-- <div
      v-if="$parent.type === 'card'"
      v-show="!active"
      class="el-carousel__mask"
    >
    </div> -->
    <slot></slot>
  </div>
</template>

<script>
import { autoprefixer } from "element-ui/src/utils/util";
const CARD_SCALE = 0.83;
export default {
  name: "ElCarouselItem",

  props: {
    name: String,
    label: {
      type: [String, Number],
      default: ""
    }
  },

  data() {
    return {
      hover: false,
      translate: 0,
      scale: 1,
      active: false,
      ready: false,
      inStage: false,
      animating: false
    };
  },

  methods: {
    processIndex(index, activeIndex, length) {
      // 当前是activeIndex是第一张，index是最后一张，返回-1，相差-1，表示二者相邻且index在左侧
      if (activeIndex === 0 && index === length - 1) {
        return -1;
      } else if (activeIndex === length - 1 && index === 0) {
        // 当前页activeIndex是最后一张，index是第一张，返回length，相差1，表示二者相邻且index在右侧
        return length;
        // 如果，index在activeIndex前一页的前面，并且之间的间隔在一半页数即以上，则返回页数长度+1，这样它们会被置于最右侧
      } else if (index < activeIndex - 1 && activeIndex - index >= length / 2) {
        return length + 1;
        // 如果，index在activeIndex后一页的后面，并且之间的间隔在一般页数即以上，则返回-2，这样它们会被置于最左侧
      } else if (index > activeIndex + 1 && index - activeIndex >= length / 2) {
        return -2;
      }
      return index;
    },

    calcCardTranslate(index, activeIndex) {
      const parentWidth = this.$parent.$el.offsetWidth;
      if (this.inStage) {
        return (
          (parentWidth * ((2 - CARD_SCALE) * (index - activeIndex) + 1)) / 4
        );
      } else if (index < activeIndex) {
        return (-(1 + CARD_SCALE) * parentWidth) / 4;
      } else {
        return ((3 + CARD_SCALE) * parentWidth) / 4;
      }
    },

    /**
     * 当activeIndex从3变成4，index=0的元素translate = -3*width => -4*width（左移）
     */
    calcTranslate(index, activeIndex, isVertical) {
      const distance = this.$parent.$el[
        isVertical ? "offsetHeight" : "offsetWidth"
      ];
      return distance * (index - activeIndex);
    },

    /**
     * 需要轮播时，由外层carousel调用translateItem，触发元素移动
     * @param index 当前carousel-item索引
     * @param activeIndex 轮播需要显示在正中间的carousel-item索引
     * @param oldIndex 过去的activeIndex
     * 所有元素都会重新计算自己的translate值，生成动态样式
     * (列表中只有两个元素需要移动，旧的activeIndex（左移移走），新的activeIndex（左移移入）？？？)
     */
    translateItem(index, activeIndex, oldIndex) {
      const parentType = this.$parent.type;
      const parentDirection = this.parentDirection;
      // 图片个数
      const length = this.$parent.items.length;

      // 如果不是card模式，修改当前状态（是否移动）
      if (parentType !== "card" && oldIndex !== undefined) {
        this.animating = index === activeIndex || index === oldIndex;
      }

      // 循环播放
      if (index !== activeIndex && length > 2 && this.$parent.loop) {
        // 处理当前索引
        index = this.processIndex(index, activeIndex, length);
      }

      // 更新当前元素active状态（是否为activeIndex），计算当前元素的Translate并修改 => 触发新的style计算，动态样式
      if (parentType === "card") {
        if (parentDirection === "vertical") {
          console.warn(
            "[Element Warn][Carousel]vertical direction is not supported in card mode"
          );
        }
        this.inStage = Math.round(Math.abs(index - activeIndex)) <= 1;
        this.active = index === activeIndex;
        this.translate = this.calcCardTranslate(index, activeIndex);
        this.scale = this.active ? 1 : CARD_SCALE;
      } else {
        this.active = index === activeIndex;
        const isVertical = parentDirection === "vertical";
        this.translate = this.calcTranslate(index, activeIndex, isVertical);
      }
      this.ready = true;
    },

    /**
     * 在card模式下点击切换
     */
    handleItemClick() {
      const parent = this.$parent;
      if (parent && parent.type === "card") {
        const index = parent.items.indexOf(this);
        parent.setActiveItem(index);
      }
    }
  },

  computed: {
    /**
     * 计算属性，返回el-carousel上的type属性【不用多次写】
     */
    parentDirection() {
      return this.$parent.direction;
    },

    /**
     * translate的值发生改变就会自动执行，计算样式，返回一个style对象，动态样式
     */
    itemStyle() {
      const translateType =
        this.parentDirection === "vertical" ? "translateY" : "translateX";
      const value = `${translateType}(${this.translate}px) scale(${this.scale})`;
      const style = {
        transform: value
      };
      return autoprefixer(style);
    }
  },

  created() {
    this.$parent && this.$parent.updateItems();
  },

  destroyed() {
    this.$parent && this.$parent.updateItems();
  }
};
</script>
