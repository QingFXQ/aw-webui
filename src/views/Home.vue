<template lang="pug">
div.personal-dashboard
  div.dashboard-header.d-flex.flex-wrap.align-items-end.justify-content-between.mb-4
    div
      p.eyebrow.mb-1 ACTIVITYWATCH · PERSONAL
      h2.mb-1 我的电脑时间
      p.text-muted.mb-0 自动统计真实活跃时间，像 Steam 一样看见长期投入。
    div.d-flex.flex-wrap.mt-3.mt-md-0
      b-dropdown.mr-2.mb-2(
        v-if="availableHosts.length > 1"
        size="sm"
        variant="outline-secondary"
        :text="deviceLabel"
      )
        b-dropdown-item-button(
          :active="selectedHost === ALL_DEVICES"
          @click="selectHost(ALL_DEVICES)"
        ) 全部设备
        b-dropdown-divider
        b-dropdown-item-button(
          v-for="host in availableHosts"
          :key="host"
          :active="selectedHost === host"
          @click="selectHost(host)"
        ) {{ host }}
      b-button.mr-2.mb-2(size="sm" variant="outline-secondary" to="/timeline") 时间线
      b-button.mr-2.mb-2(size="sm" variant="outline-secondary" :to="advancedActivityPath" :disabled="!hostParam") 高级统计
      b-button.mr-2.mb-2(size="sm" variant="outline-secondary" to="/settings") 设置
      b-button.mb-2(size="sm" variant="primary" :disabled="loading" @click="loadDashboard")
        span(v-if="!loading") 刷新
        span(v-else) 加载中…

  div.update-line.text-muted.small.mb-3(v-if="lastUpdated")
    | 最近刷新：{{ lastUpdated }}
    span.ml-2(v-if="deviceLabel") · {{ deviceLabel }}
    span.ml-2(v-if="currentHosts.length > 1") · 多设备重叠时间自动去重

  b-alert(v-if="error" show variant="danger")
    strong 统计加载失败。
    |  {{ error }}

  b-alert(v-else-if="!loading && !hostParam" show variant="warning")
    | 还没有发现可统计的设备。保持 ActivityWatch 后台运行一会儿，再刷新本页。

  div(v-if="loading").py-5.text-center.text-muted
    b-spinner.small-spinner.mr-2
    | 正在整理今天、本周和本月的活动数据…

  template(v-else-if="hostParam")
    div.row.mb-3
      div.col-md-3.mb-3
        div.metric-card.h-100
          div.metric-label 今天
          div.metric-value {{ formatDuration(summary.today) }}
          div.metric-subtitle 有效活跃时间
      div.col-md-3.mb-3
        div.metric-card.h-100
          div.metric-label 本周
          div.metric-value {{ formatDuration(summary.week) }}
          div.metric-subtitle {{ weekLabel }}
      div.col-md-3.mb-3
        div.metric-card.h-100
          div.metric-label 本月
          div.metric-value {{ formatDuration(summary.month) }}
          div.metric-subtitle {{ monthLabel }}
      div.col-md-3.mb-3
        div.metric-card.h-100
          div.metric-label 生涯累计
          div.metric-value(v-if="!lifetimeLoading && !lifetimeError") {{ formatDuration(summary.lifetime) }}
          div.metric-value(v-else-if="lifetimeLoading") …
          div.metric-value(v-else) —
          div.metric-subtitle(v-if="!lifetimeLoading && !lifetimeError") 从最早记录开始
          div.metric-subtitle(v-else-if="lifetimeLoading")
            | 后台计算中
            span(v-if="lifetimeProgress.total > 1")  {{ lifetimeProgress.done }}/{{ lifetimeProgress.total }}
          div.metric-subtitle(v-else) 暂时无法计算

    div.row.mb-3
      div.col-lg-7.mb-3
        div.dashboard-panel.h-100
          div.panel-heading.d-flex.flex-wrap.align-items-center.justify-content-between
            div
              h4.mb-1 软件排行榜
              p.text-muted.small.mb-0 {{ rankingSubtitle }}
            div.d-flex.align-items-center.mt-2.mt-sm-0
              b-button-group.mr-2(size="sm")
                b-button(
                  variant="outline-secondary"
                  :pressed="rankingScope === 'week'"
                  @click="rankingScope = 'week'"
                ) 本周
                b-button(
                  variant="outline-secondary"
                  :pressed="rankingScope === 'lifetime'"
                  @click="rankingScope = 'lifetime'"
                ) 生涯
              span.device-pill {{ deviceLabel }}

          div.empty-state(v-if="rankingScope === 'lifetime' && lifetimeLoading")
            | 正在后台计算生涯软件时长
            span(v-if="lifetimeProgress.total > 1")  · {{ lifetimeProgress.done }}/{{ lifetimeProgress.total }}
            | …
          div.empty-state(v-else-if="rankingScope === 'lifetime' && lifetimeError")
            | 生涯榜暂时不可用：{{ lifetimeError }}
          div.empty-state(v-else-if="displayedApps.length === 0") 暂时没有软件活动数据。
          div.app-row(v-else v-for="(app, index) in displayedApps" :key="app.name")
            div.app-rank {{ index + 1 }}
            div.app-info
              div.d-flex.justify-content-between.align-items-baseline
                strong.app-name {{ app.name }}
                span.app-duration {{ formatDuration(app.duration) }}
              div.usage-track
                div.usage-fill(:style="{ width: appBarWidth(app.duration) + '%' }")

      div.col-lg-5.mb-3
        div.dashboard-panel.h-100
          div.panel-heading
            h4.mb-1 最近 7 天
            p.text-muted.small.mb-0 每天真正坐在电脑前投入了多久

          div.streak-box.mb-3
            span.streak-number {{ streakLabel }}
            span.streak-copy 连续投入（近 7 天）

          div.trend-chart(v-if="dailyTrend.length")
            div.trend-column(v-for="day in dailyTrend" :key="day.date")
              div.trend-value {{ compactDuration(day.duration) }}
              div.trend-bar-wrap
                div.trend-bar(:style="{ height: trendHeight(day.duration) + '%' }")
              div.trend-day {{ day.label }}
          div.empty-state(v-else) 暂时没有趋势数据。

    div.row
      div.col-lg-6.mb-3
        div.dashboard-panel.h-100
          div.panel-heading
            h4.mb-1 投入分类
            p.text-muted.small.mb-0 沿用 ActivityWatch 的分类规则，本周汇总

          div.empty-state(v-if="topCategories.length === 0")
            | 还没有分类数据。可以去设置里给 Blender、UE、VS Code 等软件建立分类。
          div.category-row(v-for="category in topCategories" :key="category.name")
            div.d-flex.justify-content-between
              span {{ category.name }}
              strong {{ formatDuration(category.duration) }}
            div.category-track
              div.category-fill(:style="{ width: categoryBarWidth(category.duration) + '%' }")

      div.col-lg-6.mb-3
        div.dashboard-panel.h-100
          div.panel-heading
            h4.mb-1 快速入口
            p.text-muted.small.mb-0 常用统计放前面，复杂工具继续保留

          div.quick-grid
            router-link.quick-card(:to="advancedActivityPath")
              strong 高级 Activity
              span 原版详细统计、筛选与自定义视图
            router-link.quick-card(to="/timeline")
              strong Timeline
              span 按时间顺序复盘一天
            router-link.quick-card(to="/settings/categorization")
              strong 软件分类
              span 把 UE、Blender、编程工具归到技能方向
            router-link.quick-card(to="/buckets")
              strong 原始数据
              span 查看设备与 watcher 数据源
