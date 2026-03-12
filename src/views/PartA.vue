<template>
  <div class="full-page">
    <div class="info-leftbar">
      <el-card class="box-card">
        <template #header>
          <div class="one-line">
            <div class="dataset-selector">
              <div class="selector-title">Dataset:</div>
              <el-select
                v-model="datasetName"
                placeholder="Select"
                style="width: 240px"
              >
                <el-option
                  v-for="item in dataSetList"
                  :key="item.dataset"
                  :label="item.display_name"
                  :value="item.dataset"
                />
              </el-select>
            </div>
            <el-button
              type="primary"
              :disabled="!datasetName"
              @click="getDataInfo"
              >Confirm</el-button
            >
          </div>
        </template>
        <el-skeleton :rows="5" v-if="meta_loading" />
        <div class="metadata-panel" v-else-if="metadata">
          <!-- Dataset Info -->
          <div class="metadata-section">
            <div class="info-title">Dataset Info</div>
            <ul v-if="metadata.dataset_meta">
              <li v-for="(value, key) in metadata.dataset_meta" :key="key">
                <span class="meta-key">{{ formatKey(key) }}: </span>
                <span class="meta-value">{{ value }}</span>
              </li>
            </ul>
          </div>

          <!-- Training Configuration -->
          <div class="metadata-section">
            <div class="info-title">Training Configuration</div>
            <ul v-if="metadata.training_meta">
              <li v-for="(value, key) in metadata.training_meta" :key="key">
                <span class="meta-key">{{ formatKey(key) }}: </span>
                <span class="meta-value">{{ value }}</span>
              </li>
            </ul>
          </div>

          <!-- Warnings -->
        </div>
        <el-skeleton :rows="5" v-if="metrics_loading" />
        <div class="metadata-panel" v-else-if="metrics">
          <div class="metadata-section">
            <div class="info-title">Model Performance</div>
            <ul v-if="metrics.test_metrics">
              <li v-for="(value, key) in metrics.test_metrics" :key="key">
                <span class="meta-key">{{ formatKey(key) }}: </span>
                <span class="meta-value">{{
                  value ? value.toFixed(3) : "N/A"
                }}</span>
              </li>
            </ul>
          </div>
          <div class="metadata-section">
            <div class="info-title">Data Summary</div>
            <ul>
              <li>
                <span class="meta-key">Sample Count: </span>
                <span class="meta-value">{{ metrics.sample_count }}</span>
              </li>
            </ul>
            <div
              ref="distributionChart"
              style="width: 360px; height: 150px"
            ></div>
          </div>
        </div>
        <el-collapse
          v-if="metadata?.warnings?.length"
          v-model="activeNames"
          @change="handleChange"
        >
          <el-collapse-item title="⚠ Warnings" name="1">
            <ul>
              <li v-for="(warning, index) in metadata.warnings" :key="index">
                <div class="warning-message">{{ warning.message }}</div>
              </li>
            </ul>
          </el-collapse-item>
        </el-collapse>
      </el-card>
    </div>
    <div class="partA-content">
      <cluster-content v-model:datasetName="datasetName" />
      <div class="sliders" v-if="metrics">
        <div class="margin-slider">
          <div class="slider-title">Threshold:</div>
          <div class="slider">
            <el-slider
              show-input
              v-model="margin_threshold"
              :min="0"
              :max="0.9"
              :step="0.1"
            />
          </div>
        </div>
        <div class="offset-slider">
          <div class="slider-title">Offset:&nbsp;</div>
          <div class="slider">
            <el-slider
              v-model="offset"
              show-input
              :min="0"
              :step="10"
              :max="metrics.sample_count - limit"
            />
          </div>
        </div>
        <div class="limit-slider">
          <div class="slider-title">Limit:</div>
          <div class="slider">
            <el-slider
              v-model="limit"
              show-input
              :min="1"
              :step="1"
              :max="Math.min(metrics.sample_count - offset, 200)"
            />
          </div>
        </div>
        <div class="confirm-button">
          <el-button
            type="primary"
            :disabled="!datasetName"
            @click="getBottomData"
            >Confirm</el-button
          >
        </div>
      </div>
      <div class="partA-bottom" v-if="showBottom">
        <samples-content />
        <margin-content
          :datasetName="datasetName"
          :threshold="margin_threshold"
          :offset="offset"
          :limit="limit"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import axios from "@/scripts/axios.js";
import ClusterContent from "@/components/Cluster";
import MarginContent from "@/components/Margin";
import SamplesContent from "@/components/Samples";
import { onMounted, ref, nextTick, computed } from "vue";
import { useRouter } from "vue-router";
import * as echarts from "echarts";
const router = useRouter();
const dataSetList = ref([]);
const datasetName = ref("mcce");
const metadata = ref();
const metrics = ref();

const meta_loading = ref(false);
const metrics_loading = ref(false);

