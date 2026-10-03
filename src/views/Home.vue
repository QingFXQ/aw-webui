<template lang="pug">
div.personal-dashboard
  div.dashboard-header.d-flex.flex-wrap.align-items-end.justify-content-between.mb-4
    div
      p.eyebrow.mb-1 ACTIVITYWATCH
      h2.mb-1 使用概览
      p.text-muted.mb-0 今天做了多久、最常用什么、长期投入到哪一档，一眼看清。
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
      b-button.mr-2.mb-2(size="sm" variant="outline-secondary" :to="advancedActivityPath" :disabled="!hostParam") 详细统计
      b-button.mr-2.mb-2(size="sm" variant="outline-secondary" to="/settings") 设置
      b-button.mb-2(size="sm" variant="primary" :disabled="loading" @click="loadDashboard")
        span(v-if="!loading") 刷新
        span(v-else) 加载中…

  div.update-line.text-muted.small.mb-2(v-if="lastUpdated")
    | 最近刷新 {{ lastUpdated }}
    span.ml-2(v-if="deviceLabel") · {{ deviceLabel }}
    span.ml-2(v-if="currentHosts.length > 1") · 多设备重叠时间自动去重
  div.text-muted.small.mb-3(v-if="hostParam && !loading")
    | 统计口径：前台软件活跃时间，排除 AFK；后台纯音频不计入。

  b-alert(v-if="error" show variant="danger")
    strong 统计加载失败。
    |  {{ error }}

  b-alert(v-else-if="!loading && !hostParam" show variant="warning")
    | 还没有发现可统计的设备。保持 ActivityWatch 后台运行一会儿，再刷新本页。

  div(v-if="loading").py-5.text-center.text-muted
    b-spinner.small-spinner.mr-2
    | 正在整理今天、本周和本月的活动数据…

  template(v-else-if="hostParam")
    div.overview-grid.mb-3
      div.hero-card
        div.d-flex.align-items-start.justify-content-between
          div
            div.hero-kicker 今天
            div.hero-value {{ formatDuration(summary.today) }}
            div.hero-subtitle 有效活跃时间
          span.device-pill(v-if="deviceLabel") {{ deviceLabel }}

        div.hero-divider
        div.hero-stats
          div.hero-stat
            span 本周
            strong {{ formatDuration(summary.week) }}
          div.hero-stat
            span 本月
            strong {{ formatDuration(summary.month) }}
          div.hero-stat
            span 连续投入
            strong {{ streakLabel }}

      div.side-summary
        div.summary-card
          div.summary-label 生涯累计
          div.summary-value(v-if="!lifetimeLoading && !lifetimeError") {{ formatDuration(summary.lifetime) }}
          div.summary-value(v-else-if="lifetimeLoading") 计算中…
          div.summary-value(v-else) —
          div.summary-note(v-if="!lifetimeLoading && !lifetimeError") 从最早记录开始
          div.summary-note(v-else-if="lifetimeLoading")
            | 后台整理历史
            span(v-if="lifetimeProgress.total > 1")  · {{ lifetimeProgress.done }}/{{ lifetimeProgress.total }}
          div.summary-note(v-else) 暂时不可用

        div.summary-card
          div.summary-label 当前周期
          div.summary-period {{ weekLabel }}
          div.summary-note {{ monthLabel }}

    div.content-grid.mb-3
      div.dashboard-panel.apps-panel
        div.panel-heading.d-flex.flex-wrap.align-items-start.justify-content-between
          div
            div.section-kicker 软件使用
            h4.mb-1 排行
            p.text-muted.small.mb-0 {{ rankingSubtitle }}
            p.text-muted.small.mb-0 条长代表当前周期使用量，颜色代表累计使用里程碑。
          div.d-flex.align-items-center.mt-2.mt-sm-0
            b-button-group(size="sm")
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

        div.empty-state(v-if="rankingScope === 'lifetime' && lifetimeLoading")
          | 正在后台计算生涯软件时长
          span(v-if="lifetimeProgress.total > 1")  · {{ lifetimeProgress.done }}/{{ lifetimeProgress.total }}
          | …
        div.empty-state(v-else-if="rankingScope === 'lifetime' && lifetimeError")
          | 生涯榜暂时不可用：{{ lifetimeError }}
        div.empty-state(v-else-if="displayedApps.length === 0") 暂时没有软件活动数据。

        div.app-list(v-else)
          div.app-row(v-for="(app, index) in displayedApps" :key="app.name")
            div.app-rank {{ String(index + 1).padStart(2, '0') }}
            div.app-icon-shell(:style="{ borderColor: usageTierColor(app.name, app.duration) }")
              span.app-icon-fallback {{ appInitial(app.name) }}
              img.app-icon(
                v-if="appIconUrl(app.name)"
                :src="appIconUrl(app.name)"
                :alt="app.name"
                referrerpolicy="no-referrer"
                @error="onIconError"
              )
            div.app-info
              div.app-line
                strong.app-name {{ app.name }}
                span.app-duration {{ formatDuration(app.duration) }}
              div.app-meta(v-if="rankingScope === 'week'")
                | 累计 {{ compactDuration(appLifetimeDuration(app.name, app.duration)) }}
                span.ml-2 · {{ usageTierLabel(app.name, app.duration) }}
              div.app-meta(v-else) {{ usageTierLabel(app.name, app.duration) }}
              div.usage-track
                div.usage-fill(
                  :style="{ width: appBarWidth(app.duration) + '%', backgroundColor: usageTierColor(app.name, app.duration) }"
                )

        div.tier-legend.mt-3
          div.tier-legend-title 累计时长颜色
          div.tier-items
            div.tier-item(v-for="tier in usageLegend" :key="tier.label")
              span.tier-dot(:style="{ backgroundColor: tier.color, borderColor: tier.border || tier.color }")
              span {{ tier.label }}

      div.dashboard-panel.trend-panel
        div.panel-heading
          div.section-kicker 节奏
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

    div.lower-grid
      div.dashboard-panel
        div.panel-heading
          div.section-kicker 分类
          h4.mb-1 本周投入
          p.text-muted.small.mb-0 沿用 ActivityWatch 分类规则

        div.empty-state(v-if="topCategories.length === 0")
          | 还没有分类数据。可以去设置里给 Blender、UE、VS Code 等软件建立分类。
        div.category-row(v-for="category in topCategories" :key="category.name")
          div.d-flex.justify-content-between
            span {{ category.name }}
            strong {{ formatDuration(category.duration) }}
          div.category-track
            div.category-fill(:style="{ width: categoryBarWidth(category.duration) + '%' }")

      div.dashboard-panel
        div.panel-heading
          div.section-kicker 入口
          h4.mb-1 更多
          p.text-muted.small.mb-0 复杂功能继续保留，但不挤占首页

        div.quick-grid
          router-link.quick-card(:to="advancedActivityPath")
            strong 详细统计
            span 原版 Activity 视图、筛选与自定义
          router-link.quick-card(to="/timeline")
            strong 时间线
            span 按时间顺序复盘一天
          router-link.quick-card(to="/settings/categorization")
            strong 软件分类
            span 给常用软件建立分类规则
          router-link.quick-card(to="/buckets")
            strong 原始数据
            span 查看设备与 watcher 数据源