</template>

<script lang="ts">
import moment from 'moment';

import queries, { MultiQueryParams } from '~/queries';
import {
  applyScreentimeNames,
  screentimeNameMap,
  useActivityStore,
} from '~/stores/activity';
import { useBucketsStore } from '~/stores/buckets';
import { useCategoryStore } from '~/stores/categories';
import { useSettingsStore } from '~/stores/settings';
import {
  ALL_DEVICES,
  buildMultideviceHostParams,
  eligibleMultideviceHosts,
  formatHostParam,
} from '~/util/multidevice';
import { getClient } from '~/util/awclient';
import { get_day_start_with_offset, get_today_with_offset } from '~/util/time';
import { timeperiodToStr } from '~/util/timeperiod';
import { IEvent } from '~/util/interfaces';

interface RankedItem {
  name: string;
  duration: number;
}

interface TrendDay {
  date: string;
  label: string;
  duration: number;
}

interface AggregateResult {
  duration: number;
  app_events: IEvent[];
  cat_events: IEvent[];
  active_events: IEvent[];
  title_events?: IEvent[];
}

const EMPTY_AGGREGATE: AggregateResult = {
  duration: 0,
  app_events: [],
  cat_events: [],
  active_events: [],
  title_events: [],
};

const WEEKDAY_ZH = ['', '一', '二', '三', '四', '五', '六', '日'];

