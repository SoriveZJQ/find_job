<script setup lang="ts">
import { useTypeNet } from './hook'
import Header from './components/header.vue'
import { useDark } from '@vueuse/core'
import { themes } from '@/templates/config'

useTypeNet()
const isDark = useDark()
</script>

<template>
  <Header />
  <div class="introduce flex flex-space-around flex-align-around flex-align-center">
    <div class="introduce-l" data-aos="slide-right">
      <div class="typenet-text"></div>
      <button
        @click="$router.push('/theme')"
        :class="['start btn pointer', { 'dark-start': isDark }]"
      >
        马上开始 <i class="iconfont icon-goto"></i>
      </button>
    </div>
    <div class="introduce-r" data-aos="slide-left">
      <img
        :class="{ 'img-light': !isDark }"
        :style="{ transform: `rotate(${40 - idx * 20}deg)`, zIndex: 999 - idx }"
        v-for="(theme, idx) in themes.slice(2, 7)"
        :src="theme.img"
        :alt="theme.name"
        :key="theme.id"
      />
    </div>
  </div>
</template>

<style lang="scss" scoped>
html,
body {
  height: 100%;
  width: 100%;
}
.introduce {
  color: var(--font-color);
  height: 100vh;
  padding: 20px;
  position: relative;
  background: var(--background);

  .introduce-l,
  img {
    user-select: none;
    -webkit-user-drag: none;
  }
  .introduce-r {
    width: 250px;
    height: 350px;
    img {
      width: 250px;
      height: 350px;
      transform-origin: left bottom;
      position: absolute;
      border-radius: 10px;
      transition: transform 0.3s;
      &:nth-child(1) {
        animation: rotate1 0.7s;
      }
      &:nth-child(2) {
        animation: rotate2 0.7s;
      }
      &:nth-child(4) {
        animation: rotate4 0.7s;
      }
      &:nth-child(5) {
        animation: rotate5 0.7s;
      }
    }
    .img-light {
      box-shadow: 5px 5px 8px #ddd;
    }
  }

  .introduce-l {
    z-index: 2;
    .typenet-text {
      width: 500px;
      height: 300px;
    }
    .start {
      font-size: 1.1rem;
      padding: 10px 20px 10px 25px;
      border-radius: 40px;
      background: #000;
      margin-top: 10px;
      color: white;
      &:hover {
        transition: transform 0.4s;
        transform: translateY(5px);
        opacity: 0.8;
      }
    }
    .dark-start {
      background: #ff7449;
    }
  }

  @keyframes rotate1 {
    from {
      transform: rotate(20deg);
    }
    to {
      transform: rotate(40deg);
    }
  }
  @keyframes rotate2 {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(20deg);
    }
  }
  @keyframes rotate4 {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(-20deg);
    }
  }
  @keyframes rotate5 {
    from {
      transform: rotate(-20deg);
    }
    to {
      transform: rotate(-40deg);
    }
  }
}

@media screen and (max-width: 800px) {
  .introduce {
    .introduce-r {
      display: none;
    }
  }
}
</style>
