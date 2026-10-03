<template lang="pug">
div
  div
    b-alert(v-if="invalidDaterange", variant="warning", show)
      | 选择的日期范围无效，结束日期必须晚于或等于开始日期。
    b-alert(v-if="daterangeTooLong", variant="warning", show)
      | 选择的日期范围过长，最多可选择 {{ maxDuration/(24*60*60) }} 天。

  div.input-time-interval.d-flex.flex-wrap.align-items-start.justify-content-between
    // Two-row grid: labels share a fixed-width column so the Mode toggle
    // and the Range controls line up vertically and the secondary label
    // stays in the same spot (just "Range") regardless of which mode is
    // active. Previously the label flipped between "Quick range" / "Range"
    // and the inputs shifted horizontally on every toggle.
    div.time-interval-grid
      label.col-form-label.col-form-label-sm.mb-0(for="time-mode") 模式
      b-form-radio-group#time-mode(
        v-model="mode",
        @change="valueChanged",
        buttons,
        button-variant="outline-secondary",
        size="sm",
        :options="modeOptions"
      )

      label.col-form-label.col-form-label-sm.mb-0 范围
      div.d-flex.flex-wrap.align-items-center(v-if="mode == 'last_duration'")
        div.btn-group(role="group" aria-label="快速时长")
          template(v-for="(dur, idx) in durations")
            input(
              type="radio"
              :id="'dur' + idx"
              :value="dur.seconds"
              v-model="duration"
              @change="applyLastDuration"
            ).d-none
            label(:for="'dur' + idx" v-html="dur.label").btn.btn-light.btn-sm
      div.d-flex.flex-wrap.align-items-center(v-else)
        input.form-control.form-control-sm.mr-1(
          type="date", v-model="start", :max="end || undefined", style="width: auto"
          aria-label="开始日期"
        )
        input.form-control.form-control-sm.mr-1(
          type="date", v-model="end", :min="start || undefined", placeholder="（可选）", style="width: auto"
          aria-label="结束日期（可选）"
        )
        b-button(
          size="sm" variant="outline-dark"
          :disabled="invalidDaterange || emptyDaterange || daterangeTooLong"
          @click="applyRange"
        ) 应用

    div.text-right.d-none.d-md-block(v-if="showUpdate")
      b-button.px-2(@click="refresh()", variant="outline-dark", size="sm")
        icon.mr-1(name="sync")
        span.d-none.d-md-inline
          | 刷新
      div.mt-2.small.text-muted(v-if="lastUpdate")
        | 最近更新：#[time(:datetime="lastUpdate.format()") {{lastUpdate | friendlytime}}]
</template>

<style scoped lang="scss">
.input-time-interval {
  row-gap: 0.5rem;
}

.time-interval-grid {
  display: grid;
  // Fixed label column keeps Mode/Range labels aligned and the controls
  // start at the same X regardless of which mode is active.
  grid-template-columns: 4rem minmax(0, 1fr);
  column-gap: 0.75rem;
  row-gap: 0.5rem;
  align-items: center;
  min-width: 0;
}

.btn-group {
  input[type='radio']:checked + label {
    background-color: #495057;
    color: #fff;
    border-color: #495057;
  }
}

@media (max-width: 575.98px) {
  .time-interval-grid {
    grid-template-columns: 1fr;
  }

  .btn-group {
    flex-wrap: wrap;
  }
}
</style>

<script lang="ts">
import moment from 'moment';
import 'vue-awesome/icons/sync';
export default {
  name: 'input-timeinterval',
  props: {
    defaultDuration: {
      type: Number,
      default: 60 * 60,
    },
    maxDuration: {
      type: Number,
      default: null,
    },
    showUpdate: {
      type: Boolean,
      default: true,
    },
  },
  data() {
    return {
      duration: null,
      mode: 'last_duration',
      start: null,
      end: null,
      lastUpdate: null,
      durations: [
        { seconds: 0.25 * 60 * 60, label: '&frac14; 小时' },
        { seconds: 0.5 * 60 * 60, label: '&frac12; 小时' },
        { seconds: 60 * 60, label: '1 小时' },
        { seconds: 2 * 60 * 60, label: '2 小时' },
        { seconds: 3 * 60 * 60, label: '3 小时' },
        { seconds: 4 * 60 * 60, label: '4 小时' },
        { seconds: 6 * 60 * 60, label: '6 小时' },
        { seconds: 12 * 60 * 60, label: '12 小时' },
        { seconds: 24 * 60 * 60, label: '24 小时' },
        { seconds: 48 * 60 * 60, label: '48 小时' },
      ],
      modeOptions: [
        { text: '最近时长', value: 'last_duration' },
        { text: '日期范围', value: 'range' },
      ],
    };
  },
  computed: {
    value: {
      get() {
        if (this.mode == 'range' && this.start) {
          const startDate = moment(this.start);
          // If only start date is set, show that single day
          const endDate = this.end
            ? moment(this.end).add(1, 'day')
            : startDate.clone().add(1, 'day');
          return [startDate, endDate];
        } else {
          return [moment().subtract(this.duration, 'seconds'), moment()];
        }
      },
    },
    emptyDaterange() {
      return !this.start;
    },
    invalidDaterange() {
      if (!this.end) return false;
      return moment(this.start) > moment(this.end);
    },
    daterangeTooLong() {
      if (!this.end) return false;
      return moment(this.start).add(this.maxDuration, 'seconds').isBefore(moment(this.end));
    },
  },
  mounted() {
    this.duration = this.defaultDuration;
    this.valueChanged();

    // We want our lastUpdated text to update every ~500ms
    // We can do this by setting it to null and then the previous value.
    this.lastUpdateTimer = setInterval(() => {
      const _lastUpdate = this.lastUpdate;
      this.lastUpdate = null;
      this.lastUpdate = _lastUpdate;
    }, 500);
  },
  beforeDestroy() {
    clearInterval(this.lastUpdateTimer);
  },
  methods: {
    valueChanged() {
      if (
        this.mode == 'last_duration' ||
        (!this.emptyDaterange && !this.invalidDaterange && !this.daterangeTooLong)
      ) {
        this.lastUpdate = moment();
        this.$emit('input', this.value);
      }
    },
    refresh() {
      const tmpMode = this.mode;
      this.mode = '';
      this.mode = tmpMode;
      this.valueChanged();
    },
    applyRange() {
      this.mode = 'range';
      this.duration = 0;
      this.valueChanged();
    },
    applyLastDuration() {
      this.mode = 'last_duration';
      this.valueChanged();
    },
  },
};
</script>