export default {
  name: 'Home',
  data() {
    return {
      loading: true,
      lifetimeLoading: false,
      error: '',
      lifetimeError: '',
      lifetimeProgress: { done: 0, total: 0 },
      selectedHost: '',
      lastUpdated: '',
      availableHosts: [] as string[],
      rankingScope: 'week' as 'week' | 'lifetime',
      summary: {
        today: 0,
        week: 0,
        month: 0,
        lifetime: 0,
      },
      topApps: [] as RankedItem[],
      lifetimeApps: [] as RankedItem[],
      topCategories: [] as RankedItem[],
      dailyTrend: [] as TrendDay[],
      activityStore: useActivityStore(),
      bucketsStore: useBucketsStore(),
      categoryStore: useCategoryStore(),
      settingsStore: useSettingsStore(),
      loadVersion: 0,
    };
  },
  computed: {
    currentHosts(): string[] {
      if (this.selectedHost === ALL_DEVICES) return this.availableHosts;
      return this.selectedHost ? [this.selectedHost] : [];
    },
    hostParam(): string {
      if (this.selectedHost === ALL_DEVICES) return ALL_DEVICES;
      return this.selectedHost ? formatHostParam([this.selectedHost]) : '';
    },
    deviceLabel(): string {
      if (this.currentHosts.length > 1) return `${this.currentHosts.length} 台设备`;
      return this.currentHosts[0] || '';
    },
    advancedActivityPath(): string {
      if (!this.hostParam) return '/activity';
      return `/activity/${this.hostParam}/day`;
    },
    weekLabel(): string {
      const startUnit = this.settingsStore.startOfWeek === 'Sunday' ? 'week' : 'isoWeek';
      return `${moment().startOf(startUnit).format('M月D日')} – 今天`;
    },
    monthLabel(): string {
      return moment().format('YYYY年M月');
    },
    displayedApps(): RankedItem[] {
      return this.rankingScope === 'lifetime' ? this.lifetimeApps : this.topApps;
    },
    rankingSubtitle(): string {
      if (this.rankingScope === 'lifetime') return '从最早记录开始累计的软件时长';
      return '本周真实活跃时长';
    },
    maxAppDuration(): number {
      return Math.max(1, ...this.displayedApps.map((item: RankedItem) => item.duration));
    },
    maxCategoryDuration(): number {
      return Math.max(1, ...this.topCategories.map((item: RankedItem) => item.duration));
    },
    maxTrendDuration(): number {
      return Math.max(1, ...this.dailyTrend.map((item: TrendDay) => item.duration));
    },
    streakLabel(): string {
      if (!this.dailyTrend.length) return '0 天';
      let streak = 0;
      for (let i = this.dailyTrend.length - 1; i >= 0; i -= 1) {
        if (this.dailyTrend[i].duration < 60) break;
        streak += 1;
      }
      return `${streak} 天`;
    },
  },
  async mounted() {
    await this.loadDashboard();
  },
  beforeDestroy() {
    this.loadVersion += 1;
    getClient().abort();
  },
  methods: {
    selectHost(host: string) {
      if (host === this.selectedHost) return;
      this.selectedHost = host;
      this.loadDashboard();
    },

    makePeriod(startDate: string, days: number) {
      return {
        start: get_day_start_with_offset(startDate, this.settingsStore.startOfDay),
        length: [Math.max(1, days), 'days'] as [number, string],
      };
    },

    async queryAggregatePeriod(startDate: string, days: number): Promise<AggregateResult> {
      const timeperiod = this.makePeriod(startDate, days);
      const period = timeperiodToStr(timeperiod);
      const categories = this.categoryStore.classes_for_query;
      const client = getClient();

      const hosts = this.currentHosts;
      if (hosts.length > 1) {
        const { host_params, hosts_with_buckets } = buildMultideviceHostParams(
          hosts,
          this.bucketsStore.bucketsWindow,
          this.bucketsStore.bucketsAFK,
          this.bucketsStore.bucketsAndroid
        );
        const params: MultiQueryParams = {
          hosts: hosts_with_buckets,
          filter_afk: true,
          always_active_pattern: this.settingsStore.always_active_pattern,
          categories,
          filter_categories: [],
          host_params,
          include_audible: false,
          bid_browsers: [],
        };
        const data = await client.query([period], queries.multideviceQuery(params), {
          name: 'personalDashboardMulti',
          verbose: false,
        });
        const result = data && data[0] && data[0].window ? data[0].window : EMPTY_AGGREGATE;

        const screentimeBuckets = Object.values(host_params)
          .filter((p: any) => p.isIos && p.bid_android)
          .map((p: any) => p.bid_android as string);
        if (screentimeBuckets.length > 0 && result.app_events) {
          try {
            const nameData = await client.query(
              [period],
              queries.screentimeNamesQuery(screentimeBuckets),
              { name: 'personalDashboardScreenTimeNames', verbose: false }
            );
            const names: Record<string, string> = {};
            (nameData || []).forEach((events: IEvent[]) =>
              Object.assign(names, screentimeNameMap(events || []))
            );
            applyScreentimeNames(result.app_events || [], names);
            applyScreentimeNames(result.title_events || [], names);
          } catch (e) {
            console.warn('Dashboard ScreenTime name lookup failed:', e);
          }
        }

        return {
          duration: Number(result.duration || 0),
          app_events: result.app_events || [],
          cat_events: result.cat_events || [],
          active_events: result.active_events || [],
          title_events: result.title_events || [],
        };
      }

      const host = hosts[0];
      const windowBuckets = this.bucketsStore.bucketsWindow(host);
      const afkBuckets = this.bucketsStore.bucketsAFK(host);
      const androidBuckets = this.bucketsStore.bucketsAndroid(host);

      if (windowBuckets.length > 0 && afkBuckets.length > 0) {
        const params = {
          bid_window: windowBuckets[0],
          bid_afk: afkBuckets[0],
          // Dashboard only needs app/category totals, so skip browser-domain work.
          bid_browsers: [],
          filter_afk: true,
          include_audible: false,
          categories,
          filter_categories: [],
          always_active_pattern: this.settingsStore.always_active_pattern,
        };
        const data = await client.query([period], queries.fullDesktopQuery(params), {
          name: 'personalDashboardDesktop',
          verbose: false,
        });
        const result = data && data[0] && data[0].window ? data[0].window : EMPTY_AGGREGATE;
        return {
          duration: Number(result.duration || 0),
          app_events: result.app_events || [],
          cat_events: result.cat_events || [],
          active_events: result.active_events || [],
          title_events: result.title_events || [],
        };
      }

      const iosBucket = androidBuckets.find((id: string) => id.startsWith('aw-import-screentime'));
      const selectedBucket = iosBucket || androidBuckets[0];
      if (!selectedBucket) return { ...EMPTY_AGGREGATE };

      const data = await client.query(
        [period],
        queries.appQuery(selectedBucket, categories, [], !!iosBucket),
        { name: 'personalDashboardMobile', verbose: false }
      );
      const result = data && data[0] ? data[0] : EMPTY_AGGREGATE;

      if (iosBucket && result.title_events && result.app_events) {
        const names = screentimeNameMap(result.title_events || []);
        applyScreentimeNames(result.app_events || [], names);
        applyScreentimeNames(result.title_events || [], names);
      }

      return {
        duration: Number(result.duration || 0),
        app_events: result.app_events || [],
        cat_events: result.cat_events || [],
        active_events: result.active_events || [],
        title_events: result.title_events || [],
      };
    },

    snapshotApps(events: IEvent[], limit = 10): RankedItem[] {
      return (events || [])
        .map((event: IEvent) => ({
          name: event.data && event.data.app ? String(event.data.app) : '未知应用',
          duration: Number(event.duration || 0),
        }))
        .filter((item: RankedItem) => item.duration > 0)
        .slice(0, limit);
    },

    snapshotCategories(events: IEvent[]): RankedItem[] {
      return (events || [])
        .map((event: IEvent) => {
          const raw = event.data ? event.data['$category'] : null;
          const name = Array.isArray(raw)
            ? raw.length === 1 && raw[0] === 'Uncategorized'
              ? '未分类'
              : raw.join(' › ')
            : raw
              ? String(raw)
              : '未分类';
          return { name, duration: Number(event.duration || 0) };
        })
        .filter((item: RankedItem) => item.duration > 0)
        .slice(0, 8);
    },

    canSplitActiveEvents(): boolean {
      if (this.currentHosts.length > 1) return true;
      const host = this.currentHosts[0];
      return (
        this.bucketsStore.bucketsWindow(host).length > 0 &&
        this.bucketsStore.bucketsAFK(host).length > 0
      );
    },

    durationForDay(events: IEvent[], date: string): number {
      const start = moment(get_day_start_with_offset(date, this.settingsStore.startOfDay));
      const end = start.clone().add(1, 'day');
      const startMs = start.valueOf();
      const endMs = end.valueOf();

      return (events || []).reduce((total: number, event: IEvent) => {
        const eventStart = moment(event.timestamp).valueOf();
        const eventEnd = eventStart + Number(event.duration || 0) * 1000;
        const overlap = Math.max(0, Math.min(endMs, eventEnd) - Math.max(startMs, eventStart));
        return total + overlap / 1000;
      }, 0);
    },

    trendFromActiveEvents(events: IEvent[], today: string): TrendDay[] {
      const trend: TrendDay[] = [];
      for (let offset = 6; offset >= 0; offset -= 1) {
        const date = moment(today).subtract(offset, 'days').format('YYYY-MM-DD');
        const weekday = moment(date).isoWeekday();
        trend.push({
          date,
          label: `周${WEEKDAY_ZH[weekday]}`,
          duration: this.durationForDay(events, date),
        });
      }
      return trend;
    },

    async loadTrend(today: string): Promise<TrendDay[]> {
      const start = moment(today).subtract(6, 'days').format('YYYY-MM-DD');

      if (this.canSplitActiveEvents()) {
        const result = await this.queryAggregatePeriod(start, 7);
        return this.trendFromActiveEvents(result.active_events, today);
      }

      const trend: TrendDay[] = [];
      for (let offset = 6; offset >= 0; offset -= 1) {
        const date = moment(today).subtract(offset, 'days').format('YYYY-MM-DD');
        const result = await this.queryAggregatePeriod(date, 1);
        const weekday = moment(date).isoWeekday();
        trend.push({
          date,
          label: `周${WEEKDAY_ZH[weekday]}`,
          duration: result.duration,
        });
      }
      return trend;
    },

    async loadDashboard(): Promise<void> {
      const version = ++this.loadVersion;
      this.loading = true;
      this.error = '';
      this.lifetimeError = '';
      this.rankingScope = 'week';

      try {
        await this.settingsStore.ensureLoaded();
        await this.bucketsStore.ensureLoaded();
        this.categoryStore.load();

        const eligibleHosts = eligibleMultideviceHosts(
          this.bucketsStore.hosts,
          this.bucketsStore.bucketsWindow,
          this.bucketsStore.bucketsAFK,
          this.bucketsStore.bucketsAndroid
        );

        if (version !== this.loadVersion) return;

        this.availableHosts = eligibleHosts;
        if (eligibleHosts.length === 0) {
          this.selectedHost = '';
          return;
        }

        const selectedIsValid =
          this.selectedHost === ALL_DEVICES
            ? eligibleHosts.length > 1
            : eligibleHosts.includes(this.selectedHost);
        if (!selectedIsValid) {
          this.selectedHost = eligibleHosts.length > 1 ? ALL_DEVICES : eligibleHosts[0];
        }

        const today = get_today_with_offset(this.settingsStore.startOfDay);

        const todayResult = await this.queryAggregatePeriod(today, 1);
        if (version !== this.loadVersion) return;
        this.summary.today = todayResult.duration;

        const startUnit = this.settingsStore.startOfWeek === 'Sunday' ? 'week' : 'isoWeek';
        const weekStart = moment(today).startOf(startUnit).format('YYYY-MM-DD');
        const weekDays = moment(today).diff(moment(weekStart), 'days') + 1;
        const weekResult = await this.queryAggregatePeriod(weekStart, weekDays);
        if (version !== this.loadVersion) return;
        this.summary.week = weekResult.duration;
        this.topApps = this.snapshotApps(weekResult.app_events);
        this.topCategories = this.snapshotCategories(weekResult.cat_events);

        const monthStart = moment(today).startOf('month').format('YYYY-MM-DD');
        const monthDays = moment(today).diff(moment(monthStart), 'days') + 1;
        const monthResult = await this.queryAggregatePeriod(monthStart, monthDays);
        if (version !== this.loadVersion) return;
        this.summary.month = monthResult.duration;

        this.dailyTrend = await this.loadTrend(today);
        if (version !== this.loadVersion) return;

        this.lastUpdated = moment().format('HH:mm');
      } catch (e) {
        const message = e instanceof Error ? e.message : String(e);
        if (message !== 'canceled') {
          console.error(e);
          this.error = message;
        }
      } finally {
        if (version === this.loadVersion) {
          this.loading = false;
        }
      }

      if (version === this.loadVersion && this.hostParam) {
        this.loadLifetime(version);
      }
    },

    async loadLifetime(version: number): Promise<void> {
      this.lifetimeLoading = true;
      this.lifetimeError = '';
      this.lifetimeProgress = { done: 0, total: 0 };

      try {
        const today = get_today_with_offset(this.settingsStore.startOfDay);
        const result = await this.activityStore.get_earliest_date(this.hostParam);
        if (version !== this.loadVersion) return;

        const earliest = result && result.date ? result.date : today;
        const endExclusive = moment(today).add(1, 'day');
        let cursor = moment(earliest);
        const chunks: { start: string; days: number }[] = [];

        // Keep lifetime requests bounded. 92 days matches ActivityWatch's own
        // long-range day-resolution threshold and stays comfortably below the
        // server's request timeout on typical local databases.
        while (cursor.isBefore(endExclusive)) {
          const next = moment.min(cursor.clone().add(92, 'days'), endExclusive.clone());
          const days = Math.max(1, next.diff(cursor, 'days'));
          chunks.push({ start: cursor.format('YYYY-MM-DD'), days });
          cursor = next;
        }

        this.lifetimeProgress = { done: 0, total: chunks.length };

        let totalDuration = 0;
        const appTotals = new Map<string, number>();

        for (const chunk of chunks) {
          if (version !== this.loadVersion) return;

          const aggregate = await this.queryAggregatePeriod(chunk.start, chunk.days);
          totalDuration += aggregate.duration;

          for (const app of this.snapshotApps(aggregate.app_events, 100)) {
            appTotals.set(app.name, (appTotals.get(app.name) || 0) + app.duration);
          }

          this.lifetimeProgress = {
            done: this.lifetimeProgress.done + 1,
            total: chunks.length,
          };
        }

        if (version !== this.loadVersion) return;

        this.summary.lifetime = totalDuration;
        this.lifetimeApps = Array.from(appTotals.entries())
          .map(([name, duration]) => ({ name, duration }))
          .sort((a, b) => b.duration - a.duration)
          .slice(0, 10);
      } catch (e) {
        const message = e instanceof Error ? e.message : String(e);
        if (message !== 'canceled') {
          console.warn('Lifetime total unavailable:', e);
          this.lifetimeError = message;
        }
      } finally {
        if (version === this.loadVersion) {
          this.lifetimeLoading = false;
        }
      }
    },

    formatDuration(seconds: number): string {
      if (!seconds || seconds < 60) return '< 1 分钟';
      const totalMinutes = Math.round(seconds / 60);
      const hours = Math.floor(totalMinutes / 60);
      const minutes = totalMinutes % 60;
      if (hours === 0) return `${minutes} 分钟`;
      if (minutes === 0) return `${hours} 小时`;
      return `${hours} 小时 ${minutes} 分`;
    },

    compactDuration(seconds: number): string {
      if (!seconds || seconds < 60) return '0m';
      const minutes = Math.round(seconds / 60);
      if (minutes < 60) return `${minutes}m`;
      const hours = minutes / 60;
      return `${hours >= 10 ? Math.round(hours) : hours.toFixed(1)}h`;
    },

    appBarWidth(duration: number): number {
      return Math.max(2, Math.round((duration / this.maxAppDuration) * 100));
    },

    categoryBarWidth(duration: number): number {
      return Math.max(2, Math.round((duration / this.maxCategoryDuration) * 100));
    },

    trendHeight(duration: number): number {
      if (!duration) return 2;
      return Math.max(7, Math.round((duration / this.maxTrendDuration) * 100));
    },
  },
};
</script>

