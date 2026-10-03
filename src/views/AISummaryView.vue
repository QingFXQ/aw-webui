<template lang="pug">
div
  h3.mb-3 AI 活动总结

  b-alert(variant="info" show)
    | API Key 只保存在当前页面的内存中，并直接发送给所选 LLM 服务商。
    | 页面刷新后会被清除，ActivityWatch 不会接收它。
    | 如需使用 Agent 进行更深入的分析，请参阅
    | #[a(href="https://docs.activitywatch.net/en/latest/examples/agents-and-ai.html") ActivityWatch Agent 与 AI 指南]。

  div.row.mb-3
    div.col-md-4
      b-form-group(label="设备" label-class="font-weight-bold")
        b-form-select(v-model="selectedHost" :options="hostOptions")

    div.col-md-4
      b-form-group(label="日期范围" label-class="font-weight-bold")
        b-form-select(v-model="dateRange" :options="dateRangeOptions")

    div.col-md-4
      b-form-group(label="LLM 服务商" label-class="font-weight-bold")
        b-form-select(v-model="provider" :options="providerOptions" @change="onProviderChange")

  div.row.mb-3
    div.col-md-6
      b-form-group(label="API Key" label-class="font-weight-bold")
        b-form-input(
          v-model="apiKey"
          type="password"
          placeholder="sk-..."
          autocomplete="off"
          @blur="persistConfig"
        )

    div.col-md-6
      b-form-group(label="模型" label-class="font-weight-bold")
        b-form-input(v-model="model" placeholder="例如 gpt-4o-mini" @blur="persistConfig")

  div.mb-3
    b-form-group(label="隐私" label-class="font-weight-bold")
      b-form-checkbox(v-model="excludeUncategorized")
        | 排除未分类活动
      b-form-checkbox(v-model="excludePrivateCategories")
        | 排除标记为私密的分类
        span.text-muted.ml-1(v-if="privateCategories.length")
          | （{{ privateCategories.map(c => c.join(' > ')).join('，') }}）
        span.text-muted.ml-1(v-else)
          | （暂未标记；可在分类 data 中设置 #[code private: true]）
      small.text-muted
        | 开启任一过滤条件时，会完全省略浏览器域名，因为浏览器事件没有分类信息，无法按分类可靠过滤。

  div.mb-3
    b-form-group(label="提示词" label-class="font-weight-bold")
      b-form-textarea(v-model="userPrompt" rows="3" max-rows="8")

  div.mb-4
    b-button(@click="generate" variant="primary" :disabled="loading || !apiKey || !selectedHost")
      b-spinner.mr-2(v-if="loading" small)
      | {{ loading ? '生成中…' : '生成总结' }}
    b-button.ml-2(
      v-if="aggregatedText"
      variant="outline-secondary"
      @click="dataVisible = !dataVisible"
    ) {{ dataVisible ? '隐藏上下文' : '查看发送的上下文' }}

  b-alert(v-if="error" variant="danger" show dismissible @dismissed="error = ''")
    | {{ error }}

  div(v-if="dataVisible && aggregatedText")
    b-card.mb-3
      template(slot="header")
        strong 实际发送给 LLM 的上下文
      pre.mb-0(style="white-space: pre-wrap; font-size: 0.85em") {{ aggregatedText }}

  div(v-if="llmResponse")
    b-card
      template(slot="header")
        div.d-flex.justify-content-between.align-items-center
          strong AI 总结
          b-button(size="sm" variant="outline-secondary" @click="copyResponse")
            icon(name="copy")
            |  {{ copied ? '已复制' : '复制' }}
      div(style="white-space: pre-wrap") {{ llmResponse }}
</template>

<script lang="ts">
import { useBucketsStore } from '~/stores/buckets';
import { useCategoryStore } from '~/stores/categories';
import { getClient } from '~/util/awclient';
import { analysisContextQuery } from '~/queries';
import { loadLLMConfig, saveLLMConfig, callLLM, type LLMProvider } from '~/util/aiSummary';
import {
  buildActivityContext,
  formatActivityContext,
  privateCategoriesFrom,
  type CategoryName,
} from '~/util/activityContext';
import 'vue-awesome/icons/copy';

const DEFAULT_PROMPT =
  '根据下面的活动数据，简洁总结我的时间都花在了哪里。突出主要活动和专注领域，并指出值得注意的效率或时间分配模式。';

