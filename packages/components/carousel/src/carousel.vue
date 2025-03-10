<script lang="ts" setup>
  import { reactive, onMounted, onBeforeUnmount } from 'vue';
  import { ICarouselProps } from './carousel';
  import Dot from './dot.vue';
  import Director from './director.vue';

  //组件接收的值类型
  const carouselProps = defineProps(ICarouselProps);
  const state = reactive({
    currentIndex: carouselProps.initial,
    itemLen: carouselProps.options.length,
  });

  let t = null;

  //自动轮播
  const autoPlay = () => {
    if (carouselProps.autoplay) {
      t = setInterval(() => {
        setIndex('next');
        // console.log(state.currentIndex);
      }, carouselProps.duration);
    }
  };
  //获取当前的图片Index
  const setIndex = (dir: String) => {
    switch (dir) {
      case 'next':
        state.currentIndex = state.currentIndex + 1;
        if (state.currentIndex === state.itemLen) {
          state.currentIndex = 0;
        }
        break;
      case 'prev':
        state.currentIndex = state.currentIndex - 1;
        if (state.currentIndex === -1) {
          state.currentIndex = state.itemLen - 1;
        }
        break;
    }
  };

  const dotClick = (index) => {
    state.currentIndex = index;
  };

  const dirClick = (dir) => {
    setIndex(dir);
  };

  function _clearIntervalFn() {
    clearInterval(t);
    t = null;
  }

  const mouseEnter = () => {
    _clearIntervalFn();
  };

  const mouseLeave = () => {
    autoPlay();
  };

  onMounted(() => {
    autoPlay();
  });

  onBeforeUnmount(() => {
    _clearIntervalFn();
  });

  console.log(carouselProps.options.length);
</script>

<template>
  <div class="carousel" @mouseenter="mouseEnter" @mouseleave="mouseLeave">
    <div class="inner">
      <div v-for="(item, index) of carouselProps.options" :key="index" class="car-item">
        <transition>
          <img v-if="item.id === state.currentIndex" :src="item.imgUrl" alt="" />
        </transition>
        <Dot
          :has-dot="carouselProps.hasDot"
          :item-len="state.itemLen"
          :current-index="state.currentIndex"
          @dotClick="dotClick"
        ></Dot>
        <Director dir="prev" @dirClick="dirClick" />
        <Director dir="next" @dirClick="dirClick" />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .carousel {
    width: 100%;
    height: 100%;
    overflow: hidden;
    .inner {
      width: 100%;
      height: 100%;
      position: relative;

      .car-item {
        position: absolute;
        top: 0;
        left: 0;
      }
      .v-enter-active,
      .v-leave-active {
        transition: all 0.8s linear;
      }
      .v-enter-active {
        transform: translateX(100%);
      }
      .v-enter-to {
        transform: translateX(0);
      }
      .v-leave-active {
        transform: translateX(0);
      }
      .v-leave-to {
        transform: translateX(-100%);
      }
    }
  }
  img {
    width: 100%;
  }
</style>
