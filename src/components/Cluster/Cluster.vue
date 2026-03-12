<template>
  <div class="cluster-content">
    <div class="one-line">
      <div class="slider-title">Cluster Count:</div>
      <div class="cluster-slider">
        <el-slider show-input v-model="clusterK" :min="1" :max="20" :step="1" />
      </div>
      <el-button type="primary" :disabled="!datasetName" @click="getClusters"
        >Confirm</el-button
      >
    </div>
    <el-tabs v-model="activeName" class="cluster-tabs" @tab-click="handleClick">
      <el-tab-pane label="Summary" name="first" v-if="clusters">
        <div class="checkboxes">
          <el-checkbox-group v-model="activeClusters" @change="selectClusters">
            <el-checkbox
              v-for="id in allClusters"
              :label="'Cluster ' + id"
              :value="id"
              :key="id"
            />
          </el-checkbox-group>
          <el-checkbox-group v-model="activeTypes" @change="selectTypes">
            <el-checkbox
              v-for="type in ['median', 'range', 'centroid']"
              :label="formatKey(type)"
              :value="type"
              :key="type"
            />
          </el-checkbox-group>
        </div>
        <div v-if="clusters" ref="chart" style="width: 100%; height: 380px"></div
      ></el-tab-pane>

      <el-tab-pane label="Details" name="second">Details</el-tab-pane>
    </el-tabs>
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
});
const datasetName = computed({
  get: () => props.datasetName,
  set: (val) => {},
});
const chart = ref(null);
const summaryChart = ref();
const summarySeries = ref();
const clusterK = ref(4);
const clusters = ref();
const cmaps = [
  "#3182bd", //blue
  "#ff7f0e", //orange
  "#2ca02c", //green
  "#ff9896", //pale red
  "#9467bd", //purple
  "#8c564b", //brown
  "#e377c2", //pink
  "#c7c7c7", //gray
  "#bcbd22", //yellow green
  "#17becf", //light blue
  "#ff6c4b", // single outcome
  "#aec7e8", //single factor
];
const activeName = ref("first");
const activeClusters = ref();
const oldClusters = ref();
const allClusters = ref();
const activeTypes = ref(["median", "range", "centroid"]);
const oldTypes = ref(["median", "range", "centroid"]);
const formatKey = (key) => {
  return key.replace(/_/g, " ").replace(/\b\w/g, (char) => char.toUpperCase());
};
const selectClusters = (val) => {
  if (oldClusters.value.length > val.length) {
    let id = oldClusters.value.filter((id) => !val.includes(id))[0];
    console.log(id);
    let names = summarySeries.value
      .filter((s) => s.name.includes(" " + id + " "))
      .map((s) => s.name);
    names.forEach((name) => {
      summaryChart.value.dispatchAction({
        type: "legendUnSelect",
        name,
      });
    });
  } else {
    let id = val.filter((id) => !oldClusters.value.includes(id))[0];
    let names = summarySeries.value
      .filter((s) => s.name.includes(" " + id + " "))
      .map((s) => s.name);
    names.forEach((name) => {
      summaryChart.value.dispatchAction({
        type: "legendSelect",
        name,
      });
    });
  }
  oldClusters.value = val;
};
const selectTypes = (val) => {
  if (oldTypes.value.length > val.length) {
    let id = oldTypes.value.filter((id) => !val.includes(id))[0];
    console.log(id);
    let names = summarySeries.value
      .filter((s) => s.name.includes(id))
      .map((s) => s.name);
    names.forEach((name) => {
      summaryChart.value.dispatchAction({
        type: "legendUnSelect",
        name,
      });
    });
  } else {
    let id = val.filter((id) => !oldTypes.value.includes(id))[0];
    let names = summarySeries.value
      .filter((s) => s.name.includes(id))
      .map((s) => s.name);
    names.forEach((name) => {
      summaryChart.value.dispatchAction({
        type: "legendSelect",
        name,
      });
    });
  }
  oldTypes.value = val;
};
const getClusters = async () => {
  clusters.value = null;
  const url = "v1/part-a/datasets/" + datasetName.value + "/clusters";
  try {
    let response = await axios({
      method: "get",
      url,
      params: {
        cluster_k: clusterK.value,
      },
    });
    clusters.value = response.data.train_cluster_profiles;
    console.log(clusters.value);
    allClusters.value = clusters.value.flatMap((cluster) => {
      return cluster.cluster_id;
    });
    oldClusters.value = activeClusters.value = allClusters.value;
    nextTick(() => {
      drawClusters();
    });
  } catch (error) {
    console.log("error", error);
  }
};
const drawClusters = () => {
  summaryChart.value = echarts.init(chart.value);
  summarySeries.value = clusters.value.flatMap((cluster) => {
    const x = cluster.median_sequence.map((_, i) => i);
    const centroid = cluster.centroid_sequence.map((d) => d[0]);
    const median = cluster.median_sequence.map((d) => d[0]);
    const q25 = cluster.q25_sequence.map((d) => d[0]);
    const q75 = cluster.q75_sequence.map((d) => d[0]);

    const shadowData = q75.concat(q25.slice().reverse()).map((y, i) => [i, y]);

    return [
      {
        name: `Cluster ${cluster.cluster_id} range`,
        type: "line",
        data: shadowData,
        lineStyle: { opacity: 0 },
        areaStyle: { color: cmaps[cluster.cluster_id % cmaps.length] + "33" },
        showSymbol: false,
      },

      {
        name: `Cluster ${cluster.cluster_id} median`,
        type: "line",
        data: median.map((y, i) => [x[i], y]),
        lineStyle: {
          color: cmaps[cluster.cluster_id % cmaps.length],
          width: 2,
        },
        itemStyle: { opacity: 0 },
        showSymbol: false,
      },
      {
        name: `Cluster ${cluster.cluster_id} centroid`,
        type: "line",
        data: centroid.map((y, i) => [x[i], y]),
        lineStyle: {
          color: cmaps[cluster.cluster_id % cmaps.length] - 66,
          width: 2,
          type: "dashed",
        },
        itemStyle: { opacity: 0 },
        showSymbol: false,
      },
    ];
  });
  const option = {
    grid: { left: 10, right: 10, top: 60, bottom: 0 },
    tooltip: {
      trigger: "axis",
      formatter: function (params) {
        return params
          .map((p) => {
            return `${p.seriesName}: ${p.data[1].toFixed(3)}`;
          })
          .join("<br/>");
      },
    },
    legend: {
      show: true,
      top: 10,
      type: "scroll",
      left: "center",
      textStyle: { fontSize: 12 },
    },
    xAxis: { type: "value", name: "Time step" },
    yAxis: { type: "value", name: "Value", scale: true },
    series: summarySeries.value,
  };
  console.log(summarySeries.value);
  summaryChart.value.setOption(option);
};
</script>

<style lang="scss" scoped>
.cluster-content {
  width: calc(100% - 40px);
  padding: 20px;
  height: 500px;
}
.checkboxes {
  display: flex;
  justify-content: space-between;
}
.one-line {
  display: flex;
  align-items: center;
  width: 100%;
  .slider-title {
    font-size: 16px;
  }
  .cluster-slider {
    width: 450px;
    display: flex;
    align-items: center;
    margin-right: 20px;
  }
  .el-slider {
    margin-top: 0;
    margin-left: 12px;
  }
}
</style>
