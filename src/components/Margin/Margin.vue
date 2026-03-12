<template>
  <div class="margin-content">
    <div class="info-line">
      <div class="one-line">
        <div class="info-title">Low-Margin Sample Count:</div>
        <div class="info-value">{{ total }}</div>
      </div>
      <div class="one-line">
        <div class="info-title">Sample ID:</div>
        <div class="sample-selector">
          <el-select
            v-model="selectedSampleId"
            filterable
            style="width: 220px"
            placeholder="Search sample"
            @change="drawSample"
          >
            <el-option
              v-for="item in samples"
              :key="item.sample_id"
              :label="`Sample ${item.sample_id} (margin: ${item.margin.toFixed(
                3
              )})`"
              :value="item.sample_id"
            />
          </el-select>
        </div>
      </div>
    </div>

    <div ref="chartRef" style="width: calc(100% - 40px); height: 280px"></div>
  </div>
</template>
<script setup>
import axios from "@/scripts/axios.js";
import * as echarts from "echarts";
import { onMounted, ref, defineProps, computed, nextTick } from "vue";
const props = defineProps({
  datasetName: {
    type: String,
  },
  threshold: {
    type: Number,
  },
  offset: {
    type: Number,
  },
  limit: {
    type: Number,
  },
});
const total = ref();
const samples = ref([]);
const selectedSampleId = ref(null);
const chartRef = ref();
const datasetName = props.datasetName;
const threshold = props.threshold;
const offset = props.offset;
const limit = props.limit;
const getLowMarginSamples = async () => {
  const url = "v1/part-a/datasets/" + datasetName + "/samples/low-margin";
  try {
    let response = await axios({
      method: "get",
      url,
    });
    total.value = response.data.total;
    samples.value = response.data.items;
  } catch (error) {
    console.log("error", error);
  }
};
const drawSample = () => {
  const chart = echarts.init(chartRef.value);
  const sample = samples.value.find(
    (s) => s.sample_id === selectedSampleId.value
  );

  if (!sample) return;

  const sequence = sample.sequence.map((d) => d[0]);

  const option = {
    grid: { left: 10, right: 10, top: 20, bottom: 20 },
    title: {
      text: `Sample ${sample.sample_id}`,
      subtext: `label=${sample.label}  pred=${
        sample.pred_class
      }  margin=${sample.margin.toFixed(3)}`,
    },

    tooltip: {
      trigger: "axis",
      formatter: (params) => {
        const p = params[0];
        return `t=${p.axisValue}<br/>value=${p.value.toFixed(3)}`;
      },
    },

    xAxis: {
      type: "category",
      data: sequence.map((_, i) => i),
    },

    yAxis: {
      type: "value",
      scale: true,
    },

    series: [
      {
        name: "sequence",
        type: "line",
        data: sequence,
        showSymbol: false,
        smooth: false,
      },
    ],
  };

  chart.setOption(option);
};
onMounted(() => {
  getLowMarginSamples();
});
</script>

<style lang="scss" scoped>
.margin-content {
  flex: 1;
  .info-line {
    display: flex;
    justify-content: space-between;
  }
  .one-line {
    display: flex;
    gap: 10px;
    align-items: center;
    .info-title {
      font-size: 16px;
    }
    .info-value {
      font-size: 16px;
    }
  }
}
</style>