<style lang="scss" scoped>
.personal-dashboard {
  max-width: 1180px;
  margin: 0 auto;
}

.eyebrow {
  font-size: 0.72rem;
  letter-spacing: 0.14em;
  font-weight: 700;
  opacity: 0.58;
}

.update-line {
  min-height: 1.25rem;
}

.metric-card,
.dashboard-panel {
  border: 1px solid rgba(127, 127, 127, 0.2);
  border-radius: 14px;
  background: rgba(127, 127, 127, 0.045);
}

.metric-card {
  padding: 1.15rem;
}

.metric-label,
.metric-subtitle {
  color: #7c8188;
}

.metric-label {
  font-size: 0.82rem;
  font-weight: 600;
}

.metric-value {
  margin: 0.25rem 0 0.15rem;
  font-size: clamp(1.55rem, 3vw, 2.15rem);
  line-height: 1.05;
  font-weight: 700;
  letter-spacing: -0.035em;
}

.metric-subtitle {
  font-size: 0.78rem;
}

.dashboard-panel {
  padding: 1.2rem;
}

.panel-heading {
  margin-bottom: 1rem;
}

.device-pill {
  padding: 0.28rem 0.58rem;
  border-radius: 999px;
  background: rgba(127, 127, 127, 0.12);
  color: #747980;
  font-size: 0.76rem;
  white-space: nowrap;
}

