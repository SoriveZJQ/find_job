<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(
  defineProps<{
    flag: boolean
    footer?: boolean
    title?: string
    confirmText?: string
    resetText?: string
    width?: string
  }>(),
  {
    title: '温馨提示',
    width: '30%',
    confirmText: '提交',
    resetText: '重置',
    footer: true
  }
)
const emit = defineEmits(['cancel', 'confirm', 'update:flag'])

const dialogVisible = computed({
  get() {
    return props.flag
  },
  set(visible: boolean) {
    emit('update:flag', visible)
  }
})
</script>

<template>
  <el-dialog
    v-model="dialogVisible"
    :title="title"
    :style="{ width }"
    align-center
    destroy-on-close
  >
    <slot></slot>
    <template #footer v-if="footer">
      <span class="dialog-footer">
        <button class="primary mr-20 btn" @click="$emit('confirm')">{{ confirmText }}</button>
        <button class="primary btn" @click="$emit('cancel')">{{ resetText }}</button>
      </span>
    </template>
  </el-dialog>
</template>
