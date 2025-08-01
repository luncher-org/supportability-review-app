<script>
import { mapGetters } from 'vuex';
import { allHash } from '@shell/utils/promise';

import { Banner } from '@components/Banner';
import AsyncButton from '@shell/components/AsyncButton';
import Loading from '@shell/components/Loading';

import { CATALOG } from '@shell/config/types';
import { REPO_TYPE, REPO, CHART, VERSION, LOCAL } from '@shell/config/query-params';

import { SR_REPO, SR_CHARTS } from '../config/types';
import { handleGrowl, getLatestStableVersion } from '../utils/utils';

export default {
  name: 'InstallView',
  components: {
    AsyncButton,
    Banner,
    Loading
  },
  data() {
    return { reloadReady: false, initLoading: true, chartBranch: '' };
  },
  async fetch() {
    this.reloadReady = false;
    const reqs = {};

    if (this.$store.getters['management/canList'](CATALOG.CLUSTER_REPO)) {
      reqs.repos = this.$store.dispatch('management/findAll', {
        type: CATALOG.CLUSTER_REPO
      });
    }

    // force the refresh of the catalog to populate the charts
    if (!this.srRepo || !this.operatorChart) {
      reqs.catalogRefresh = this.$store.dispatch('catalog/refresh');
    }

    await allHash(reqs);
    this.initLoading = false;

    fetch('/rancherversion', {
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      }
    })
      .then((response) => {
        if (!response.ok) {
          throw new Error(`HTTP error! Status: ${response.status}`);
        }
        return response.json();
      })
      .then((data) => {
        const versionArray = data.Version.split('.');
        this.chartBranch = 'release-' + versionArray[0] + '.' + versionArray[1];
      })
      .catch((error) => {
        console.error('Fetch error:', error);
      });
  },
  computed: {
    ...mapGetters({
      charts: 'catalog/charts',
      repos: 'catalog/repos',
      t: 'i18n/t'
    }),

    srRepo() {
      const chart = this.charts?.find((chart) => chart.chartName === SR_CHARTS.OPERATOR);

      return this.repos?.find((repo) => repo.id === chart?.repoName);
    },

    operatorChart() {
      const chart =
        this.$store.getters['catalog/chart']({
          repoName: this.srRepo ? this.srRepo.id : null,
          repoType: 'cluster',
          chartName: SR_CHARTS.OPERATOR
        }) || null;
      return chart;
    }
  },
  methods: {
    async addRepository(btnCb) {
      try {
        const repoObj = await this.$store.dispatch('management/create', {
          type: CATALOG.CLUSTER_REPO,
          metadata: { name: 'supportability-review-operator-charts' },
          spec: {
            gitBranch: this.chartBranch,
            gitRepo: SR_REPO.REPO
          }
        });

        try {
          await repoObj.save();
        } catch (e) {
          handleGrowl({ error: e, store: this.$store });
          btnCb(false);

          return;
        }

        await this.refreshCharts();
        btnCb(true);
      } catch (e) {
        handleGrowl({ error: e, store: this.$store });
        btnCb(false);
      }
    },

    async refreshCharts(retry = 0, init) {
      try {
        await this.$store.dispatch('catalog/refresh');
      } catch (e) {
        handleGrowl({ error: e, store: this.$store });
      }

      if (!this.operatorChart && retry === 0) {
        await this.$store.dispatch('management/findAll', {
          type: CATALOG.CLUSTER_REPO
        });
        await this.refreshCharts(retry + 1);
      }

      if (!this.operatorChart && retry === 1 && !init) {
        this.reloadReady = true;
      }
    },

    async chartRoute() {
      if (!this.operatorChart) {
        try {
          await this.refreshCharts();
        } catch (e) {
          handleGrowl({ error: e, store: this.$store });

          return;
        }
      }

      const { repoType, repoName, chartName, versions } = this.operatorChart;

      const latestStableVersion = getLatestStableVersion(versions);

      if (latestStableVersion) {
        const query = {
          [REPO_TYPE]: repoType,
          [REPO]: repoName,
          [CHART]: chartName,
          [VERSION]: latestStableVersion.version
        };

        this.$router.push({
          name: 'c-cluster-apps-charts-install',
          params: { cluster: LOCAL },
          query
        });
      } else {
        const error = {
          _statusText: this.t('sr.versionError.title'),
          message: this.t('sr.versionError.message')
        };

        handleGrowl({ error, store: this.$store });
      }
    },

    reload() {
      this.$router.go();
    }
  }
};
</script>