</template>

<script lang="ts">
import moment from 'moment';

import queries, { MultiQueryParams } from '~/queries';
import { applyScreentimeNames, screentimeNameMap, useActivityStore } from '~/stores/activity';
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

interface UsageTier {
  hours: number;
  color: string;
  label: string;
}

const EMPTY_AGGREGATE: AggregateResult = {
  duration: 0,
  app_events: [],
  cat_events: [],
  active_events: [],
  title_events: [],
};

const WEEKDAY_ZH = ['', '一', '二', '三', '四', '五', '六', '日'];

const USAGE_TIERS: UsageTier[] = [
  { hours: 1000, color: '#111111', label: '黑 · 1000h' },
  { hours: 800, color: '#facc15', label: '黄 · 800h' },
  { hours: 500, color: '#ef4444', label: '红 · 500h' },
  { hours: 200, color: '#f97316', label: '橙 · 200h' },
  { hours: 100, color: '#8b5cf6', label: '紫 · 100h' },
  { hours: 50, color: '#3b82f6', label: '蓝 · 50h' },
  { hours: 20, color: '#22c55e', label: '绿 · 20h' },
  { hours: 10, color: '#f8fafc', label: '白 · 10h' },
  { hours: 0, color: '#94a3b8', label: '灰 · <10h' },
];

