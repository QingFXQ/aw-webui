<template lang="pug">
div.library-home
  header.library-header
    div
      div.library-eyebrow ACTIVITYWATCH
      h1 我的软件库
      p 把电脑使用时间当成一份长期记录来看。
    div.library-header-actions
      b-dropdown(
        v-if="availableHosts.length > 1"
        size="sm"
        variant="dark"
        :text="deviceLabel || '设备'"
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
      b-button(size="sm" variant="dark" to="/timeline") 时间线
      b-button(size="sm" variant="dark" :to="advancedActivityPath" :disabled="!hostParam") 详细
      b-button(size="sm" variant="dark" to="/settings") 设置

  nav.library-tabs
    span.library-tab.active 软件
    router-link.library-tab(:to="advancedActivityPath") 趋势

  div.library-summary(v-if="hostParam")
    div.summary-item.summary-primary
      span 今天
      strong {{ compactDuration(summary.today) }}
    div.summary-item
      span 本周
      strong {{ compactDuration(summary.week) }}
    div.summary-item
      span 本月
      strong {{ compactDuration(summary.month) }}
    div.summary-item
      span 累计
      strong(v-if="!lifetimeLoading && !lifetimeError") {{ compactDuration(summary.lifetime) }}
      strong(v-else-if="lifetimeLoading") …
      strong(v-else) —

  div.library-toolbar
    div.search-wrap
      span.search-symbol ⌕
      b-form-input(
        v-model.trim="searchQuery"
        size="sm"
        placeholder="搜索软件"
        autocomplete="off"
      )
    b-form-select.scope-select(v-model="rankingScope" size="sm" :options="scopeOptions")
    b-form-select.sort-select(v-model="sortMode" size="sm" :options="sortOptions")
    b-button.refresh-button(size="sm" variant="dark" :disabled="loading" @click="loadDashboard")
      span(v-if="!loading") 刷新
      span(v-else) 加载中…

  div.library-status(v-if="lastUpdated && !loading")
    span {{ filteredLibraryApps.length }} 个软件
    span(v-if="deviceLabel") · {{ deviceLabel }}
    span · 最近刷新 {{ lastUpdated }}
    span · 已排除 AFK

  b-alert(v-if="error" show variant="danger") {{ error }}
  b-alert(v-else-if="!loading && !hostParam" show variant="warning")
    | 暂时没有发现可统计的设备。保持 ActivityWatch 运行一会儿后刷新。

  div.library-loading(v-if="loading")
    b-spinner.small-spinner
    span 正在整理软件使用记录…

  template(v-else-if="hostParam")
    div.library-list-head
      span 软件
      span {{ periodLabel }}
      span 占比

    div.library-empty(v-if="filteredLibraryApps.length === 0")
      | {{ searchQuery ? '没有匹配的软件。' : '暂时没有软件活动数据。' }}

    div.library-list(v-else)
      div.library-row(v-for="app in filteredLibraryApps" :key="app.name")
        div.app-visual(:style="appCoverStyle(app.name)")
          span.app-fallback {{ appInitial(app.name) }}
          img.app-icon(
            v-if="appIconUrl(app.name)"
            :src="appIconUrl(app.name)"
            :alt="app.name"
            referrerpolicy="no-referrer"
            @error="onIconError"
          )

        div.app-main
          div.app-title-row
            strong.app-title {{ app.name }}
            span.total-inline(v-if="hasLifetimeDuration(app.name)") 累计 {{ hoursDuration(totalDurationFor(app)) }}
            span.total-inline(v-else-if="lifetimeLoading") 累计 …
            span.total-inline(v-else) 累计 —
          div.app-hours-row
            span.primary-hours {{ hoursDuration(periodDurationFor(app)) }}
            span.period-hours {{ periodLabel }}
          div.app-progress
            div.app-progress-fill(
              :style="{ width: libraryBarWidth(app) + '%', backgroundColor: appAccent(app.name) }"
            )

        div.period-cell
          strong {{ hoursDuration(periodDurationFor(app)) }}
          span {{ periodLabel }}

        div.share-cell
          strong {{ shareFor(app) }}%
          span {{ periodLabel }}占比

  footer.library-footer(v-if="hostParam && !loading")
    span 进度条和占比都按当前筛选周期计算；累计时长在后台分段读取。
    router-link(:to="advancedActivityPath") 查看 ActivityWatch 原版详细统计 →
</template>

<script lang="ts">
import moment from 'moment';

import Home from './Home.vue';
import { get_today_with_offset } from '~/util/time';

interface LibraryItem {
  name: string;
  duration: number;
}

const APP_ACCENTS = ['#00c2ff', '#ff9f0a', '#a78bfa', '#34d399', '#fb7185', '#60a5fa'];

export default {
  name: 'LibraryHome',
  extends: Home,
  data() {
    return {
      searchQuery: '',
      sortMode: 'total' as 'total' | 'period' | 'name',
      scopeOptions: [
        { value: 'week', text: '本周' },
        { value: 'lifetime', text: '全部时间' },
      ],
      sortOptions: [
        { value: 'total', text: '按总时长' },
        { value: 'period', text: '按当前周期' },
        { value: 'name', text: '按名称' },
      ],
    };
  },
  computed: {
    periodLabel(): string {
      return this.rankingScope === 'lifetime' ? '全部时间' : '本周';
    },
    periodTotal(): number {
      return this.rankingScope === 'lifetime' ? this.summary.lifetime : this.summary.week;
    },
    libraryApps(): LibraryItem[] {
      const byName = new Map<string, LibraryItem>();
      const add = (item: LibraryItem) => {
        const current = byName.get(item.name);
        if (!current || item.duration > current.duration) byName.set(item.name, item);
      };
      (this.topApps || []).forEach(add);
      (this.lifetimeApps || []).forEach(add);
      return Array.from(byName.values());
    },
    filteredLibraryApps(): LibraryItem[] {
      const query = this.searchQuery.toLowerCase();
      const items = this.libraryApps.filter((item: LibraryItem) =>
        item.name.toLowerCase().includes(query)
      );

      return items.sort((a: LibraryItem, b: LibraryItem) => {
        if (this.sortMode === 'name') return a.name.localeCompare(b.name);
        if (this.sortMode === 'period') {
          return this.periodDurationFor(b) - this.periodDurationFor(a);
        }

        const aTotal = this.hasLifetimeDuration(a.name)
          ? this.totalDurationFor(a)
          : this.periodDurationFor(a);
        const bTotal = this.hasLifetimeDuration(b.name)
          ? this.totalDurationFor(b)
          : this.periodDurationFor(b);
        return bTotal - aTotal;
      });
    },
    libraryMaxDuration(): number {
      return Math.max(
        1,
        ...this.filteredLibraryApps.map((item: LibraryItem) => this.periodDurationFor(item))
      );
    },
  },
  methods: {
    snapshotApps(events: any[], limit = 80): LibraryItem[] {
      return (events || [])
        .map((event: any) => ({
          name: event.data && event.data.app ? String(event.data.app) : '未知应用',
          duration: Number(event.duration || 0),
        }))
        .filter((item: LibraryItem) => item.duration > 0)
        .slice(0, limit);
    },

    weekDurationFor(name: string): number {
      const item = (this.topApps || []).find((app: LibraryItem) => app.name === name);
      return item ? item.duration : 0;
    },

    hasLifetimeDuration(name: string): boolean {
      return Object.prototype.hasOwnProperty.call(this.lifetimeAppTotals, name);
    },

    totalDurationFor(app: LibraryItem): number {
      return Number(this.lifetimeAppTotals[app.name] || 0);
    },

    periodDurationFor(app: LibraryItem): number {
      if (this.rankingScope === 'lifetime') return this.totalDurationFor(app);
      return this.weekDurationFor(app.name);
    },

    hoursDuration(seconds: number): string {
      if (!seconds || seconds < 60) return '0h';
      const hours = seconds / 3600;
      if (hours >= 1000) return `${Math.round(hours).toLocaleString()}h`;
      if (hours >= 100) return `${Math.round(hours)}h`;
      return `${hours.toFixed(1)}h`;
    },

    libraryBarWidth(app: LibraryItem): number {
      const duration = this.periodDurationFor(app);
      return Math.max(duration > 0 ? 2 : 0, Math.round((duration / this.libraryMaxDuration) * 100));
    },

    shareFor(app: LibraryItem): number {
      if (!this.periodTotal) return 0;
      return Math.max(0, Math.min(100, Math.round((this.periodDurationFor(app) / this.periodTotal) * 100)));
    },

    appAccent(name: string): string {
      let hash = 0;
      for (let i = 0; i < name.length; i += 1) {
        hash = (hash * 31 + name.charCodeAt(i)) >>> 0;
      }
      return APP_ACCENTS[hash % APP_ACCENTS.length];
    },

    appCoverStyle(name: string): Record<string, string> {
      const accent = this.appAccent(name);
      return {
        borderColor: accent,
        background: `linear-gradient(135deg, ${accent}33 0%, #202328 58%, #141619 100%)`,
      };
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

          for (const app of this.snapshotApps(aggregate.app_events, 160)) {
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
          .slice(0, 80);
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
  },
};
</script>

<style lang="scss" scoped>
.library-home {
  --page: #0f1113;
  --surface: #121416;
  --surface-2: #171a1d;
  --surface-3: #202329;
  --line: rgba(255, 255, 255, 0.075);
  --muted: #7d8289;
  --text: #f2f3f5;
  --soft: #c2c5ca;
  max-width: 1160px;
  min-height: calc(100vh - 86px);
  margin: 0 auto;
  padding: 0.7rem 0 3rem;
  color: var(--text);
  background: var(--page);
  box-shadow: 0 0 0 100vmax var(--page);
  clip-path: inset(0 -100vmax);
}

.library-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem 0 0.9rem;
}

.library-header h1 {
  margin: 0.12rem 0 0.18rem;
  font-size: 2rem;
  font-weight: 760;
  letter-spacing: -0.045em;
}

.library-header p {
  margin: 0;
  color: var(--muted);
  font-size: 0.84rem;
}

.library-eyebrow {
  color: #666b72;
  font-size: 0.66rem;
  font-weight: 760;
  letter-spacing: 0.16em;
}

.library-header-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.45rem;
}

.library-header-actions .btn,
.refresh-button {
  border-color: var(--line);
  color: #c8cbd0;
  background: var(--surface-2);
}

.library-tabs {
  display: flex;
  gap: 1.4rem;
  border-bottom: 1px solid var(--line);
}

.library-tab {
  position: relative;
  padding: 0.55rem 0 0.65rem;
  color: #777c83;
  font-size: 0.92rem;
  text-decoration: none;
}

.library-tab:hover {
  color: #d7d9dc;
  text-decoration: none;
}

.library-tab.active {
  color: #f2f3f5;
  font-weight: 650;
}

.library-tab.active::after {
  position: absolute;
  right: 0;
  bottom: -1px;
  left: 0;
  height: 2px;
  content: '';
  background: #e9eaec;
}

.library-summary {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  margin-top: 0.8rem;
  border: 1px solid var(--line);
  border-radius: 10px;
  overflow: hidden;
  background: #121416;
}

.summary-item {
  min-width: 0;
  padding: 0.78rem 1rem;
  border-left: 1px solid var(--line);
}

.summary-item:first-child {
  border-left: 0;
}

.summary-item span {
  display: block;
  margin-bottom: 0.15rem;
  color: var(--muted);
  font-size: 0.69rem;
}

.summary-item strong {
  display: block;
  overflow: hidden;
  font-size: 1.18rem;
  font-variant-numeric: tabular-nums;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.summary-primary strong {
  font-size: 1.36rem;
}

.library-toolbar {
  display: grid;
  grid-template-columns: minmax(240px, 1fr) 140px 150px auto;
  gap: 0.55rem;
  align-items: center;
  padding: 0.82rem 0 0.45rem;
}

.search-wrap {
  position: relative;
}

.search-symbol {
  position: absolute;
  z-index: 2;
  left: 0.72rem;
  top: 50%;
  color: #676c73;
  font-size: 1.22rem;
  transform: translateY(-53%);
  pointer-events: none;
}

.search-wrap input {
  padding-left: 2rem;
}

.library-toolbar .form-control,
.library-toolbar .custom-select {
  height: 2.15rem;
  border-color: var(--line);
  color: var(--soft);
  background-color: var(--surface-2);
  box-shadow: none;
}

.library-toolbar .form-control:focus,
.library-toolbar .custom-select:focus {
  border-color: rgba(255, 255, 255, 0.17);
}

.library-toolbar .form-control::placeholder {
  color: #62666d;
}

.library-status {
  padding: 0.05rem 0 0.6rem;
  color: #666b72;
  font-size: 0.68rem;
}

.library-loading {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.65rem;
  min-height: 260px;
  color: var(--muted);
}

.small-spinner {
  width: 1.05rem;
  height: 1.05rem;
}

.library-list-head {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 112px 88px;
  gap: 1rem;
  padding: 0.48rem 0.9rem;
  border-bottom: 1px solid var(--line);
  color: #5f646b;
  font-size: 0.66rem;
  text-align: right;
}

.library-list-head span:first-child {
  padding-left: 8.8rem;
  text-align: left;
}

.library-list {
  border-top: 1px solid rgba(255, 255, 255, 0.025);
  background: var(--surface);
}

.library-row {
  display: grid;
  grid-template-columns: 7.7rem minmax(0, 1fr) 112px 88px;
  gap: 1rem;
  align-items: center;
  min-height: 92px;
  padding: 0.62rem 0.9rem;
  border-bottom: 1px solid var(--line);
  transition: background-color 120ms ease;
}

.library-row:hover {
  background: rgba(255, 255, 255, 0.028);
}

.app-visual {
  position: relative;
  width: 7.7rem;
  height: 4.4rem;
  display: grid;
  place-items: center;
  overflow: hidden;
  border: 2px solid #34383e;
  border-radius: 8px;
}

.app-fallback {
  color: rgba(255, 255, 255, 0.42);
  font-size: 1.5rem;
  font-weight: 760;
}

.app-icon {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  padding: 0.92rem 2rem;
  object-fit: contain;
  background: transparent;
}

.app-main {
  min-width: 0;
}

.app-title-row,
.app-hours-row {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 0.75rem;
}

.app-title {
  min-width: 0;
  overflow: hidden;
  color: var(--text);
  font-size: 1rem;
  font-weight: 620;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.total-inline {
  color: #6f747b;
  font-size: 0.67rem;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.app-hours-row {
  margin-top: 0.3rem;
}

.primary-hours {
  color: #d7d9dc;
  font-size: 0.86rem;
  font-variant-numeric: tabular-nums;
}

.period-hours {
  color: #73787f;
  font-size: 0.69rem;
}

.app-progress {
  height: 5px;
  margin-top: 0.5rem;
  overflow: hidden;
  border-radius: 999px;
  background: #25282d;
}

.app-progress-fill {
  height: 100%;
  border-radius: inherit;
  transition: width 180ms ease;
}

.period-cell,
.share-cell {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  min-width: 0;
}

.period-cell strong,
.share-cell strong {
  color: #dadcdf;
  font-size: 0.86rem;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.period-cell span,
.share-cell span {
  margin-top: 0.12rem;
  color: #5f646b;
  font-size: 0.63rem;
  white-space: nowrap;
}

.library-empty {
  padding: 4rem 1rem;
  color: var(--muted);
  text-align: center;
}

.library-footer {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  padding: 0.82rem 0.2rem;
  color: #5f646b;
  font-size: 0.66rem;
}

.library-footer a {
  color: #8fa9ff;
}

@media (max-width: 767.98px) {
  .library-home {
    padding: 0 0.15rem 2rem;
  }

  .library-header {
    align-items: flex-start;
    flex-direction: column;
  }

  .library-summary {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .summary-item:nth-child(3) {
    border-left: 0;
    border-top: 1px solid var(--line);
  }

  .summary-item:nth-child(4) {
    border-top: 1px solid var(--line);
  }

  .library-toolbar {
    grid-template-columns: 1fr 1fr;
  }

  .search-wrap {
    grid-column: 1 / -1;
  }

  .refresh-button {
    display: none;
  }

  .library-list-head {
    display: none;
  }

  .library-row {
    grid-template-columns: 5.7rem minmax(0, 1fr);
    gap: 0.72rem;
    min-height: 82px;
    padding: 0.58rem 0.15rem;
  }

  .app-visual {
    width: 5.7rem;
    height: 3.4rem;
  }

  .app-icon {
    padding: 0.7rem 1.4rem;
  }

  .period-cell,
  .share-cell {
    display: none;
  }

  .total-inline {
    display: none;
  }

  .library-footer {
    flex-direction: column;
  }
}
</style>
