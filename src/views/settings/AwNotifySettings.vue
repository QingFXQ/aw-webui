<template lang="pug">
div
  div.d-flex.justify-content-between.align-items-center.mb-3
    div
      h5.mb-1 活动提醒
      small.text-muted 配置 Android 与桌面端共用的 aw-notify 提醒
    b-btn(@click="save" size="sm" variant="primary" :disabled="saving || loading")
      | {{ saving ? '保存中…' : '保存' }}

  b-alert(v-if="error" show variant="danger") {{ error }}
  b-alert(v-if="success" show variant="success" dismissible @dismissed="success = false") 设置已保存。

  div(v-if="loading")
    b-spinner(small) 加载中…

  div(v-else)
    p.text-muted.small.mb-3
      | aw-notify 会定期检查这些提醒。累计时间跨过设定阈值时会触发通知。
      | 同一份配置可用于 Android 和 aw-tauri。

    div(v-if="alerts.length === 0")
      p.text-muted.font-italic 暂未配置提醒。

    b-card.mb-2(v-for="(alert, idx) in alerts" :key="idx")
      div.d-flex.align-items-start
        div.flex-grow-1
          b-form-group(label="名称" label-cols-sm="3" label-size="sm")
            b-input(v-model="alert.label" size="sm" placeholder="例如：工作")
          b-form-group(label="分类" label-cols-sm="3" label-size="sm")
            b-input(
              v-model="alert.category"
              size="sm"
              placeholder="All"
            )
            small.form-text.text-muted
              | 与 ActivityWatch 分类规则中的分类名称匹配；使用 All 表示全部时间。
          b-form-group(
            label="阈值"
            label-cols-sm="3"
            label-size="sm"
            :invalid-feedback="thresholdError(alert.thresholdStr)"
            :state="thresholdState(alert.thresholdStr)"
          )
            b-input(
              v-model="alert.thresholdStr"
              size="sm"
              placeholder="例如：60, 120, 240"
              :state="thresholdState(alert.thresholdStr)"
            )
            small.form-text.text-muted 使用英文逗号分隔的正整数分钟数；每跨过一个阈值就触发一次通知。
          b-form-group(label="类型" label-cols-sm="3" label-size="sm")
            b-form-radio-group(v-model="alert.positive" :options="goalOptions" size="sm")
        b-btn.ml-2(@click="removeAlert(idx)" variant="outline-danger" size="sm" title="删除提醒")
          icon(name="trash")

    b-btn.mt-1(@click="addAlert" variant="outline-secondary" size="sm")
      icon(name="plus")
      |  添加提醒
</template>

<script lang="ts">
import 'vue-awesome/icons/plus';
import 'vue-awesome/icons/trash';

import { getClient } from '~/util/awclient';
import {
  AwNotifyAlert,
  AwNotifyConfig,
  parseAwNotifyConfig,
  parseThresholds,
} from '~/util/aw-notify';

const SETTINGS_KEY = 'aw-notify';

interface AlertRow {
  label: string;
  category: string;
  thresholdStr: string;
  positive: boolean;
}

function dtoToRow(dto: AwNotifyAlert): AlertRow {
  return {
    label: dto.label ?? '',
    category: dto.category,
    thresholdStr: dto.thresholds_minutes.join(', '),
    positive: dto.positive,
  };
}

function rowToDto(row: AlertRow): AwNotifyAlert {
  const thresholds = parseThresholds(row.thresholdStr);
  if (!thresholds) {
    throw new Error('阈值必须是使用英文逗号分隔的正整数分钟数。');
  }
  return {
    label: row.label.trim() || null,
    category: row.category.trim() || 'All',
    thresholds_minutes: thresholds,
    positive: row.positive,
  };
}

export default {
  name: 'AwNotifySettings',
  data() {
    return {
      alerts: [] as AlertRow[],
      config: {} as AwNotifyConfig,
      loading: false,
      saving: false,
      error: '',
      success: false,
      goalOptions: [
        { text: '警告（超过限制）', value: false },
        { text: '目标（达到目标）', value: true },
      ],
    };
  },
  async mounted() {
    await this.load();
  },
  methods: {
    async load() {
      this.loading = true;
      this.error = '';
      try {
        const client = getClient();
        const resp = await client.req.get(`/0/settings/${SETTINGS_KEY}`);
        // aw-server-rust answers 200 with an empty body (null) for a key that was
        // never set, rather than 404; treat both the same.
        if (resp.data === null || resp.data === undefined || resp.data === '') {
          this.useDefaults();
          return;
        }
        const config = parseAwNotifyConfig(resp.data);
        if (!config) {
          throw new Error('已保存的 aw-notify 设置格式暂不受支持。');
        }
        this.config = config;
        this.alerts = config.alerts.map(dtoToRow);
      } catch (e: any) {
        if (e?.response?.status === 404) {
          this.useDefaults();
        } else {
          this.error = `加载设置失败：${e?.message ?? e}`;
        }
      } finally {
        this.loading = false;
      }
    },
    useDefaults() {
      this.config = {} as AwNotifyConfig;
      this.alerts = this.defaultAlerts();
    },
    async save() {
      this.error = '';
      this.success = false;
      if (this.alerts.some(row => parseThresholds(row.thresholdStr) === null)) {
        this.error = '阈值必须是使用英文逗号分隔的正整数分钟数。';
        return;
      }
      this.saving = true;
      try {
        const client = getClient();
        const payload: AwNotifyConfig = { ...this.config, alerts: this.alerts.map(rowToDto) };
        await client.req.post(`/0/settings/${SETTINGS_KEY}`, payload, {
          headers: { 'Content-Type': 'application/json' },
        });
        this.success = true;
      } catch (e: any) {
        this.error = `保存设置失败：${e?.message ?? e}`;
      } finally {
        this.saving = false;
      }
    },
    thresholdError(value: string): string {
      return parseThresholds(value) === null ? '请输入用英文逗号分隔的正整数分钟数。' : '';
    },
    thresholdState(value: string): boolean | null {
      return parseThresholds(value) === null ? false : null;
    },
    addAlert() {
      this.alerts.push({
        label: '',
        category: 'All',
        thresholdStr: '60, 120',
        positive: false,
      });
    },
    removeAlert(idx: number) {
      this.alerts.splice(idx, 1);
    },
    defaultAlerts(): AlertRow[] {
      return [
        { label: '全部', category: 'All', thresholdStr: '60, 240, 480', positive: false },
        { label: '💼 工作', category: 'Work', thresholdStr: '60, 120, 240', positive: true },
      ];
    },
  },
};
</script>