const APP_ICON_MATCHES: { needles: string[]; slug: string }[] = [
  { needles: ['blender'], slug: 'blender' },
  { needles: ['unrealeditor', 'unreal engine', 'ue5'], slug: 'unrealengine' },
  { needles: ['visual studio code', 'code.exe', 'code - insiders'], slug: 'visualstudiocode' },
  { needles: ['devenv', 'visual studio'], slug: 'visualstudio' },
  { needles: ['chrome'], slug: 'googlechrome' },
  { needles: ['edge'], slug: 'microsoftedge' },
  { needles: ['firefox'], slug: 'firefoxbrowser' },
  { needles: ['safari'], slug: 'safari' },
  { needles: ['photoshop'], slug: 'adobephotoshop' },
  { needles: ['illustrator'], slug: 'adobeillustrator' },
  { needles: ['premiere'], slug: 'adobepremierepro' },
  { needles: ['after effects'], slug: 'adobeaftereffects' },
  { needles: ['figma'], slug: 'figma' },
  { needles: ['obsidian'], slug: 'obsidian' },
  { needles: ['notion'], slug: 'notion' },
  { needles: ['discord'], slug: 'discord' },
  { needles: ['spotify'], slug: 'spotify' },
  { needles: ['steam'], slug: 'steam' },
  { needles: ['docker'], slug: 'docker' },
  { needles: ['github'], slug: 'github' },
  { needles: ['unity'], slug: 'unity' },
  { needles: ['davinci'], slug: 'davinciresolve' },
  { needles: ['revit', 'autocad', 'autodesk'], slug: 'autodesk' },
  { needles: ['python'], slug: 'python' },
  { needles: ['powershell'], slug: 'powershell' },
  { needles: ['ollama'], slug: 'ollama' },
];

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
      lifetimeAppTotals: {} as Record<string, number>,
      topCategories: [] as RankedItem[],
      dailyTrend: [] as TrendDay[],
      usageLegend: [
        { label: '10h', color: '#f8fafc', border: '#cbd5e1' },
        { label: '20h', color: '#22c55e' },
        { label: '50h', color: '#3b82f6' },
        { label: '100h', color: '#8b5cf6' },
        { label: '200h', color: '#f97316' },
        { label: '500h', color: '#ef4444' },
        { label: '800h', color: '#facc15' },
        { label: '1000h', color: '#111111' },
      ],
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
      if (this.loadVersion > 0) {
        getClient().abort();
      }

      const version = ++this.loadVersion;
      this.loading = true;
      this.error = '';
      this.lifetimeError = '';
      this.rankingScope = 'week';
      this.summary = { today: 0, week: 0, month: 0, lifetime: 0 };
      this.topApps = [];
      this.lifetimeApps = [];
      this.lifetimeAppTotals = {};
      this.topCategories = [];
      this.dailyTrend = [];
      this.lifetimeProgress = { done: 0, total: 0 };

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

      if (version === this.loadVersion && this.hostParam && !this.error) {
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
        this.lifetimeAppTotals = Object.fromEntries(appTotals.entries());
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

    appIconUrl(name: string): string {
      const normalized = String(name || '').toLowerCase();
      const match = APP_ICON_MATCHES.find(item =>
        item.needles.some(needle => normalized.includes(needle))
      );
      return match ? `https://cdn.simpleicons.org/${match.slug}` : '';
    },

    appInitial(name: string): string {
      const trimmed = String(name || '?').trim();
      return trimmed ? trimmed.charAt(0).toUpperCase() : '?';
    },

    onIconError(event: Event): void {
      const target = event.currentTarget as HTMLImageElement | null;
      if (target) target.style.display = 'none';
    },

    appLifetimeDuration(name: string, fallbackDuration: number): number {
      return this.lifetimeAppTotals[name] || fallbackDuration || 0;
    },

    usageTier(name: string, fallbackDuration: number): UsageTier {
      const hours = this.appLifetimeDuration(name, fallbackDuration) / 3600;
      return USAGE_TIERS.find(tier => hours >= tier.hours) || USAGE_TIERS[USAGE_TIERS.length - 1];
    },

    usageTierColor(name: string, fallbackDuration: number): string {
      return this.usageTier(name, fallbackDuration).color;
    },

    usageTierLabel(name: string, fallbackDuration: number): string {
      return this.usageTier(name, fallbackDuration).label;
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
  max-width: 1220px;
  margin: 0 auto;
}

.dashboard-header h2 {
  font-weight: 760;
  letter-spacing: -0.035em;
}

.eyebrow,
.section-kicker,
.hero-kicker {
  font-size: 0.72rem;
  letter-spacing: 0.14em;
  font-weight: 700;
  text-transform: uppercase;
  opacity: 0.58;
}

.update-line {
  min-height: 1.25rem;
}

.overview-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.6fr) minmax(260px, 0.8fr);
  gap: 1rem;
}

