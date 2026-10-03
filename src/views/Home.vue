<template lang="pug">
div.personal-dashboard
  div.dashboard-header.d-flex.flex-wrap.align-items-end.justify-content-between.mb-4
    div
      p.eyebrow.mb-1 ACTIVITYWATCH · PERSONAL
      h2.mb-1 我的电脑时间
      p.text-muted.mb-0 自动统计真实活跃时间，像 Steam 一样看见长期投入。
    div.d-flex.mt-3.mt-md-0
      b-button.mr-2(size="sm" variant="outline-secondary" to="/timeline") 时间线
      b-button.mr-2(size="sm" variant="outline-secondary" :to="advancedActivityPath" :disabled="!hostParam") 高级统计
      b-button(size="sm" variant="outline-secondary" to="/settings") 设置

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
          div.metric-value(v-if="!lifetimeLoading") {{ formatDuration(summary.lifetime) }}
          div.metric-value(v-else) …
          div.metric-subtitle(v-if="!lifetimeLoading") 从最早记录开始
          div.metric-subtitle(v-else) 后台计算中

    div.row.mb-3
      div.col-lg-7.mb-3
        div.dashboard-panel.h-100
          div.panel-heading.d-flex.align-items-center.justify-content-between
            div
              h4.mb-1 软件排行榜
              p.text-muted.small.mb-0 本周真实活跃时长
            span.device-pill {{ deviceLabel }}

          div.empty-state(v-if="topApps.length === 0") 暂时没有软件活动数据。
          div.app-row(v-for="(app, index) in topApps" :key="app.name")
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
            span.streak-copy 连续投入

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

import { useActivityStore, QueryOptions } from '~/stores/activity';
import { useBucketsStore } from '~/stores/buckets';
import { useSettingsStore } from '~/stores/settings';
import { ALL_DEVICES, eligibleMultideviceHosts } from '~/util/multidevice';
import { get_day_start_with_offset, get_today_with_offset } from '~/util/time';

interface RankedItem {
  name: string;
  duration: number;
}

interface TrendDay {
  date: string;
  label: string;
  duration: number;
}

export default {
  name: 'Home',
  data() {
    return {
      loading: true,
      lifetimeLoading: false,
      error: '',
      hostParam: '',
      deviceLabel: '',
      summary: {
        today: 0,
        week: 0,
        month: 0,
        lifetime: 0,
      },
      topApps: [] as RankedItem[],
      topCategories: [] as RankedItem[],
      dailyTrend: [] as TrendDay[],
      activityStore: useActivityStore(),
      bucketsStore: useBucketsStore(),
      settingsStore: useSettingsStore(),
    };
  },
  computed: {
    advancedActivityPath(): string {
      if (!this.hostParam) return '/activity';
      return `/activity/${this.hostParam}/day`;
    },
    weekLabel(): string {
      return `${moment().startOf('isoWeek').format('M月D日')} – 今天`;
    },
    monthLabel(): string {
      return moment().format('YYYY年M月');
    },
    maxAppDuration(): number {
      return Math.max(1, ...this.topApps.map((item: RankedItem) => item.duration));
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
      return streak === this.dailyTrend.length ? `${streak}+ 天` : `${streak} 天`;
    },
  },
  async mounted() {
    await this.loadDashboard();
  },
  beforeDestroy() {
    this.activityStore.reset();
  },
  methods: {
    makePeriod(startDate: string, days: number): any {
      return {
        start: get_day_start_with_offset(startDate, this.settingsStore.startOfDay),
        length: [Math.max(1, days), 'days'],
      };
    },

    async queryPeriod(startDate: string, days: number): Promise<void> {
      const options: QueryOptions = {
        host: this.hostParam,
        timeperiod: this.makePeriod(startDate, days),
        filter_afk: true,
        include_audible: true,
        include_stopwatch: false,
        skip_active_history: true,
        force: true,
        always_active_pattern: this.settingsStore.always_active_pattern,
      };
      await this.activityStore.ensure_loaded(options);
    },

    snapshotApps(): RankedItem[] {
      return (this.activityStore.window.top_apps || [])
        .map((event: any) => ({
          name: event.data && event.data.app ? String(event.data.app) : '未知应用',
          duration: Number(event.duration || 0),
        }))
        .filter((item: RankedItem) => item.duration > 0)
        .slice(0, 10);
    },

    snapshotCategories(): RankedItem[] {
      return (this.activityStore.category.top || [])
        .map((event: any) => {
          const raw = event.data ? event.data['$category'] : null;
          const name = Array.isArray(raw) ? raw.join(' › ') : raw ? String(raw) : '未分类';
          return { name, duration: Number(event.duration || 0) };
        })
        .filter((item: RankedItem) => item.duration > 0)
        .slice(0, 8);
    },

    async loadDashboard(): Promise<void> {
      this.loading = true;
      this.error = '';
      try {
        await this.settingsStore.ensureLoaded();
        await this.bucketsStore.ensureLoaded();

        const eligibleHosts = eligibleMultideviceHosts(
          this.bucketsStore.hosts,
          this.bucketsStore.bucketsWindow,
          this.bucketsStore.bucketsAFK,
          this.bucketsStore.bucketsAndroid
        );

        if (eligibleHosts.length === 0) {
          this.hostParam = '';
          return;
        }

        this.hostParam = eligibleHosts.length > 1 ? ALL_DEVICES : eligibleHosts[0];
        this.deviceLabel =
          eligibleHosts.length > 1 ? `${eligibleHosts.length} 台设备` : eligibleHosts[0];

        const today = get_today_with_offset(this.settingsStore.startOfDay);

        await this.queryPeriod(today, 1);
        this.summary.today = this.activityStore.active.duration || 0;

        const weekStart = moment(today).startOf('isoWeek').format('YYYY-MM-DD');
        const weekDays = moment(today).diff(moment(weekStart), 'days') + 1;
        await this.queryPeriod(weekStart, weekDays);
        this.summary.week = this.activityStore.active.duration || 0;
        this.topApps = this.snapshotApps();
        this.topCategories = this.snapshotCategories();

        const monthStart = moment(today).startOf('month').format('YYYY-MM-DD');
        const monthDays = moment(today).diff(moment(monthStart), 'days') + 1;
        await this.queryPeriod(monthStart, monthDays);
        this.summary.month = this.activityStore.active.duration || 0;

        const trend: TrendDay[] = [];
        for (let offset = 6; offset >= 0; offset -= 1) {
          const date = moment(today).subtract(offset, 'days').format('YYYY-MM-DD');
          await this.queryPeriod(date, 1);
          trend.push({
            date,
            label: moment(date).format('dd'),
            duration: this.activityStore.active.duration || 0,
          });
        }
        this.dailyTrend = trend;
      } catch (e) {
        const message = e instanceof Error ? e.message : String(e);
        if (message !== 'canceled') {
          console.error(e);
          this.error = message;
        }
      } finally {
        this.loading = false;
      }

      if (this.hostParam) {
        this.loadLifetime();
      }
    },

    async loadLifetime(): Promise<void> {
      this.lifetimeLoading = true;
      try {
        const today = get_today_with_offset(this.settingsStore.startOfDay);
        const result = await this.activityStore.get_earliest_date(this.hostParam);
        const earliest = result && result.date ? result.date : today;
        const days = moment(today).diff(moment(earliest), 'days') + 1;
        await this.queryPeriod(earliest, days);
        this.summary.lifetime = this.activityStore.active.duration || 0;
      } catch (e) {
        console.warn('Lifetime total unavailable:', e);
      } finally {
        this.lifetimeLoading = false;
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
}
</style>
