<template>
  <div
    ref="avatarRef"
    class="vue-color-avatar"
    :style="{
      width: `${avatarSize}px`,
      height: `${avatarSize}px`,
      ...getWrapperShapeStyle(),
    }"
    :class="getWrapperShapeClassName()"
    @click="resetAvatarOption"
  >
    <Background :color="avatarOption.background.color" />

    <div class="avatar-payload" v-html="svgContent" />

    <Border
      :color="avatarOption.background.borderColor"
      :radius="getWrapperShapeStyle().borderRadius"
    />
  </div>
</template>

<script lang="ts" setup>
import { ref, shallowRef, toRefs, watch, watchEffect } from 'vue'

import { WidgetType, WrapperShape } from '@/enums'
import type { AvatarOption } from '@/types'
import { getRandomAvatarOption } from '@/utils'
import { AVATAR_LAYER, NONE, SHAPE_STYLE_SET } from '@/utils/constant'
import { widgetData } from '@/utils/dynamic-data'

import Background from './widgets/Background.vue'
import Border from './widgets/Border.vue'

const props = withDefaults(defineProps<{ size?: number }>(), {
  size: 280,
})

const emit = defineEmits(['imageChange'])

const { size: avatarSize } = toRefs(props)

const avatarOption = ref<AvatarOption>(getRandomAvatarOption())

const avatarRef = shallowRef<HTMLDivElement | null>(null)

function getWrapperShapeClassName() {
  return {
    [WrapperShape.Circle]:
      avatarOption.value.wrapperShape === WrapperShape.Circle,
    // [WrapperShape.Square]:
    //   avatarOption.value.wrapperShape === WrapperShape.Square,
    // [WrapperShape.Squircle]:
    //   avatarOption.value.wrapperShape === WrapperShape.Squircle,
  }
}

function getWrapperShapeStyle() {
  return SHAPE_STYLE_SET[avatarOption.value.wrapperShape!]
}

const svgContent = ref('')

function resetAvatarOption() {
  avatarOption.value = getRandomAvatarOption()
}

async function outputImage() {
  const avatarEle = avatarRef.value

  if (avatarEle) {
    const html2canvas = (await import('html2canvas')).default
    const canvas = await html2canvas(avatarEle, {
      backgroundColor: null,
    })
    emit('imageChange', canvas.toDataURL())
  }
}

defineExpose({ resetAvatarOption })

watch(
  () => avatarOption.value,
  (val) => setTimeout(() => outputImage(), 100),
  {
    immediate: true,
  }
)

watchEffect(async () => {
  const sortedList = Object.entries(avatarOption.value.widgets).sort(
    ([prevShape, prev], [nextShape, next]) => {
      const ix = prev.zIndex ?? AVATAR_LAYER[prevShape]?.zIndex ?? 0
      const iix = next.zIndex ?? AVATAR_LAYER[nextShape]?.zIndex ?? 0
      return ix - iix
    }
  )

  // const promises: Promise<string>[] = sortedList.map(async ([widgetType, opt]) => {
  //   return (
  //     await import(`../assets/widgets/${widgetType}/${opt.shape}.svg?raw`)
  //   ).default
  // })

  const promises: Promise<string>[] = sortedList.map(
    async ([widgetType, opt]) => {
      if (opt.shape !== NONE && widgetData?.[widgetType]?.[opt.shape]) {
        return (await widgetData[widgetType][opt.shape]()).default
      }
      return ''
    }
  )

  let skinColor: string | undefined

  const svgRawList = await Promise.all(promises).then((raw) => {
    return raw.map((svgRaw, i) => {
      const [widgetType, widget] = sortedList[i]
      let widgetFillColor = widget.fillColor

      if (widgetType === WidgetType.Face) {
        skinColor = widgetFillColor
      }
      if (skinColor && widgetType === WidgetType.Ear) {
        widgetFillColor = skinColor
      }

      const content = svgRaw
        .slice(svgRaw.indexOf('>', svgRaw.indexOf('<svg')) + 1)
        .replace('</svg>', '')
        .replaceAll('$fillColor', widgetFillColor || 'transparent')

      return `
        <g id="vue-color-avatar-${sortedList[i][0]}">
          ${content}
        </g>
      `
    })
  })

  svgContent.value = `
    <svg
      width="${avatarSize.value}"
      height="${avatarSize.value}"
      viewBox="0 0 ${avatarSize.value / 0.7} ${avatarSize.value / 0.7}"
      preserveAspectRatio="xMidYMax meet"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
    >
      <g transform="translate(100, 65)">
        ${svgRawList.join('')}
      </g>
    </svg>
  `
})
</script>

<style lang="scss" scoped>
.vue-color-avatar {
  position: relative;
  overflow: hidden;

  .avatar-payload {
    position: relative;
    z-index: 2;
    width: 100%;
    height: 100%;
  }
}
</style>
