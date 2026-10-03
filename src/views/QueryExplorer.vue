<template lang="pug">

div
  h3 查询浏览器

  | 如需了解如何编写查询，请查看 #[a(href="https://docs.activitywatch.net/en/latest/examples/querying-data.html") 文档]。

  hr

  div.alert.alert-danger(v-if="error")
    | {{error}}

  div.alert.alert-danger(v-if="saved_query_error")
    | {{saved_query_error}}

  form
    div.form-row.align-items-end
      div.form-group.col-lg-6
        label.mb-1(for="saved-query-select") 已保存的查询
        select#saved-query-select.form-control(v-model="selected_saved_query_id", @change="loadSelectedQuery()")
          option(value="") 选择已保存的查询…
          option(v-for="savedQuery in savedQueries", :key="savedQuery.id", :value="savedQuery.id")
            | {{savedQuery.name}}
      div.form-group.col-lg-6
        div.saved-query-actions
          button.btn.btn-success.mr-2(type="button", @click="saveCurrentQuery()") 保存当前查询
          button.btn.btn-secondary.mr-2(type="button", @click="renameSelectedQuery()", :disabled="!selected_saved_query_id") 重命名
          button.btn.btn-danger(type="button", @click="deleteSelectedQuery()", :disabled="!selected_saved_query_id")
            icon(name="trash")
            |  删除

    div.form-row
      div.form-group.col-md-6
        | 开始
        input.form-control(type="date", :max="today", v-model="startdate")
      div.form-group.col-md-6
        | 结束
        input.form-control(type="date", :max="tomorrow", v-model="enddate")

    div.form-group
      textarea.form-control(v-model="query_code", @keypress.ctrl.enter="query()" style="font-family: monospace", rows=10)
    div.form-inline
      div.form-group
        button.btn.btn-success(type="button", @click="query()") 执行查询
      span(style="padding-left: 1em;")
      | {{eventcount_str}}

  hr

  aw-selectable-eventview(:events="events", :event_type="event_type")

  b-modal(
    v-model="showSaveQueryModal"
    title="保存查询"
    ok-title="保存"
    cancel-title="取消"
    @ok="onSaveQueryConfirm"
    @shown="$refs.saveQueryNameInput && $refs.saveQueryNameInput.focus()"
  )
    b-form-group(label="查询名称：")
      b-form-input(
        ref="saveQueryNameInput"
        v-model="saveQueryName"
        placeholder="查询名称"
      )

  b-modal(
    v-model="showRenameQueryModal"
    title="重命名查询"
    ok-title="重命名"
    cancel-title="取消"
    @ok="onRenameQueryConfirm"
    @shown="$refs.renameQueryNameInput && $refs.renameQueryNameInput.focus()"
  )
    b-form-group(label="新的查询名称：")
      b-form-input(
        ref="renameQueryNameInput"
        v-model="renameQueryName"
        placeholder="查询名称"
      )
</template>

<style scoped lang="scss">
.saved-query-actions {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
}
</style>

<script lang="ts">
import 'vue-awesome/icons/trash';
import moment from 'moment';
import _ from 'lodash';
import { querystr_to_array } from '~/queries';
import { useCategoryStore } from '~/stores/categories';
import { useSettingsStore } from '~/stores/settings';
import {
  buildSavedQuery,
  buildSavedQueryId,
  getDefaultSavedQueryName,
  resolveSavedQueryDates,
  SavedQuery,
  sortSavedQueries,
} from '~/util/savedQueries';

const today = moment().startOf('day');
const tomorrow = moment(today).add(24, 'hours');

