<template lang="pug">
div
  h3 校验存储桶
  p.small
    | 用于检查存储桶及其中事件数据是否存在异常。

  b-row
    b-col
      h4 存储桶
      b-form-select(v-model="bucket" :options="buckets" :disabled="buckets.length === 0")
      p.small
        | 选择要校验的存储桶。
      p.small(v-if="events !== null")
        | 事件数：{{ events.length }}

  div(v-if="duplicateEvents !== null")
    details
      summary
        icon.mx-2(name="check", style="color: #0C0", v-if="duplicateEvents.length === 0")
        icon.mx-2(name="exclamation-triangle", style="color: #CC0", v-else)
        | 重复事件：{{ duplicateEvents.length }}
      div.p-2
        p(v-if="duplicateEvents.length === 0")
          | 未发现重复事件。
        p(v-else)
          | 共发现 {{ duplicateEvents.length }} 组重复事件。
          ul.mt-2
            li(v-for="overlap in duplicateEvents")
              | {{ overlap[0].start.toISOString() }} - (ID: {{ overlap[0].event.id }} & {{ overlap[1].event.id }}): {{ JSON.stringify(overlap[0].data) }}

  div(v-if="overlappingEvents !== null")
    details
      summary
        icon.mx-2(name="check", style="color: #0C0", v-if="overlappingEvents.length === 0")
        icon.mx-2(name="exclamation-triangle", style="color: #CC0", v-else)
        | 重叠事件：{{ overlappingEvents.length }} 组，总重叠时长 {{ overlapDuration / 1000 | friendlyduration }}
      div.p-2
        p(v-if="overlappingEvents.length === 0")
          | 未发现重叠事件。
        p(v-else)
          | 共发现 {{ overlappingEvents.length }} 组重叠事件。
          br
          span(v-if="overlapDurationSameData > 0")
            | 其中有 {{ overlapDurationSameData / 1000 | friendlyduration }} 的重叠事件数据完全相同，这部分事件可能可以安全合并。
          p.mt-2(v-for="event in overlappingEvents")
            ul
              li {{ event[0].start.toISOString() }}/{{ event[0].end.toISOString() }} - (ID: {{ event[0].event.id }}): {{ JSON.stringify(event[0].event.data) }}
              li {{ event[1].start.toISOString() }}/{{ event[1].end.toISOString() }} - (ID: {{ event[1].event.id }}): {{ JSON.stringify(event[1].event.data) }}

  div(v-if="zeroDurationEvents !== null")
    details
      summary
        icon.mx-2(name="check", style="color: #0C0", v-if="zeroDurationEvents.length === 0")
        icon.mx-2(name="info-circle", style="color: #09F", v-else)
        | 零时长事件：{{ zeroDurationEvents.length }}
      div.p-2
        p.ml-3(v-if="zeroDurationEvents.length === 0")
          | 未发现零时长事件。
        p.ml-3(v-else)
          | 共发现 {{ zeroDurationEvents.length }} 个零时长事件：
          ul.mt-2
            li(v-for="event in zeroDurationEvents")
              | {{ event.timestamp.toISOString() }}/{{ new Date(new Date(event.timestamp).valueOf() + 1000 * event.duration).toISOString() }} - (ID: {{ event.id }}): {{ JSON.stringify(event.data) }}
</template>

<script lang="ts">
import 'vue-awesome/icons/check';
import 'vue-awesome/icons/exclamation-triangle';
import 'vue-awesome/icons/info-circle';
import { getClient } from '~/util/awclient';
import { overlappingEvents } from '~/util/transforms';
import _ from 'lodash';

export default {
  name: 'aw-bucket-merge',
  data() {
    return {
      buckets: [],
      bucket: null,
      events: null,
    };
  },
  computed: {
    validate() {
      const set = this.bucket !== null;
      const not_overlapping =
        this.overlappingEvents !== null && this.overlappingEvents.length === 0;
      return set && not_overlapping;
    },
    duplicateEvents() {
      if (this.overlappingEvents === null) {
        return null;
      }
      return this.overlappingEvents.filter(overlap => {
        if (
          _.isEqual(overlap[0].start, overlap[1].start) &&
          _.isEqual(overlap[0].end, overlap[1].end) &&
          _.isEqual(overlap[0].event.data, overlap[1].event.data)
        ) {
          return true;
        }
      });
    },
    overlappingEvents() {
      if (this.events === null) {
        return null;
      }
      return overlappingEvents(this.events, this.events);
    },
    overlapDuration() {
      if (this.overlappingEvents === null) {
        return null;
      }
      return this.overlappingEvents.reduce((acc, event) => {
        const start = event[0].start > event[1].start ? event[0].start : event[1].start;
        const end = event[0].end < event[1].end ? event[0].end : event[1].end;
        return acc + (end - start);
      }, 0);
    },
    overlapDurationSameData() {
      if (this.overlappingEvents === null) {
        return null;
      }
      return this.overlappingEvents.reduce((acc, event) => {
        if (!_.isEqual(event[0].event.data, event[1].event.data)) {
          return acc;
        }
        const start = event[0].start > event[1].start ? event[0].start : event[1].start;
        const end = event[0].end < event[1].end ? event[0].end : event[1].end;
        return acc + (end - start);
      }, 0);
    },
    zeroDurationEvents() {
      if (this.events === null) {
        return null;
      }
      return this.events.filter(event => event.duration === 0);
    },
  },
  watch: {
    bucket: async function (new_bucket_id) {
      this.events = null;
      this.events = await this.getEvents(new_bucket_id);
    },
  },
  mounted() {
    this.getBuckets();
  },
  methods: {
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
