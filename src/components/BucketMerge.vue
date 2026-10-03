<template lang="pug">
div
  h3 合并存储桶
  p.small
    | 有时你可能需要把两个存储桶中的事件合并到同一个存储桶里。
    | 例如主机名发生变化后，同一个 watcher 可能生成两个不同的存储桶，此工具可以把它们重新合并。

  b-row
    b-col
      h4 来源存储桶
      b-form-select(v-model="bucket_from" :options="buckets" :disabled="buckets.length === 0")
      p.small
        | 选择要迁出事件的存储桶。
        | 合并完成后，这个来源存储桶不会自动删除。
      p.small(v-if="events_from !== null")
        | 事件数：{{ events_from.length }}
    b-col
      h4 目标存储桶
      b-form-select(v-model="bucket_to" :options="buckets" :disabled="buckets.length === 0")
      p.small
        | 选择要接收事件的存储桶。
        | 合并后的事件会写入这里。
      p.small(v-if="events_to !== null")
        | 事件数：{{ events_to.length }}

  div(v-if="overlappingEvents !== null && overlappingEvents.length > 0")
    h3 重叠事件
    p
      | 发现 {{ overlappingEvents.length }} 组相互重叠的事件：
      ul
        li(v-for="event in overlappingEvents")
          | {{ event[0].start }} - {{ event[0].end }}（{{ event[0].event.id }}）
          | 与
          | {{ event[1].start }} - {{ event[1].end }}（{{ event[1].event.id }}）重叠

  b-button(variant="success" :disabled="!validate" @click="merge()") 合并
</template>

<script lang="ts">
import { getClient } from '~/util/awclient';
import { overlappingEvents } from '~/util/transforms';

export default {
  name: 'aw-bucket-merge',
  data() {
    return {
      buckets: [],
      bucket_from: null,
      bucket_to: null,
      events_from: null,
      events_to: null,
    };
  },
  computed: {
    validate() {
      const set = this.bucket_from !== null && this.bucket_to !== null;
      const not_same = this.bucket_from !== this.bucket_to;
      const not_overlapping =
        this.overlappingEvents !== null && this.overlappingEvents.length === 0;
      return set && not_same && not_overlapping;
    },
    overlappingEvents() {
      if (this.events_from === null || this.events_to === null) {
        return null;
      }
      return overlappingEvents(this.events_from, this.events_to);
    },
  },
  watch: {
    bucket_from: async function (new_bucket_id) {
      this.events_from = await this.getEvents(new_bucket_id);
    },
    bucket_to: async function (new_bucket_id) {
      this.events_to = await this.getEvents(new_bucket_id);
    },
  },
  mounted() {
    this.getBuckets();
  },
  methods: {
    merge: async function () {
      const client = getClient();
      const events = this.events_from;
      const bucket_id = this.bucket_to;
      const result = await client.insertEvents(bucket_id, events);
      console.log('result', result);
    },
    getBuckets: async function () {
      const client = getClient();
      const buckets = await client.getBuckets();
      this.buckets = Object.keys(buckets).map(bucket_id => {
        return { value: bucket_id, text: bucket_id };
      });
    },
    getEvents: async function (bucket_id) {
      const client = getClient();
      return await client.getEvents(bucket_id);
    },
  },
};
</script>