export default {
  name: 'QueryExplorer',
  data() {
    const settingsStore = useSettingsStore();

    // allow queries to be saved in a URL parameter
    let queryCode = this.$route.query.q;

    if (_.isEmpty(this.$route.query.q)) {
      queryCode = `
afk_events = query_bucket(find_bucket("aw-watcher-afk_"));
window_events = query_bucket(find_bucket("aw-watcher-window_"));
window_events = filter_period_intersect(window_events, filter_keyvals(afk_events, "status", ["not-afk"]));
merged_events = merge_events_by_keys(window_events, ["app", "title"]);
merged_events = categorize(merged_events, __CATEGORIES__);
RETURN = sort_by_duration(merged_events);
`.trim();
    }

    return {
      settingsStore,
      query_code: queryCode,
      event_type: 'currentwindow',
      events: [],
      today: today.format(),
      tomorrow: tomorrow.format(),
      error: '',
      saved_query_error: '',
      selected_saved_query_id: '',
      startdate: today.format('YYYY-MM-DD'),
      enddate: tomorrow.format('YYYY-MM-DD'),
      showSaveQueryModal: false,
      saveQueryName: '',
      showRenameQueryModal: false,
      renameQueryName: '',
    };
  },
  computed: {
    savedQueries: function (): SavedQuery[] {
      return sortSavedQueries(this.settingsStore.saved_queries || []);
    },
    selectedSavedQuery: function (): SavedQuery | null {
      return this.savedQueries.find(query => query.id === this.selected_saved_query_id) || null;
    },
    eventcount_str: function () {
      if (Array.isArray(this.events)) return '事件数量：' + this.events.length;
      else return '';
    },
  },
  mounted: async function () {
    await this.settingsStore.ensureLoaded();
    useCategoryStore().load();
  },
  methods: {
    persistSavedQueries: async function (savedQueries: SavedQuery[]) {
      try {
        this.saved_query_error = '';
        await this.settingsStore.update({ saved_queries: sortSavedQueries(savedQueries) });
        return true;
      } catch (e) {
        console.error('Failed to save query presets', e);
        this.saved_query_error = '保存查询预设失败。';
        return false;
      }
    },
    loadSelectedQuery: function () {
      if (!this.selectedSavedQuery) {
        return;
      }

      const { startdate, enddate } = resolveSavedQueryDates(this.selectedSavedQuery);
      this.query_code = this.selectedSavedQuery.query_code;
      this.event_type = this.selectedSavedQuery.event_type || 'currentwindow';
      this.startdate = startdate;
      this.enddate = enddate;
      this.saved_query_error = '';
    },
    saveCurrentQuery: async function () {
      const current = this.selectedSavedQuery;

      if (current) {
        if (!confirm(`更新已保存的查询“${current.name}”？`)) {
          return;
        }

        const updatedQuery = buildSavedQuery({
          id: current.id,
          name: current.name,
          query_code: this.query_code,
          startdate: this.startdate,
          enddate: this.enddate,
          event_type: this.event_type,
        });

        const didPersist = await this.persistSavedQueries(
          this.savedQueries.map(savedQuery =>
            savedQuery.id === current.id ? updatedQuery : savedQuery
          )
        );
        if (didPersist) {
          this.selected_saved_query_id = current.id;
        }
        return;
      }

      // No existing query — open modal to get a name
      this.saveQueryName = getDefaultSavedQueryName(this.query_code);
      this.showSaveQueryModal = true;
    },
    onSaveQueryConfirm: async function (event) {
      event.preventDefault();

      const trimmedName = this.saveQueryName.trim();
      if (_.isEmpty(trimmedName)) {
        this.saved_query_error = '查询名称不能为空。';
        return;
      }

      const newId = buildSavedQueryId(
        trimmedName,
        this.savedQueries.map(savedQuery => savedQuery.id)
      );
      const newQuery = buildSavedQuery({
        id: newId,
        name: trimmedName,
        query_code: this.query_code,
        startdate: this.startdate,
        enddate: this.enddate,
        event_type: this.event_type,
      });

      const didPersist = await this.persistSavedQueries([...this.savedQueries, newQuery]);
      if (didPersist) {
        this.selected_saved_query_id = newId;
        this.showSaveQueryModal = false;
      }
    },
    renameSelectedQuery: async function () {
      if (!this.selectedSavedQuery) {
        return;
      }

      this.renameQueryName = this.selectedSavedQuery.name;
      this.showRenameQueryModal = true;
    },
    onRenameQueryConfirm: async function (event) {
      event.preventDefault();

      if (!this.selectedSavedQuery) {
        return;
      }

      const trimmedName = this.renameQueryName.trim();
      if (_.isEmpty(trimmedName)) {
        this.saved_query_error = '查询名称不能为空。';
        return;
      }

      const selectedQueryId = this.selectedSavedQuery.id;
      const didPersist = await this.persistSavedQueries(
        this.savedQueries.map(savedQuery =>
          savedQuery.id === selectedQueryId ? { ...savedQuery, name: trimmedName } : savedQuery
        )
      );
      if (didPersist) {
        this.showRenameQueryModal = false;
      }
    },
    deleteSelectedQuery: async function () {
      if (!this.selectedSavedQuery) {
        return;
      }

      if (!confirm(`删除已保存的查询“${this.selectedSavedQuery.name}”？此操作无法撤销。`)) {
        return;
      }

      const didPersist = await this.persistSavedQueries(
        this.savedQueries.filter(savedQuery => savedQuery.id !== this.selected_saved_query_id)
      );
      if (didPersist) {
        this.selected_saved_query_id = '';
      }
    },
    query: async function () {
      let query = this.query_code;

      // replace magic string `__CATEGORIES__` in query text with latest category rule
      if (_.includes(query, '__CATEGORIES__')) {
        const categoryRules = useCategoryStore().classes_for_query;

        if (useCategoryStore().classes_for_query.length === 0) {
          this.error = '查询中使用了 __CATEGORIES__，但当前还没有定义任何分类。';
          return;
        }

        query = query.replace('__CATEGORIES__', JSON.stringify(categoryRules));
      }

      // the aw-client expects an array of commands with whitespace cleaned up
      query = querystr_to_array(query as string);
      const timeperiods = [moment(this.startdate).format() + '/' + moment(this.enddate).format()];

      try {
        const data = await this.$aw.query(timeperiods, query);
        this.events = data[0];
        this.error = '';
      } catch (e) {
        this.error = e.response.data.message;
      }
    },
  },
};
</script>