<template>
  <div class="not-installed p-10">
    <div class="sr-logo mt-20 mb-10">
      <svg
        version="1.0"
        xmlns="http://www.w3.org/2000/svg"
        width="48"
        height="48"
        viewBox="0 0 345.000000 345.000000"
        preserveAspectRatio="xMidYMid meet">
        <g transform="translate(0.000000,345.000000) scale(0.100000,-0.100000)" stroke="none">
          <path
            d="M1073 2966 c-23 -28 -48 -155 -35 -168 13 -13 223 -22 303 -14 93 10 93 10 74 97 -24 107 -64 133 -137 89 -41 -25 -65 -25 -103 0 -41 27 -79 26
-102 -4z" />
          <path
            d="M810 2770 c-29 -55 25 -120 119 -143 51 -12 51 -12 51 -52 0 -22 10 -60 21 -85 42 -93 117 -144 214 -145 121 0 212 79 238 206 l13 64 41 7 c106
18 174 89 138 145 -20 29 -30 29 -180 -11 -85 -23 -121 -27 -235 -27 -115 0 -150 4 -235 27 -145 39 -171 41 -185 14z" />
          <path
            d="M2378 2539 c-3 -8 -48 -11 -138 -10 l-135 2 -27 -43 c-29 -45 -28 -62 3 -53 18 6 19 1 19 -70 l0 -76 48 3 c47 3 47 3 50 41 l3 37 90 0 89 0 0 -27 c0 -45 12 -55 61 -51 43 3 44 4 47 41 2 28 8 37 22 37 21 0 30 19 30 66 0 29 3 34 24 34 31 0 37 10 30 49 -7 34 -24 42 -24 12 0 -28 -18 -36 -39 -17
-12 11 -36 16 -74 16 -31 0 -57 5 -57 10 0 14 -17 13 -22 -1z" />
          <path
            d="M1016 2290 c-26 -10 -50 -21 -52 -23 -6 -6 233 -167 247 -167 13 0 249 152 249 161 0 6 -61 34 -96 44 -12 3 -51 -2 -88 -10 -59 -14 -72 -14 -125
0 -73 19 -75 18 -135 -5z" />
          <path
            d="M842 2118 c-154 -291 -144 -247 -112 -459 12 -74 25 -151 31 -171 10 -37 59 -88 85 -88 12 0 14 -62 14 -411 0 -264 4 -417 10 -430 10 -19 5 -19 -225 -19 l-236 0 3 83 3 82 174 3 c196 3 229 13 185 56 -13 13 -45 16 -190 16 l-175 0 3 100 3 101 178 2 c145 2 179 5 187 17 13 21 12 26 -6 45 -14 13 -42 
