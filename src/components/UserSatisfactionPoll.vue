<template lang="pug">
  div
    b-alert(v-if="isPollVisible", variant="info", show)
      button(type="button", class="close", @click="isPollVisible=false") &times;
      form
        p
          | 你已经使用 ActivityWatch 一段时间了。按 1–10 分评价，你有多大可能把它推荐给朋友或同事？10 分表示非常愿意推荐。
        div(class="radio-options")
          div(v-for="i in options", class="option-group")
            input(type="radio", :id="'option' + i", name="rating", :value="i", v-model="rating")
            br
            label(:for="'option' + i", style="display: block")
              | {{ i }}
      div(style="display: flex; justify-content: space-between")
        a(@click="dontShowAgain" href="#")
          | 不再显示
        input(type="submit" value="提交" @click="submit")

    b-alert(v-if="isPosFollowUpVisible", variant="info" show)
      button(type="button", class="close", @click="isPosFollowUpVisible=false") &times;
      p
        | 很高兴你喜欢 ActivityWatch，我们还想继续把它做得更好。
        br
        | 如果愿意支持项目，可以考虑下面这些方式：
      ul.small
        li
          | 通过 #[a(href="https://www.patreon.com/erikbjare") Patreon]、#[a(href="https://opencollective.com/activitywatch") Open Collective] 或 #[a(href="https://activitywatch.net/donate/") 其他捐赠方式]支持项目。
        li
          | 把 ActivityWatch 推荐给朋友和同事。
        li
          | 在社交媒体分享，我们在 #[a(href="https://twitter.com/ActivityWatchIt") Twitter] 和 #[a(href="https://www.facebook.com/ActivityWatch") Facebook] 上都有账号。
        li
          | 在 #[a(href="https://alternativeto.net/software/activitywatch/about/") AlternativeTo] 和 #[a(href="https://play.google.com/store/apps/details?id=net.activitywatch.android") Google Play] 上评分。
        li
          | 加入我们的 #[a(href="https://discord.gg/vDskV9q") Discord] 社区。
        li
          | 订阅 #[a(href="http://eepurl.com/cTU6QX") 邮件简报]（发送频率很低）。

    b-alert(v-if="isNegFollowUpVisible", variant="info" show)
      button(type="button", class="close", @click="isNegFollowUpVisible=false") &times;
      | 很遗憾这次体验没有让你满意。如果愿意帮助我们改进，可以：
      ul
        li
          | 填写 #[a(href="https://forms.gle/q2N9K5RoERBV8kqPA") 反馈表]。
        li
          | 在 #[a(href="https://forum.activitywatch.net/c/features") 论坛] 为你希望加入的功能投票。
</template>

<style scoped>
.radio-options {
  display: flex;
  justify-content: space-around;
}

.option-group {
  text-align: center;
}

ul {
  margin: 0;
}
</style>

<script lang="ts">
import { range } from 'lodash/fp';
import moment from 'moment';

import { useSettingsStore } from '~/stores/settings';

const NUM_OPTIONS = 10;
// BACKOFF_PERIOD is how many seconds to wait to show the poll again if the user closed it
const BACKOFF_PERIOD = 7 * 24 * 60 * 60;
// The following may be used for testing
// const INITIAL_WAIT_PERIOD = 1;
// const BACKOFF_PERIOD = 1;

export default {
  name: 'user-satisfaction-poll',
  data() {
    return {
      isPollVisible: false,
      isPosFollowUpVisible: false,
      isNegFollowUpVisible: false,
      // options is an array of [1, ..., NUM_OPTIONS]
      options: range(1, NUM_OPTIONS + 1),
      rating: null,
    };
  },
  computed: {
    data: {
      get() {
        const settingsStore = useSettingsStore();
        return settingsStore.userSatisfactionPollData;
      },
      set(value) {
        const settingsStore = useSettingsStore();
        const data = settingsStore.userSatisfactionPollData;
        settingsStore.update({
          userSatisfactionPollData: { ...data, ...value },
        });
      },
    },
  },
  async mounted() {
    if (!this.data.isEnabled) {
      return;
    }

    if (moment() >= moment(this.data.nextPollTime)) {
      this.data.timesPollIsShown = this.data.timesPollIsShown + 1;
      this.isPollVisible = true;
      this.data.nextPollTime = moment().add(BACKOFF_PERIOD, 'seconds');
    }

    if (this.data.timesPollIsShown > 2) {
      this.data.isEnabled = false;
    }
  },
  methods: {
    submit() {
      this.isPollVisible = false;
      this.data = { ...this.data, isEnabled: false };

      if (parseInt(this.rating) >= 6) {
        this.isPosFollowUpVisible = true;
      } else {
        this.isNegFollowUpVisible = true;
      }
    },
    dontShowAgain() {
      this.isPollVisible = false;
      this.data = { ...this.data, isEnabled: false };
    },
  },
};
</script>
