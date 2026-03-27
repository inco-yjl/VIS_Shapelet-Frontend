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
          <div class="cluster-checkbox">
            Clusters:&nbsp;&nbsp;
            <el-checkbox-group
              v-model="activeClusters"
              @change="selectClusters"
            >
              <el-checkbox
                v-for="id in allClusters"
                :label="id"
                :value="id"
                :key="id"
              />
            </el-checkbox-group>
          </div>
          <el-checkbox-group v-model="activeTypes" @change="selectTypes">
            <el-checkbox
              v-for="type in ['median', 'range', 'centroid']"
              :label="formatKey(type)"
              :value="type"
              :key="type"
            />
          </el-checkbox-group>
        </div>
        <div
          v-if="clusters"
          ref="chart"
          style="width: 100%; height: 360px"
        ></div
      ></el-tab-pane>

      <el-tab-pane label="Details" name="second">
        <div class="cluster-selector">
          <el-select
            v-model="chosenId"
            filterable
            style="width: 220px"
            placeholder="Search Cluster"
            @change="chooseCluster"
          >
            <el-option
              v-for="id in allClusters"
              :label="'Cluster ' + id"
              :value="id"
              :key="id"
            />
          </el-select>
        </div>
        <div class="cluster-info">
          <b>Size:</b>&nbsp;{{ chosenCluster?.size }} |&nbsp;<b
            >Samples:&nbsp;</b
          >
          {{ chosenCluster?.sample_ids_preview.join() }}
        </div>
        <div ref="detailChart" style="width: 100%; height: 340px"></div>
      </el-tab-pane>
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
const detailChart = ref(null);
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
const chosenCluster = ref();
const chosenId = ref();
const chooseCluster = () => {
  const index = clusters.value.findIndex(
    (cluster) => cluster.cluster_id === chosenId.value
  );
  chosenCluster.value = clusters.value[index];
  drawDetailCluster();
};
const selectClusters = (val) => {
  let mychart = echarts.getInstanceByDom(chart.value);
  if (oldClusters.value.length > val.length) {
    let id = oldClusters.value.filter((id) => !val.includes(id))[0];
    console.log(id);
    let names = summarySeries.value
      .filter((s) => s.name.includes(" " + id + " "))
      .map((s) => s.name);
    names.forEach((name) => {
      mychart.dispatchAction({
        type: "legendUnSelect",
        name,
      });
    });
  } else {
    let id = val.filter((id) => !oldClusters.value.includes(id))[0];
    let names = [];
    activeTypes.value.forEach((type) => {
      names.push("Cluster " + id + " " + type);
    });

    names.forEach((name) => {
      mychart.dispatchAction({
        type: "legendSelect",
        name,
      });
    });
  }
  oldClusters.value = val;
};
const selectTypes = (val) => {
  let mychart = echarts.getInstanceByDom(chart.value);
  if (oldTypes.value.length > val.length) {
    let id = oldTypes.value.filter((id) => !val.includes(id))[0];
    console.log(id);
    let names = summarySeries.value
      .filter((s) => s.name.includes(id))
      .map((s) => s.name);
    names.forEach((name) => {
      mychart.dispatchAction({
        type: "legendUnSelect",
        name,
      });
    });
  } else {
    let type = val.filter((type) => !oldTypes.value.includes(type))[0];

    let names = [];
    activeClusters.value.forEach((id) => {
      names.push("Cluster " + id + " " + type);
    });
    names.forEach((name) => {
      mychart.dispatchAction({
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
    oldTypes.value = activeTypes.value = ["median", "range", "centroid"];
    nextTick(() => {
      drawClusters();
    });
  } catch (error) {
    console.log("error", error);
  }
};
const drawDetailCluster = () => {
  if (!chosenCluster.value) return;

  const mychart = echarts.init(detailChart.value);

  const cluster = chosenCluster.value;

  const x = cluster.median_sequence.map((_, i) => i);
  const centroid = cluster.centroid_sequence.map((d) => d[0]);
  const median = cluster.median_sequence.map((d) => d[0]);
  const q25 = cluster.q25_sequence.map((d) => d[0]);
  const q75 = cluster.q75_sequence.map((d) => d[0]);

  const baseColor = cmaps[0];

  const series = [
    buildRangePolygonSeries("Range", x, q25, q75, baseColor + "22"),
    {
      name: "Median",
      type: "line",
      data: median.map((y, i) => [x[i], y]),
      lineStyle: {
        color: baseColor,
        width: 3,
      },
      itemStyle: { opacity: 0 },
      showSymbol: false,
    },
    {
      name: "Median",
      type: "line",
      data: median.map((y, i) => [x[i], y]),
      lineStyle: {
        color: baseColor,
        width: 3,
      },
      itemStyle: { opacity: 0 },
      showSymbol: false,
    },
    {
      name: "Centroid",
      type: "line",
      data: centroid.map((y, i) => [x[i], y]),
      lineStyle: {
        color: "#ff6b6b", //contrast
        width: 3,
      },
      itemStyle: { opacity: 0 },
      showSymbol: false,
    },
  ];

  const option = {
    grid: { left: 10, right: 10, top: 50, bottom: 20 },

    tooltip: {
      trigger: "axis",
      formatter: function (params) {
        return params
          .map((p) => `${p.seriesName}: ${p.data[1].toFixed(3)}`)
          .join("<br/>");
      },
    },

    legend: {
      show: true,
      top: 10,
      left: "center",
    },

    xAxis: {
      type: "value",
      name: "Time step",
    },

    yAxis: {
      type: "value",
      name: "Value",
      scale: true,
    },

    series,
  };

  mychart.setOption(option);
};
const buildRangePolygonSeries = (
  name,
  xValues,
  lowerBand,
  upperBand,
  fillColor
) => ({
  name,
  type: "custom",
  data: [0],
  silent: true,
  tooltip: { show: false },
  renderItem: (params, api) => {
    const points = [];
    for (let i = 0; i < xValues.length; i += 1) {
      points.push(api.coord([xValues[i], upperBand[i]]));
    }
    for (let i = xValues.length - 1; i >= 0; i -= 1) {
      points.push(api.coord([xValues[i], lowerBand[i]]));
    }
    return {
      type: "polygon",
      shape: { points },
      style: api.style({
        fill: fillColor,
        stroke: "none",
      }),
    };
  },
});

const drawClusters = () => {
  const mychart = echarts.init(chart.value);
  let series = clusters.value.flatMap((cluster) => {
    const x = cluster.median_sequence.map((_, i) => i);
    const centroid = cluster.centroid_sequence.map((d) => d[0]);
    const median = cluster.median_sequence.map((d) => d[0]);
    const q25 = cluster.q25_sequence.map((d) => d[0]);
    const q75 = cluster.q75_sequence.map((d) => d[0]);

    return [
      buildRangePolygonSeries(
        `Cluster ${cluster.cluster_id} range`,
        x,
        q25,
        q75,
        cmaps[cluster.cluster_id % cmaps.length] + "33"
      ),
      {
        name: `Cluster ${cluster.cluster_id} median`,
        type: "line",
        data: median.map((y, i) => [x[i], y]),
        lineStyle: {
          color: cmaps[cluster.cluster_id % cmaps.length] + "66",
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
          color: cmaps[cluster.cluster_id % cmaps.length],
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
        console.log(params);
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
    series,
  };
  summarySeries.value = series;
  mychart.setOption(option);
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
  .cluster-checkbox {
    display: flex;
    align-items: center;
  }
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

.cluster-info {
  display: flex;
  align-items: center;
  margin-top: 8px;
  line-height: 32px;
  font-size: 16px;
}
</style>