export default {
  name: 'AISummaryView',
  data() {
    const saved = loadLLMConfig();
    return {
      bucketsStore: useBucketsStore(),
      categoryStore: useCategoryStore(),

      selectedHost: '',
      dateRange: 'last7d',
      provider: (saved.provider || 'openai') as LLMProvider,
      apiKey: saved.apiKey || '',
      model: saved.model || '',
      userPrompt: DEFAULT_PROMPT,
      excludeUncategorized: false,
      excludePrivateCategories: true,

      loading: false,
      error: '',
      aggregatedText: '',
      llmResponse: '',
      dataVisible: false,
      copied: false,
    };
  },
  computed: {
    hostOptions(): { value: string; text: string }[] {
      const buckets = this.bucketsStore.buckets || [];
      const hosts = new Set<string>();
      for (const b of buckets) {
        // This view builds a desktop query (window + AFK + browser). Exclude
        // Android buckets so Android-only hosts are not offered: aw-watcher-android
        // uses the same 'currentwindow' type but its events would be intersected
        // with a desktop AFK bucket, yielding wrong or empty results.
        if (b.type === 'currentwindow' && !b.id.startsWith('aw-watcher-android')) {
          hosts.add(b.hostname);
        }
      }
      return Array.from(hosts).map(h => ({ value: h, text: h }));
    },
    dateRangeOptions() {
      return [
        { value: 'last7d', text: '最近 7 天' },
        { value: 'last30d', text: '最近 30 天' },
        { value: 'last90d', text: '最近 90 天' },
      ];
    },
    providerOptions() {
      return [
        { value: 'openai', text: 'OpenAI' },
        { value: 'anthropic', text: 'Anthropic' },
      ];
    },
    periodDays(): number {
      return this.dateRange === 'last7d' ? 7 : this.dateRange === 'last30d' ? 30 : 90;
    },
    privateCategories(): CategoryName[] {
      return privateCategoriesFrom(this.categoryStore.classes || []);
    },
    defaultModel(): string {
      if (this.provider === 'anthropic') return 'claude-haiku-4-5-20251001';
      return 'gpt-4o-mini';
    },
  },
  async mounted() {
    await this.bucketsStore.ensureLoaded();
    await this.categoryStore.load();
    if (this.hostOptions.length > 0 && !this.selectedHost) {
      this.selectedHost = this.hostOptions[0].value;
    }
    if (!this.model) {
      this.model = this.defaultModel;
    }
  },
  methods: {
    onProviderChange() {
      if (
        !this.model ||
        this.model === 'gpt-4o-mini' ||
        this.model === 'claude-haiku-4-5-20251001'
      ) {
        this.model = this.defaultModel;
      }
      this.persistConfig();
    },
    persistConfig() {
      saveLLMConfig({
        provider: this.provider,
        apiKey: this.apiKey,
        model: this.model,
      });
    },
    async generate() {
      this.error = '';
      this.llmResponse = '';
      this.aggregatedText = '';

      if (!this.selectedHost) {
        this.error = '没有找到包含 window watcher 数据的设备。';
        return;
      }
      if (!this.apiKey.trim()) {
        this.error = '请输入 LLM API Key。';
        return;
      }

      this.loading = true;
      try {
        const summaryText = await this.buildContextText();
        this.aggregatedText = summaryText;

        const fullPrompt = `${this.userPrompt}\n\n${summaryText}`;
        const effectiveModel = this.model.trim() || this.defaultModel;

        const response = await callLLM(
          {
            provider: this.provider,
            apiKey: this.apiKey.trim(),
            model: effectiveModel,
          },
          fullPrompt
        );
        this.llmResponse = response;
      } catch (err: any) {
        this.error = err?.message || String(err);
      } finally {
        this.loading = false;
      }
    },
    async buildContextText(): Promise<string> {
      const buckets = this.bucketsStore.buckets || [];
      // Use the store's bucketsWindow() filter: it excludes aw-watcher-android
      // buckets which share the 'currentwindow' type but sort before the desktop
      // watcher, so a naive find() could silently select an Android bucket.
      const windowBucketId = this.bucketsStore.bucketsWindow(this.selectedHost)[0];
      if (!windowBucketId) {
        throw new Error(`设备 ${this.selectedHost} 没有找到 window watcher 存储桶。`);
      }
      const afkBucket = buckets.find(
        b => b.type === 'afkstatus' && b.hostname === this.selectedHost
      );
      // Only include browser buckets for the selected host; mixing buckets from
      // other hosts would send another device's domains to the LLM under the
      // selected host's summary. Use bucketsBrowser() rather than an inline
      // filter so the established 'unknown'-hostname fallback is respected: when
      // a browser bucket's hostname is 'unknown' (common in older AW setups),
      // strict equality against this.selectedHost would silently drop it.
      // The fallback is attributed at best to "some device": in multi-device
      // installs those domains may belong to another host. We keep the fallback
      // (dropping it would lose all browser data on older single-device setups)
      // but disclose it explicitly in the context text below, so the user sees
      // exactly what is being sent in the preview before it reaches the LLM.
      const browserBuckets = this.bucketsStore.bucketsBrowser(this.selectedHost);
      // Fallback buckets are those the strict per-host getter added on top of the
      // exact-host ones (i.e. 'unknown'-hostname buckets).
      const browserFallbackUsed =
        browserBuckets.length >
        this.bucketsStore.bucketsByType(this.selectedHost, 'web.tab.current').length;

      const end = new Date();
      const start = new Date(end.getTime() - this.periodDays * 24 * 60 * 60 * 1000);

      // Query-side AFK filtering and categorization: only derived statistics
      // (never raw titles or URLs) are exported from the result below.
      const query = analysisContextQuery({
        bid_window: windowBucketId,
        bid_afk: afkBucket ? afkBucket.id : '',
        bid_browsers: browserBuckets,
        filter_afk: Boolean(afkBucket),
        categories: this.categoryStore.classes_for_query,
        filter_categories: [],
      });
      const [result] = await getClient().query([{ start, end }], query);

      const context = buildActivityContext({
        events: result.events || [],
        domainEvents: result.browser_domains || [],
        trackedSeconds: result.tracked_duration || 0,
        start,
        end,
        hosts: [this.selectedHost],
        timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
        privacy: {
          excludeUncategorized: this.excludeUncategorized,
          privateCategories: this.excludePrivateCategories ? this.privateCategories : [],
        },
      });
      const text = formatActivityContext(context);
      if (browserFallbackUsed) {
        return (
          '注意：浏览器数据来自主机名无法解析（“unknown”）的存储桶，因此这些域名可能来自其他设备。\n' +
          text
        );
      }
      return text;
    },
    async copyResponse() {
      try {
        await navigator.clipboard.writeText(this.llmResponse);
        this.copied = true;
        setTimeout(() => {
          this.copied = false;
        }, 2000);
      } catch {
        this.error = '无法复制到剪贴板。';
      }
    },
  },
};
</script>
