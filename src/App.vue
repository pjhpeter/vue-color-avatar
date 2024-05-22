<template>
  <img :src="image" />
  <VueColorAvatar
    :style="{
      transform: 'rotateY(-180deg)',
    }"
    @image-change="imageChange"
  />
</template>

<script lang="ts" setup>
import { ref } from 'vue'

import VueColorAvatar from '@/components/VueColorAvatar.vue'

const image = ref('')
const imageChange = (img: string) => (image.value = img)
</script>