const activeNames = ref([]);
const margin_threshold = ref(0.1);
const distributionChart = ref();
const offset = ref(0);
const limit = ref(50);
const MAX_LIMIT = 200;
const showBottom = ref(false);
const getBottomData = () => {
  showBottom.value = false;
  showBottom.value = true;
};
const getDatasetList = async () => {
  try {
    let response = await axios({
      method: "get",
      url: "v1/part-a/datasets",
    });
    console.log(response);
    dataSetList.value = response.data.datasets;
  } catch (error) {
    console.log("error", error);
  }
};
const getDataInfo = () => {
  localStorage.setItem("shapeletDataset", datasetName.value);
  getMetadata();
  getDatametrics();
};
const getMetadata = async () => {
  meta_loading.value = true;
  metadata.value = null;
  const url = "v1/part-a/datasets/" + datasetName.value + "/meta";
  try {
    let response = await axios({
      method: "get",
      url,
    });
    metadata.value = response.data;
    meta_loading.value = false;
    console.log(metadata.value);
  } catch (error) {
    console.log("error", error);
  }
};
const formatKey = (key) => {
  switch (key) {
    case "acc":
      key = "accuracy";
      break;
    case "f1":
      key = "F1 Score";
      break;
    case "auc":
      key = "AUC";
      break;
    default:
      break;
  }
  return key.replace(/_/g, " ").replace(/\b\w/g, (char) => char.toUpperCase());
};
const getDatametrics = async () => {
  metrics_loading.value = true;
  metrics.value = null;
  const url = "v1/part-a/datasets/" + datasetName.value + "/metrics";
  try {
    let response = await axios({
      method: "get",
      url,
      params: {
        margin_threshold: margin_threshold.value,
      },
    });
    console.log(response);
    metrics_loading.value = false;
    metrics.value = response.data;
    nextTick(() => {
      drawDistributionChart();
    });
  } catch (error) {
    console.log("error", error);
  }
};
const drawDistributionChart = () => {
  const chart = echarts.init(distributionChart.value);
  const classDistribution = metrics.value.class_distribution;
  const labels = Object.keys(classDistribution);
  const values = Object.values(classDistribution);
  let barWidth = 70 / Object.keys(metrics.value.class_distribution).length;
  if (barWidth > 40) barWidth = 40;
  const option = {
    tooltip: {},
    grid: { left: 50, right: 40, top: 0, bottom: 30 },
    xAxis: {
      type: "value",
      splitLine: {
        show: false,
      },
      axisLine: {
        show: false,
      },
      axisTick: {
        show: false,
      },
    },
    yAxis: {
      type: "category",
      data: labels.map((v) => `Class ${v}`),
      axisLine: {
        show: false,
      },
      axisTick: {
        show: false,
      },
    },
    series: [
      {
        type: "bar",
        barWidth,
        data: values,
      },
    ],
  };
  chart.setOption(option);
};
onMounted(() => {
  getDatasetList();
});
</script>
<style lang="scss">
.info-leftbar {
  .el-collapse-item__title {
    color: rgb(179, 23, 23);
  }
}
</style>
<style lang="scss" scoped>
.full-page {
  width: 100%;
  height: 100%;
  display: flex;
  .partA-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    .sliders {
      height: 20px;
      width: 100%;
      padding: 8px 20px;
      display: flex;
      align-items: center;
      .confirm-button {
        display: flex;
        align-items: center;
        justify-content: center;
      }
      .el-slider {
        margin-top: 0;
        margin-left: 12px;
      }
      .slider-title {
        font-size: 16px;
      }
      .offset-slider {
        display: flex;
        align-items: center;
        .slider {
          width: 350px;
          display: flex;
          align-items: center;
          margin-right: 20px;
        }
      }
      .limit-slider {
        display: flex;
        align-items: center;

        .slider {
          width: 300px;
          display: flex;
          align-items: center;
          margin-right: 20px;
        }
      }
      .margin-slider {
        display: flex;
        align-items: center;

        .slider {
          width: 250px;
          display: flex;
          align-items: center;
          margin-right: 20px;
        }
      }
    }
    .partA-bottom {
      flex: 1;
      display: flex;

      padding: 8px 20px;
    }
  }
}
.info-leftbar {
  max-width: 400px;
  min-width: 400px;
  font-size: 20px;
  position: relative;
  padding-left: 16px;
  padding-right: 16px;
  padding-top: 1vh;
  height: 99vh;
  overflow-y: auto; /* 当内容过多时y轴出现滚动条 */
  background-color: #e0eeee55;

  .metadata-panel {
    display: flex;
    flex-direction: column;
    gap: 10px;

    .metadata-section {
      display: flex;
      flex-direction: column;
      .info-title {
        font-size: 20px;
        font-weight: bold;
      }

      ul {
        font-size: 16px;
        line-height: 24px;
      }
    }
  }

  .box-card {
    max-height: 90vh;
    overflow: auto;
  }
  .one-line {
    display: flex;
    .slider-name {
      font-size: 16px;
    }
  }

  .dataset-selector {
    align-items: center;
    display: flex;
    gap: 8px;
    flex: 1;
    .selector-title {
      font-size: 16px;
    }
    .el-select {
      width: 150px !important;
    }
  }
}
</style>
