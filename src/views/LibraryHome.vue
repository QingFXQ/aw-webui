<template lang="pug">
div.library-home
  header.library-header
    div
      div.library-eyebrow ACTIVITYWATCH
      h1 我的软件库
      p 只看真正活跃的使用时间。
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
    span 最近刷新 {{ lastUpdated }}
    span(v-if="deviceLabel") · {{ deviceLabel }}
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
      span 当前周期
      span 累计

    div.library-empty(v-if="filteredLibraryApps.length === 0")
      | {{ searchQuery ? '没有匹配的软件。' : '暂时没有软件活动数据。' }}

    div.library-list(v-else)
      div.library-row(v-for="app in filteredLibraryApps" :key="app.name")
        div.app-visual(:style="{ borderColor: usageTierColor(app.name, totalDurationFor(app)) }")
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
            span.tier-label {{ usageTierLabel(app.name, totalDurationFor(app)) }}
          div.app-hours-row
            span.primary-hours {{ hoursDuration(totalDurationFor(app)) }}
            span.period-hours {{ periodLabel }} {{ hoursDuration(periodDurationFor(app)) }}
          div.app-progress
            div.app-progress-fill(
              :style="{ width: libraryBarWidth(app) + '%', backgroundColor: usageTierColor(app.name, totalDurationFor(app)) }"
            )

        div.period-cell
          strong {{ hoursDuration(periodDurationFor(app)) }}
          span {{ periodLabel }}

        div.total-cell
          strong(v-if="!lifetimeLoading || lifetimeAppTotals[app.name]") {{ hoursDuration(totalDurationFor(app)) }}
          strong(v-else) …
          span 累计

  footer.library-footer(v-if="hostParam && !loading")
    span 颜色按累计使用时长分档；进度条表示当前列表中的相对使用量。
    router-link(:to="advancedActivityPath") 查看 ActivityWatch 原版详细统计 →
</template>

<script lang="ts">
import Home from './Home.vue';

interface LibraryItem {
  name: string;
  duration: number;
}

export default {
  name: 'LibraryHome',
  extends: Home,
  data() {
    return {
      searchQuery: '',
      sortMode: 'total' as 'total' | 'period' | 'name',
      scopeOptions: [
        { value: 'week', text: '本周' },
        { value: 'lifetime', text: '总时长' },
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
      return this.rankingScope === 'lifetime' ? '总时长' : '本周';
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
        return this.totalDurationFor(b) - this.totalDurationFor(a);
      });
    },
    libraryMaxDuration(): number {
      return Math.max(
        1,
        ...this.filteredLibraryApps.map((item: LibraryItem) =>
          this.rankingScope === 'lifetime'
            ? this.totalDurationFor(item)
            : this.periodDurationFor(item)
        )
      );
    },
  },
  methods: {
    snapshotApps(events: any[], limit = 40): LibraryItem[] {
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
    totalDurationFor(app: LibraryItem): number {
      return Number(this.lifetimeAppTotals[app.name] || app.duration || 0);
    },
    periodDurationFor(app: LibraryItem): number {
      if (this.rankingScope === 'lifetime') return this.totalDurationFor(app);
      return this.weekDurationFor(app.name);
    },
    hoursDuration(seconds: number): string {
      if (!seconds || seconds < 60) return '0h';
      const hours = seconds / 3600;
      if (hours >= 100) return `${Math.round(hours)}h`;
      if (hours >= 10) return `${hours.toFixed(1)}h`;
      return `${hours.toFixed(1)}h`;
    },
    libraryBarWidth(app: LibraryItem): number {
      const duration =
        this.rankingScope === 'lifetime' ? this.totalDurationFor(app) : this.periodDurationFor(app);
      return Math.max(2, Math.round((duration / this.libraryMaxDuration) * 100));
    },
  },
};
</script>

<style lang="scss" scoped>
.library-home {
  --surface: #111315;
  --surface-2: #17191c;
  --surface-3: #1d2024;
  --line: rgba(255, 255, 255, 0.075);
  --muted: #83878d;
  --text: #f1f2f4;
  --soft: #b9bcc1;
  max-width: 1120px;
  min-height: calc(100vh - 86px);
  margin: 0 auto;
  padding: 0.7rem 0 3rem;
  color: var(--text);
}

