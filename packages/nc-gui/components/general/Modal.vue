<script lang="ts" setup>
const props = withDefaults(
  defineProps<{
    visible: boolean
    width?: string | number
    size?: 'small' | 'medium' | 'large'
    destroyOnClose?: boolean
    maskClosable?: boolean
    closable?: boolean
  }>(),
  {
    size: 'medium',
    destroyOnClose: true,
    maskClosable: true,
    closable: false,
  },
)

const emits = defineEmits(['update:visible'])

const { width: propWidth, destroyOnClose, closable, maskClosable } = props

const width = computed(() => {
  if (propWidth) {
    return propWidth
  }

  if (props.size === 'small') {
    return '28rem'
  }

  if (props.size === 'medium') {
    return '40rem'
  }

  if (props.size === 'large') {
    return '80rem'
  }

  return 'max(30vw, 600px)'
})

const height = computed(() => {
  if (props.size === 'small') {
    return 'auto'
  }

  if (props.size === 'medium') {
    return '26.5'
  }

  if (props.size === 'large') {
    return '80vh'
  }

  return 'auto'
})

const visible = useVModel(props, 'visible', emits)
</script>

<template>
  <a-modal
    v-model:visible="visible"
    :class="{ active: visible }"
    :width="width"
    :closable="closable"
    wrap-class-name="nc-modal-wrapper"
    :footer="null"
    :destroy-on-close="destroyOnClose"
    :mask-closable="maskClosable"
    @keydown.esc="visible = false"
  >
    <div :class="`nc-modal max-h-[${height}]`">
      <slot />
    </div>
  </a-modal>
</template>

<style lang="scss">
.nc-modal-wrapper {
  .ant-modal-content {
    @apply !p-0;
  }
}
</style>
