<template>
  <div class="column content-box">
    <woot-button
      color-scheme="success"
      class-names="button--fixed-top"
      icon="arrow-download"
      @click="downloadAgentReports"
    >
      {{ $t('REPORT.DOWNLOAD_AGENT_REPORTS') }}
    </woot-button>
    <report-filter-selector
      :show-agents-filter="false"
      :show-group-by-filter="true"
      @filter-change="onFilterChange"
    />
    <div class="row">
      <woot-report-stats-card
        v-for="(metric, index) in metrics"
        :key="metric.NAME"
        :desc="metric.DESC"
        :heading="metric.NAME"
        :info-text="displayInfoText(metric.KEY)"
        :index="index"
        :on-click="changeSelection"
        :point="displayMetric(metric.KEY)"
        :trend="calculateTrend(metric.KEY)"
        :selected="index === currentSelection"
      />
    </div>
    <div class="report-bar">
      <woot-loading-state
        v-if="accountReport.isFetching"
        :message="$t('REPORT.LOADING_CHART')"
      />
      <div v-else class="chart-container">
        <woot-bar
          v-if="accountReport.data.length"
          :collection="collection"
          :chart-options="chartOptions"
        />
        <span v-else class="empty-state">
          {{ $t('REPORT.NO_ENOUGH_DATA') }}
        </span>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import fromUnixTime from 'date-fns/fromUnixTime';
import format from 'date-fns/format';
import ReportFilterSelector from './components/FilterSelector';
import { GROUP_BY_FILTER, METRIC_CHART } from './constants';
import reportMixin from 'dashboard/mixins/reportMixin';
import alertMixin from 'shared/mixins/alertMixin';
import { formatTime } from '@chatwoot/utils';
import { REPORTS_EVENTS } from '../../../../helper/AnalyticsHelper/events';

const REPORTS_KEYS = {
  CONVERSATIONS: 'conversations_count',
  INCOMING_MESSAGES: 'incoming_messages_count',
  OUTGOING_MESSAGES: 'outgoing_messages_count',
  FIRST_RESPONSE_TIME: 'avg_first_response_time',
  RESOLUTION_TIME: 'avg_resolution_time',
  RESOLUTION_COUNT: 'resolutions_count',
};

export default {
  name: 'ConversationReports',
  components: {
    ReportFilterSelector,
  },
  mixins: [reportMixin, alertMixin],
  data() {
    return {
      from: 0,
      to: 0,
      currentSelection: 0,
      groupBy: GROUP_BY_FILTER[1],
      businessHours: false,
    };
  },
  computed: {
    ...mapGetters({
      accountSummary: 'getAccountSummary',
      accountReport: 'getAccountReports',
    }),
    collection() {
      if (this.accountReport.isFetching) {
        return {};
      }
      if (!this.accountReport.data.length) return {};
      const labels = this.accountReport.data.map(element => {
        if (this.groupBy.period === GROUP_BY_FILTER[2].period) {
          let week_date = new Date(fromUnixTime(element.timestamp));
          const first_day = week_date.getDate() - week_date.getDay();
          const last_day = first_day + 6;

          const week_first_date = new Date(week_date.setDate(first_day));
          const week_last_date = new Date(week_date.setDate(last_day));

          return `${format(week_first_date, 'dd/MM/yy')} - ${format(
            week_last_date,
            'dd/MM/yy'
          )}`;
        }
        if (this.groupBy.period === GROUP_BY_FILTER[3].period) {
          return format(fromUnixTime(element.timestamp), 'MMM-yyyy');
        }
        if (this.groupBy.period === GROUP_BY_FILTER[4].period) {
          return format(fromUnixTime(element.timestamp), 'yyyy');
        }
        return format(fromUnixTime(element.timestamp), 'dd-MMM-yyyy');
      });

      const datasets = METRIC_CHART[
        this.metrics[this.currentSelection].KEY
      ].datasets.map(dataset => {
        switch (dataset.type) {
          case 'bar':
            return {
              ...dataset,
              yAxisID: 'y-left',
              label: this.metrics[this.currentSelection].NAME,
              data: this.accountReport.data.map(element => element.value),
            };
          case 'line':
            return {
              ...dataset,
              yAxisID: 'y-right',
              label: this.metrics[0].NAME,
              data: this.accountReport.data.map(element => element.count),
            };
          default:
            return dataset;
        }
      });

      return {
        labels,
        datasets,
      };
    },
    chartOptions() {
      let tooltips = {};
      if (this.isAverageMetricType(this.metrics[this.currentSelection].KEY)) {
        tooltips.callbacks = {
          label: tooltipItem => {
            return this.$t(this.metrics[this.currentSelection].TOOLTIP_TEXT, {
              metricValue: formatTime(tooltipItem.yLabel),
              conversationCount: this.accountReport.data[tooltipItem.index]
                .count,
            });
          },
        };
      }

      return {
        scales: METRIC_CHART[this.metrics[this.currentSelection].KEY].scales,
        tooltips: tooltips,
      };
    },
    metrics() {
      const reportKeys = [
        'CONVERSATIONS',
        'INCOMING_MESSAGES',
        'OUTGOING_MESSAGES',
        'FIRST_RESPONSE_TIME',
        'RESOLUTION_TIME',
        'RESOLUTION_COUNT',
      ];
      const infoText = {
        FIRST_RESPONSE_TIME: this.$t(
          `REPORT.METRICS.FIRST_RESPONSE_TIME.INFO_TEXT`
        ),
        RESOLUTION_TIME: this.$t(`REPORT.METRICS.RESOLUTION_TIME.INFO_TEXT`),
      };
      return reportKeys.map(key => ({
        NAME: this.$t(`REPORT.METRICS.${key}.NAME`),
        KEY: REPORTS_KEYS[key],
        DESC: this.$t(`REPORT.METRICS.${key}.DESC`),
        INFO_TEXT: infoText[key],
        TOOLTIP_TEXT: `REPORT.METRICS.${key}.TOOLTIP_TEXT`,
      }));
    },
  },
  methods: {
    fetchAllData() {
      this.fetchAccountSummary();
      this.fetchChartData();
    },
    fetchAccountSummary() {
      try {
        this.$store.dispatch('fetchAccountSummary', this.getRequestPayload());
      } catch {
        this.showAlert(this.$t('REPORT.SUMMARY_FETCHING_FAILED'));
      }
    },
    fetchChartData() {
      try {
        this.$store.dispatch('fetchAccountReport', {
          metric: this.metrics[this.currentSelection].KEY,
          ...this.getRequestPayload(),
        });
      } catch {
        this.showAlert(this.$t('REPORT.DATA_FETCHING_FAILED'));
      }
    },
    getRequestPayload() {
      const { from, to, groupBy, businessHours } = this;

      return {
        from,
        to,
        groupBy: groupBy.period,
        businessHours,
      };
    },
    downloadAgentReports() {
      const { from, to } = this;
      const fileName = `agent-report-${format(
        fromUnixTime(to),
        'dd-MM-yyyy'
      )}.csv`;
      this.$store.dispatch('downloadAgentReports', { from, to, fileName });
    },
    changeSelection(index) {
      this.currentSelection = index;
      this.fetchChartData();
    },
    onFilterChange({ from, to, groupBy, businessHours }) {
      this.from = from;
      this.to = to;
      this.groupBy = groupBy;
      this.businessHours = businessHours;
      this.fetchAllData();

      this.$track(REPORTS_EVENTS.FILTER_REPORT, {
        filterValue: { from, to, groupBy, businessHours },
        reportType: 'conversations',
      });
    },
  },
};
</script>