.library-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem 0 1.15rem;
}

.library-header h1 {
  margin: 0.12rem 0 0.2rem;
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
  color: #73777e;
  font-size: 0.68rem;
  font-weight: 750;
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
  background: var(--surface-2);
}

.library-summary {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  border-top: 1px solid var(--line);
  border-bottom: 1px solid var(--line);
  background: rgba(255, 255, 255, 0.015);
}

.summary-item {
  min-width: 0;
  padding: 0.9rem 1rem;
  border-left: 1px solid var(--line);
}

.summary-item:first-child {
  border-left: 0;
}

.summary-item span {
  display: block;
  margin-bottom: 0.18rem;
  color: var(--muted);
  font-size: 0.72rem;
}

.summary-item strong {
  display: block;
  overflow: hidden;
  font-size: 1.24rem;
  font-variant-numeric: tabular-nums;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.summary-primary strong {
  font-size: 1.48rem;
}

.library-toolbar {
  display: grid;
  grid-template-columns: minmax(210px, 1fr) 140px 150px auto;
  gap: 0.55rem;
  align-items: center;
  padding: 0.95rem 0 0.7rem;
}

.search-wrap {
  position: relative;
}

.search-symbol {
  position: absolute;
  z-index: 2;
  left: 0.72rem;
  top: 50%;
  color: #676b72;
  font-size: 1.25rem;
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

.library-toolbar .form-control::placeholder {
  color: #656970;
}

.library-status {
  padding: 0 0 0.65rem;
  color: #6f737a;
  font-size: 0.7rem;
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
  grid-template-columns: minmax(0, 1fr) 110px 110px;
  gap: 1rem;
  padding: 0.55rem 0.85rem;
  border-bottom: 1px solid var(--line);
  color: #666b72;
  font-size: 0.68rem;
  text-align: right;
}

.library-list-head span:first-child {
  padding-left: 5.55rem;
  text-align: left;
}

.library-list {
  background: var(--surface);
}

.library-row {
  display: grid;
  grid-template-columns: 4.5rem minmax(0, 1fr) 110px 110px;
  gap: 1rem;
  align-items: center;
  min-height: 86px;
  padding: 0.7rem 0.85rem;
  border-bottom: 1px solid var(--line);
  transition: background-color 120ms ease;
}

.library-row:hover {
  background: rgba(255, 255, 255, 0.028);
}

.app-visual {
  position: relative;
  width: 4.5rem;
  height: 3.45rem;
  display: grid;
  place-items: center;
  overflow: hidden;
  border: 2px solid #34383e;
  border-radius: 9px;
  background: #202328;
}

.app-fallback {
  color: #7f848c;
  font-size: 1.25rem;
  font-weight: 760;
}

.app-icon {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  padding: 0.72rem 1rem;
  object-fit: contain;
  background: #eff1f4;
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

.tier-label {
  color: #62676e;
  font-size: 0.64rem;
  white-space: nowrap;
}

.app-hours-row {
  margin-top: 0.28rem;
}

.primary-hours {
  color: #d8dadd;
  font-size: 0.83rem;
  font-variant-numeric: tabular-nums;
}

.period-hours {
  color: #777b82;
  font-size: 0.72rem;
  font-variant-numeric: tabular-nums;
}

.app-progress {
  height: 5px;
  margin-top: 0.48rem;
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
.total-cell {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  min-width: 0;
}

.period-cell strong,
.total-cell strong {
  color: #d8dadd;
  font-size: 0.86rem;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.period-cell span,
.total-cell span {
  margin-top: 0.14rem;
  color: #666b72;
  font-size: 0.65rem;
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
  padding: 0.85rem 0.2rem;
  color: #646970;
  font-size: 0.68rem;
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
    grid-template-columns: 3.9rem minmax(0, 1fr);
    gap: 0.78rem;
    min-height: 80px;
    padding: 0.65rem 0.2rem;
  }

  .app-visual {
    width: 3.9rem;
    height: 3.1rem;
  }

  .period-cell,
  .total-cell {
    display: none;
  }

  .tier-label {
    display: none;
  }

  .library-footer {
    flex-direction: column;
  }
}
</style>
