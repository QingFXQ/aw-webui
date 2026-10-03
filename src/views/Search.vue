<template lang="pug">
div
  h3 搜索

  b-alert(v-if="error" show variant="danger")
    | {{error}}

  b-input-group(size="lg")
    b-input(v-model="pattern" v-on:keyup.enter="search()" placeholder="输入要搜索的正则表达式")
    b-input-group-append
      b-button(type="button", @click="search()" variant="success")
        icon.mr-1(name="search")
        | 搜索

  div.d-flex.mt-1
    span.mr-auto.small.text-muted 主机：{{queryOptions.hostname}}
    b-button.border-0(size="sm", variant="outline-dark" @click="show_options = !show_options")
      span(v-if="!show_options")
        | #[icon(name="angle-double-down")] 显示选项
      span(v-else)
        | #[icon(name="angle-double-up")] 收起选项

  div(v-show="show_options")
    h4 选项
    aw-query-options(v-model="queryOptions")

  div(v-if="status == 'searching'")
    div #[icon(name="spinner" pulse)] 搜索中…

  div(v-if="events != null")
    hr

    aw-selectable-eventview(:events="events")

    div
      | 没找到想要的内容？
      br
      | 将搜索范围向前扩展一周：#[b-button(size="sm" variant="outline-dark" @click="extendByWeek()") +1 周]
</template>

<script lang="ts">
import _ from 'lodash';
import moment from 'moment';
import { canonicalEvents, querystr_to_array } from '~/queries';

import 'vue-awesome/icons/search';
import 'vue-awesome/icons/spinner';
import 'vue-awesome/icons/angle-double-down';
import 'vue-awesome/icons/angle-double-up';

export default {
  name: 'Search',
  data() {
    return {
      pattern: '',
      events: null,

      status: null,
      error: '',

      // Options
      show_options: false,
      queryOptions: {
        start: moment().subtract(1, 'day').format('YYYY-MM-DD'),
        stop: moment().add(1, 'day').format('YYYY-MM-DD'),
      },
    };
  },
  methods: {
    search: async function () {
      let query = canonicalEvents({
        bid_window: 'aw-watcher-window_' + this.queryOptions.hostname,
        bid_afk: 'aw-watcher-afk_' + this.queryOptions.hostname,
        filter_afk: this.queryOptions.filter_afk,
        categories: [[['searched'], { type: 'regex', regex: this.pattern, ignore_case: true }]],
        filter_categories: [['searched']],
      });
      query += '; RETURN = events;';

      const query_array = querystr_to_array(query);
      const timeperiods = [
        moment(this.queryOptions.start).format() + '/' + moment(this.queryOptions.stop).format(),
      ];
      try {
        this.status = 'searching';
        const data = await this.$aw.query(timeperiods, query_array);
        this.events = _.orderBy(data[0], ['timestamp'], ['desc']);
        this.error = '';
      } catch (e) {
        console.error(e);
        this.error = e.response.data.message;
      } finally {
        this.status = null;
      }
    },
    extendByWeek() {
      this.queryOptions.start = moment(this.queryOptions.start)
        .subtract(1, 'week')
        .format('YYYY-MM-DD');
      this.search();
    },
  },
};
</script>