16 -188 15 l-171 -2 -2 101 -1 101 173 0 c144 0 176 3 189 16 44 43 11 53 -185 56 l-174 3 -3 50 c-4 62 -25 81 -56 49 -14 -14 -16 -69 -16 -455 l0 -439 -68 0 c-79 0 -108 -15 -98 -48 l7 -22 1560 0 c1435 0 1560 1 1566 16 3 9 2 24 -4 33 -8 13 -33 17 -137 19 l-126 3 0 437 c0 380 -2 440 -16 460 -35 50 -104
12 -104 -58 l0 -40 -382 -2 -383 -3 0 -30 0 -30 385 -3 385 -2 -3 -105 -3 -105 -649 0 c-645 0 -649 0 -655 -20 -16 -51 -31 -50 664 -50 l646 0 -3 -100 -3 -100 -638 0 c-675 0 -678 0 -669 -46 3 -19 20 -19 655 -22 l652 -2 3 -90 3 -90 -702 0 -703 0 10 26 c6 15 10 233 10 550 l0 524 96 -48 c81 -40 94 -50 90
-67 -3 -11 -6 -57 -6 -102 l0 -82 -46 -3 c-54 -3 -77 -26 -55 -52 9 -11 31 -16 71 -16 l57 0 18 -41 c35 -76 115 -120 180 -99 46 15 105 84 130 151 29 78 27 232 -4 311 -30 76 -88 134 -144 143 -23 3 -87 27 -142 52 -114 52 -88 16 -201 283 -31 74 -60 139 -64 143 -4 5 -17 3 -29 -5 -21 -13 -21 -18 -21 -311
l0 -298 -112 4 c-62 3 -129 8 -149 12 -23 4 -39 3 -43 -4 -5 -7 -49 -11 -121 -11 l-115 0 -2 312 c-2 224 -6 312 -15 315 -6 2 -38 -47 -71 -109z m1137 -498 c39 -54 55 -125 48 -219 -7 -114 -61 -211 -117 -211 -27 0 -68 38 -92 83 -19 35 -22 59 -23 146 l0 103 32 3 c18 1 45 10 59 20 29 19 57 76 48 99 -12 31 16
15 45 -24z m-815 -105 c25 -19 60 -19 80 0 11 11 40 15 120 15 l106 0 0 -469 c0 -567 12 -521 -141 -521 l-99 0 0 338 c0 250 -3 341 -12 350 -16 16 -23 15
-42 -4 -14 -13 -16 -59 -16 -350 l0 -334 -95 0 c-148 0 -135 -50 -135 515 l0
475 108 0 c78 -1 112 -5 126 -15z" />
          <path
            d="M2862 2113 c-5 -10 -12 -41 -16 -69 -8 -61 1 -71 64 -77 33 -3 29 -4 -15 -3 l-60 1 0 45 c0 25 -4 51 -8 58 -6 9 -70 12 -261 12 l-253 0 -53 -81 c-48 -73 -52 -81 -37 -96 14 -14 18 -14 46 6 l30 22 3 -153 3 -153 91 -3 c105 -3 104 -4 104 96 l0 52 185 0 185 0 0 -62 c0 -35 4 -69 8 -76 6 -9 33 -12 98 -10 l89 3 3 68 3 67 -44 0 c-47 0 -67 13 -66 45 0 14 2 15 6 5 11 -30 37 -40
103 -40 96 0 100 6 100 158 0 96 -3 124 -16 136 -12 13 -40 16 -140 16 l-124 0 0 25 c0 29 -15 33 -28 8z" />
          <path
            d="M3247 2123 c-4 -3 -7 -20 -7 -38 0 -35 -15 -55 -42 -55 -13 0 -18 -8 -18 -30 0 -29 2 -30 44 -30 58 0 70 16 62 80 -10 74 -20 93 -39 73z" />
        </g>
      </svg>
    </div>
    <h1 class="mb-20">
      {{ t('product.sr') }}
    </h1>
    <p v-clean-html="t('product.description', {}, true)" class="description" data-testid="sr-description-text" />
    <Banner class="mt-20 mb-40" color="warning" data-testid="warning-not-install-or-no-schema">
      <div v-clean-html="t('product.notInstalledOrNoSchema', {}, true)" />
    </Banner>

    <div class="relative">
      <Loading v-if="initLoading" mode="relative" class="mt-20" />

      <!-- Install charts -->
      <div v-else class="relative">
        <div v-if="!operatorChart && !reloadReady" mode="relative" class="mt-20">
          <AsyncButton mode="srRepository" @click="addRepository" />
        </div>

        <div v-else-if="!operatorChart && reloadReady">
          <Banner color="warning">
            <span class="mb-20">
              {{ t('sr.appInstall.reload') }}
            </span>
            <button class="ml-10 btn btn-sm role-primary" data-testid="install-reload-btn" @click="reload()">
              {{ t('generic.reload') }}
            </button>
          </Banner>
        </div>

        <button
          v-else
          data-testid="charts-install-button"
          class="btn role-primary mt-20"
          :disabled="!operatorChart"
          @click.prevent="chartRoute">
          {{ t('sr.appInstall.button') }}
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
body.theme-dark .sr-logo svg {
  fill: var(--body-text);
}
</style>

<style lang="scss" scoped>
.not-installed {
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  margin: 100px 0;

  .sr-logo svg {
    transform: scale(1.5);
  }

  .description {
    line-height: 20px;
  }
}

.relative {
  position: relative;
}
</style>
