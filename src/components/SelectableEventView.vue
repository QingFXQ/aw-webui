<template lang="pug">
div
  div.form-group
    select.form-control(v-model="vis_method")
      option(value="eventlist") 事件列表
      option(value="timeline") 时间线
      option(value="summary") 汇总
      option(value="raw") 原始 JSON

  div(v-if="vis_method == 'timeline'")
    vis-timeline(:buckets="[{'id': 'search', 'type': 'search', 'events': events}]")
  div(v-if="vis_method == 'eventlist'")
    aw-eventlist(:events="events")
  div(v-if="vis_method == 'summary'")
    input.form-control(type="text" v-model.lazy.trim="summaryKey" placeholder="数据字段" style="margin-bottom: 1em;")
    aw-summary(:fields="events", :colorfunc="colorfunc", :namefunc="namefunc")
  div(v-if="vis_method == 'raw'")
    pre {{ events }}
</template>

<script lang="ts">
export default {
  name: 'aw-selectable-eventview',
  props: {
    events: Array,
    event_type: { type: String, default: 'currentwindow' },
  },
  data: function () {
    return {
      vis_method: 'eventlist',

      /* Summary props */
      summaryKey: 'title',
      colorfunc: null,
      namefunc: null,
    };
  },
  mounted: async function () {
    this.colorfunc = this.summaryKeyFunc;
    this.namefunc = this.summaryKeyFunc;
  },
  methods: {
    summaryKeyFunc: function (e) {
      return e.data[this.summaryKey];
    },
  },
};
</script>