.hero-card,
.summary-card,
.dashboard-panel {
  border: 1px solid rgba(127, 127, 127, 0.18);
  background: linear-gradient(145deg, rgba(127, 127, 127, 0.055), rgba(127, 127, 127, 0.018));
  box-shadow: 0 10px 30px rgba(20, 25, 35, 0.045);
}

.hero-card {
  min-height: 245px;
  padding: 1.5rem;
  border-radius: 22px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.hero-value {
  margin: 0.2rem 0 0.3rem;
  font-size: clamp(2.5rem, 6vw, 4.7rem);
  line-height: 0.98;
  font-weight: 760;
  letter-spacing: -0.06em;
}

.hero-subtitle,
.summary-note,
.app-meta {
  color: #7c8188;
}

.hero-subtitle {
  font-size: 0.85rem;
}

.hero-divider {
  height: 1px;
  margin: 1.35rem 0 1rem;
  background: rgba(127, 127, 127, 0.16);
}

.hero-stats {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 1rem;
}

.hero-stat {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.hero-stat span {
  margin-bottom: 0.18rem;
  color: #858a90;
  font-size: 0.75rem;
}

.hero-stat strong {
  overflow: hidden;
  font-size: 1rem;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.side-summary {
  display: grid;
  grid-template-rows: 1fr 1fr;
  gap: 1rem;
}

.summary-card {
  padding: 1.2rem;
  border-radius: 18px;
}

.summary-label {
  color: #858a90;
  font-size: 0.76rem;
  font-weight: 650;
}

.summary-value {
  margin: 0.28rem 0 0.2rem;
  font-size: clamp(1.65rem, 3vw, 2.25rem);
  line-height: 1.05;
  font-weight: 740;
  letter-spacing: -0.04em;
}

.summary-period {
  margin: 0.35rem 0 0.2rem;
  font-size: 1.05rem;
  font-weight: 680;
}

.summary-note {
  font-size: 0.76rem;
}

.content-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.55fr) minmax(300px, 0.75fr);
  gap: 1rem;
}

.lower-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
}

.dashboard-panel {
  padding: 1.25rem;
  border-radius: 18px;
}

.panel-heading {
  margin-bottom: 1rem;
}

.panel-heading h4 {
  font-weight: 720;
  letter-spacing: -0.025em;
}

.device-pill {
  padding: 0.32rem 0.62rem;
  border-radius: 999px;
  background: rgba(127, 127, 127, 0.1);
  color: #747980;
  font-size: 0.74rem;
  white-space: nowrap;
}

.app-list {
  display: flex;
  flex-direction: column;
}

.app-row {
  display: grid;
  grid-template-columns: 1.8rem 2.75rem minmax(0, 1fr);
  gap: 0.72rem;
  align-items: center;
  padding: 0.62rem 0;
  border-top: 1px solid rgba(127, 127, 127, 0.09);
}

.app-row:first-child {
  border-top: 0;
}

.app-rank {
  color: #9aa0a8;
  font-size: 0.7rem;
  font-variant-numeric: tabular-nums;
  text-align: center;
}

.app-icon-shell {
  position: relative;
  width: 2.55rem;
  height: 2.55rem;
  display: grid;
  place-items: center;
  overflow: hidden;
  border: 2px solid #cbd5e1;
  border-radius: 12px;
  background: rgba(127, 127, 127, 0.055);
  transition: border-color 180ms ease;
}