.app-row {
  display: grid;
  grid-template-columns: 1.8rem minmax(0, 1fr);
  gap: 0.55rem;
  align-items: center;
  padding: 0.48rem 0;
}

.app-rank {
  color: #8a8f95;
  font-size: 0.78rem;
  text-align: center;
}

.app-name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.app-duration {
  margin-left: 0.75rem;
  color: #777d84;
  font-size: 0.8rem;
  white-space: nowrap;
}

.usage-track,
.category-track {
  height: 5px;
  margin-top: 0.35rem;
  overflow: hidden;
  border-radius: 999px;
  background: rgba(127, 127, 127, 0.12);
}

.usage-fill,
.category-fill {
  height: 100%;
  border-radius: inherit;
  background: currentColor;
  color: #6b8afd;
  opacity: 0.88;
}

.category-fill {
  color: #65a985;
}

.category-row {
  padding: 0.48rem 0;
  font-size: 0.9rem;
}

.streak-box {
  display: flex;
  align-items: baseline;
  gap: 0.45rem;
}

.streak-number {
  font-size: 1.9rem;
  font-weight: 700;
  letter-spacing: -0.03em;
}

.streak-copy {
  color: #7b8086;
  font-size: 0.84rem;
}

.trend-chart {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 0.45rem;
  height: 190px;
}

