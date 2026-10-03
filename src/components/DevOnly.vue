<template lang="pug">
div(v-if="show", style="border: 1px solid #aaa; border-radius: 5px")
  b-alert(v-if="note", variant="warning" show)
    span.float-left
      | 此区域不会出现在正式构建中
      span(v-if="reason")
        | （{{ reason }}）
    b-btn.float-right.hide-devonly(@click="() => { hide = true }", variant="outline-secondary", size="sm")
      | 隐藏
    | 。
  slot
</template>

<script lang="ts">
export default {
  props: {
    note: {
      type: Boolean,
      default: true,
    },
    reason: {
      type: String,
      default: null,
    },
  },
  data: () => {
    return { hide: false };
  },
  computed: {
    show() {
      return !PRODUCTION && !this.hide;
    },
  },
};
</script>