.app-icon-fallback {
  font-size: 0.95rem;
  font-weight: 760;
  opacity: 0.65;
}

.app-icon {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  padding: 0.48rem;
  object-fit: contain;
  background: rgba(255, 255, 255, 0.92);
}

.app-info {
  min-width: 0;
}

.app-line {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 1rem;
}

.app-name {
  min-width: 0;
  overflow: hidden;
  font-size: 0.92rem;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.app-duration {
  color: #696f77;
  font-size: 0.82rem;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.app-meta {
  margin-top: 0.1rem;
  font-size: 0.7rem;
}

.usage-track,
.category-track {
  height: 6px;
  margin-top: 0.42rem;
  overflow: hidden;
  border-radius: 999px;
  background: rgba(127, 127, 127, 0.11);
}

.usage-fill,
.category-fill {
  height: 100%;
  border-radius: inherit;
  transition: width 220ms ease, background-color 220ms ease;
}

.usage-fill {
  box-shadow: inset 0 0 0 1px rgba(15, 23, 42, 0.08);
}

.category-fill {
  background: #6b8afd;
  opacity: 0.82;
}

.tier-legend {
  padding-top: 0.8rem;
  border-top: 1px solid rgba(127, 127, 127, 0.11);
}

.tier-legend-title {
  margin-bottom: 0.48rem;
  color: #858a90;
  font-size: 0.7rem;
}

.tier-items {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem 0.82rem;
}

.tier-item {
  display: flex;
  align-items: center;
  gap: 0.32rem;
  color: #858a90;
  font-size: 0.68rem;
}

.tier-dot {
  width: 0.58rem;
  height: 0.58rem;
  border: 1px solid transparent;
  border-radius: 50%;
}

.category-row {
  padding: 0.5rem 0;
  font-size: 0.88rem;
}

.streak-box {
  display: flex;
  align-items: baseline;
  gap: 0.45rem;
}

.streak-number {
  font-size: 2rem;
  font-weight: 740;
  letter-spacing: -0.04em;
}

.streak-copy {
  color: #7b8086;
  font-size: 0.8rem;
}

.trend-chart {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 0.45rem;
  height: 205px;
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
  color: #7d8288;
  font-size: 0.68rem;
}

.trend-bar-wrap {
  height: 146px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  border-bottom: 1px solid rgba(127, 127, 127, 0.16);
}

.trend-bar {
  width: min(28px, 70%);
  min-height: 2px;
  border-radius: 6px 6px 2px 2px;
  background: linear-gradient(180deg, #7d95f7, #5f79e8);
  opacity: 0.88;
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
  min-height: 92px;
  padding: 0.85rem;
  border: 1px solid rgba(127, 127, 127, 0.15);
  border-radius: 12px;
  color: inherit;
  text-decoration: none;
  background: rgba(127, 127, 127, 0.025);
  transition: transform 150ms ease, border-color 150ms ease, background-color 150ms ease;
}

.quick-card:hover {
  border-color: rgba(107, 138, 253, 0.48);
  background: rgba(107, 138, 253, 0.045);
  text-decoration: none;
  transform: translateY(-1px);
}

.quick-card span {
  margin-top: 0.35rem;
  color: #7c8188;
  font-size: 0.76rem;
  line-height: 1.35;
}

.empty-state {
  padding: 1rem 0;
  color: #858a90;
  font-size: 0.86rem;
}

.small-spinner {
  width: 1.15rem;
  height: 1.15rem;
}

@media (max-width: 991.98px) {
  .overview-grid,
  .content-grid,
  .lower-grid {
    grid-template-columns: 1fr;
  }

  .side-summary {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    grid-template-rows: none;
  }
}

@media (max-width: 575.98px) {
  .dashboard-header h2 {
    font-size: 1.7rem;
  }

  .hero-card {
    min-height: 220px;
    padding: 1.1rem;
  }

  .hero-stats,
  .side-summary {
    grid-template-columns: 1fr;
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

  .app-row {
    grid-template-columns: 1.45rem 2.4rem minmax(0, 1fr);
    gap: 0.52rem;
  }

  .app-icon-shell {
    width: 2.2rem;
    height: 2.2rem;
    border-radius: 10px;
  }

  .tier-items {
    gap: 0.42rem 0.62rem;
  }
}
</style>