.trend-column {
  min-width: 0;
  display: grid;
  grid-template-rows: 1.25rem 1fr 1.25rem;
  align-items: end;
  text-align: center;
}

.trend-value,
.trend-day {
  font-size: 0.7rem;
  color: #7d8288;
}

.trend-bar-wrap {
  height: 132px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  border-bottom: 1px solid rgba(127, 127, 127, 0.18);
}

.trend-bar {
  width: min(28px, 70%);
  min-height: 2px;
  border-radius: 5px 5px 2px 2px;
  background: #6b8afd;
  opacity: 0.82;
  transition: height 180ms ease;
}

.quick-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.7rem;
}

.quick-card {
  display: flex;
  flex-direction: column;
  min-height: 96px;
  padding: 0.85rem;
  border: 1px solid rgba(127, 127, 127, 0.18);
  border-radius: 11px;
  color: inherit;
  text-decoration: none;
  background: rgba(127, 127, 127, 0.035);
}

.quick-card:hover {
  text-decoration: none;
  border-color: rgba(107, 138, 253, 0.55);
  transform: translateY(-1px);
}

.quick-card span {
  margin-top: 0.35rem;
  color: #7c8188;
  font-size: 0.78rem;
  line-height: 1.35;
}

.empty-state {
  padding: 1rem 0;
  color: #858a90;
  font-size: 0.88rem;
}

.small-spinner {
  width: 1.15rem;
  height: 1.15rem;
}

@media (max-width: 575.98px) {
  .dashboard-header h2 {
    font-size: 1.6rem;
  }

  .quick-grid {
    grid-template-columns: 1fr;
  }

  .trend-chart {
    gap: 0.2rem;
  }

  .device-pill {
    display: none;
  }
}
</style>