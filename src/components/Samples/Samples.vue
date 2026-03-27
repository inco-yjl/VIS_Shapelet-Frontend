<template>
  <div class="samples-content">
    <div class="header-line">
      <div class="title">Test Samples Clustered By Predicted Class</div>
      <div class="meta">
        <span>Total Samples: {{ allSummaries.length }}</span>
        <span v-if="activeClusterKey"
          >Loaded In Cluster: {{ activeClusterLoadedCount }}</span
        >
        <span v-if="activeClusterDepthSummary">
          Soft Depth Range:
          {{ toFixedSafe(activeClusterDepthSummary.minDepth) }} ~
          {{ toFixedSafe(activeClusterDepthSummary.maxDepth) }}
        </span>
        <el-button
          type="primary"
          plain
          size="small"
          :disabled="!datasetName || summaryLoading"
          @click="loadAllSummaries"
        >
          Reload
        </el-button>
      </div>
    </div>

    <el-alert
      v-if="errorMessage"
      :title="errorMessage"
      type="warning"
      show-icon
      :closable="false"
      class="error-alert"
    />

    <el-skeleton :rows="6" animated v-if="summaryLoading" />

    <div v-else-if="clusterKeys.length" class="cluster-body">
      <el-tabs v-model="activeClusterKey" class="cluster-tabs">
        <el-tab-pane
          v-for="clusterKey in clusterKeys"
          :key="clusterKey"
          :name="String(clusterKey)"
          :label="`Pred Class ${clusterKey} (${clusterMap[clusterKey].samples.length})`"
        />
      </el-tabs>

      <div class="cluster-loading" v-if="clusterLoading">
        Loading and computing soft-depth profiles for this cluster...
      </div>

      <div class="content-line">
        <div ref="chartRef" class="cluster-chart"></div>

        <div class="sample-list">
          <div class="list-title">
            Samples In This Cluster (sorted by soft depth)
          </div>
          <div class="list-tools">
            <el-switch
              v-model="onlyMisclassified"
              inline-prompt
              active-text="pred!=true"
              inactive-text="all"
            />
            <div class="topk-control">
              <span>TopK</span>
              <el-input-number
                v-model="outerTopK"
                :min="1"
                :max="200"
                :step="1"
                size="small"
                controls-position="right"
              />
            </div>
            <span class="selected-count"
              >Selected: {{ selectedSampleIds.length }}</span
            >
          </div>
          <el-table
            ref="sampleTableRef"
            :data="displayClusterSamples"
            height="200"
            row-key="sample_id"
            highlight-current-row
            :reserve-selection="true"
            @selection-change="onTableSelectionChange"
            @row-click="onTableRowClick"
          >
            <el-table-column type="selection" width="45" />
            <el-table-column prop="sample_id" label="Sample" min-width="90" />
            <el-table-column prop="label" label="True" width="60" />
            <el-table-column label="Pred" width="60">
              <template #default="{ row }">
                {{ row.prediction?.pred_class ?? "N/A" }}
              </template>
            </el-table-column>
            <el-table-column label="Margin" width="86">
              <template #default="{ row }">
                {{ toFixedSafe(row.prediction?.margin) }}
              </template>
            </el-table-column>
            <el-table-column label="Depth" width="86">
              <template #default="{ row }">
                {{ toFixedSafe(row.soft_depth) }}
              </template>
            </el-table-column>
          </el-table>
        </div>
      </div>

      <div class="detail-line" v-if="selectedSamples.length">
        <span>Selected Count: {{ selectedSamples.length }}</span>
        <span
          >IDs: {{ selectedSamples.map((s) => s.sample_id).join(", ") }}</span
        >
      </div>
    </div>

    <div v-else class="empty-state">
      No sample data loaded for current dataset.
    </div>
  </div>
</template>

<script setup>
import axios from "@/scripts/axios.js";
import * as echarts from "echarts";
import {
  computed,
  defineProps,
  nextTick,
  onBeforeUnmount,
  ref,
  watch,
} from "vue";

const props = defineProps({
  datasetName: {
    type: String,
    default: "",
  },
  classDistribution: {
    type: Object,
    default: () => ({}),
  },
});

const summaryLoading = ref(false);
const clusterLoading = ref(false);
const errorMessage = ref("");
const allSummaries = ref([]);
const clusterMap = ref({});
const clusterDetailCache = ref({});
const clusterDepthCache = ref({});
const activeClusterKey = ref("");
const selectedSampleIds = ref([]);
const onlyMisclassified = ref(false);

const chartRef = ref(null);
const sampleTableRef = ref(null);
let chartInstance = null;
let summaryLoadToken = 0;

const CENTRAL_RATIO = 0.5;
const BAND_MODE = "quantile";
const OUTER_CURVE_TOP_K_MAX = 200;
const outerTopK = ref(80);
let topKRenderDebounceTimer = null;

const classLabels = computed(() => {
  const labels = Object.keys(props.classDistribution || {}).map((v) =>
    Number(v)
  );
  return labels.filter((v) => !Number.isNaN(v)).sort((a, b) => a - b);
});

const clusterKeys = computed(() => {
  return Object.keys(clusterMap.value)
    .map((v) => Number(v))
    .sort((a, b) => a - b);
});

const activeClusterDepthState = computed(() => {
  const key = Number(activeClusterKey.value);
  if (Number.isNaN(key)) return null;
  return clusterDepthCache.value[key] || null;
});

const activeClusterDepthSummary = computed(() => {
  const state = activeClusterDepthState.value;
  if (!state) return null;
  return {
    minDepth: state.minDepth,
    maxDepth: state.maxDepth,
  };
});

const activeClusterSamples = computed(() => {
  const key = Number(activeClusterKey.value);
  if (Number.isNaN(key) || !clusterMap.value[key]) return [];
  const sortedSamples = activeClusterDepthState.value?.sortedSamples;
  if (sortedSamples) return sortedSamples;
  const baseSamples = clusterMap.value[key].samples || [];
  const depthBySampleId = activeClusterDepthState.value?.depthBySampleId || {};
  return [...baseSamples]
    .map((sample) => ({
      ...sample,
      soft_depth: depthBySampleId[sample.sample_id] ?? null,
    }))
    .sort((a, b) => {
      const da = typeof a.soft_depth === "number" ? a.soft_depth : -Infinity;
      const db = typeof b.soft_depth === "number" ? b.soft_depth : -Infinity;
      return db - da;
    });
});

const isMisclassified = (sample) => {
  const pred = sample?.prediction?.pred_class;
  const label = sample?.label;
  return (
    typeof pred === "number" && typeof label === "number" && pred !== label
  );
};

const displayClusterSamples = computed(() => {
  if (!onlyMisclassified.value) return activeClusterSamples.value;
  return activeClusterSamples.value.filter((sample) => isMisclassified(sample));
});

const selectedSamples = computed(() => {
  if (!selectedSampleIds.value.length) return [];
  const idSet = new Set(selectedSampleIds.value.map((id) => String(id)));
  return activeClusterSamples.value.filter((sample) =>
    idSet.has(String(sample.sample_id))
  );
});

const activeClusterLoadedCount = computed(() => {
  const key = Number(activeClusterKey.value);
  if (Number.isNaN(key)) return 0;
  const cache = clusterDetailCache.value[key];
  if (!cache?.detailsById) return 0;
  return Object.keys(cache.detailsById).length;
});

const toFixedSafe = (num) => {
  if (typeof num !== "number" || Number.isNaN(num)) return "N/A";
  return num.toFixed(3);
};

const normalizeSelectedIds = (ids) => {
  const uniq = new Set((ids || []).map((id) => String(id)).filter(Boolean));
  return [...uniq];
};

const normalizeSequence = (sequence) => {
  return (sequence || []).map((point) =>
    Array.isArray(point) ? Number(point[0]) : Number(point)
  );
};

const getClassSamplesOnePage = async (label, offset, limit) => {
  const url = `v1/part-a/datasets/${props.datasetName}/samples`;
  const response = await axios({
    method: "get",
    url,
    params: { label, offset, limit },
  });
  return response.data;
};

const getSampleDetail = async (sampleId) => {
  const url = `v1/part-a/datasets/${props.datasetName}/samples/${sampleId}`;
  const response = await axios({
    method: "get",
    url,
    params: { split: "test" },
  });
  return response.data;
};

const mapWithConcurrency = async (arr, concurrency, mapper) => {
  const results = new Array(arr.length);
  let cursor = 0;
  const worker = async () => {
    while (cursor < arr.length) {
      const i = cursor;
      cursor += 1;
      try {
        results[i] = await mapper(arr[i], i);
      } catch (error) {
        results[i] = null;
      }
    }
  };
  const count = Math.min(concurrency, arr.length);
  await Promise.all(new Array(count).fill(0).map(() => worker()));
  return results.filter(Boolean);
};

const getAllSampleSummaries = async (token) => {
  const pageLimit = 200;
  const allItems = [];
  for (const label of classLabels.value) {
    let offset = 0;
    let total = 0;
    do {
      if (token !== summaryLoadToken) return [];
      const page = await getClassSamplesOnePage(label, offset, pageLimit);
      const items = page.items || [];
      total = page.total || 0;
      allItems.push(...items);
      offset += pageLimit;
    } while (offset < total);
  }
  return allItems;
};

const buildClusterMapFromSummaries = (summaries) => {
  const grouped = {};
  summaries.forEach((item) => {
    const predClass = item?.prediction?.pred_class;
    if (predClass === undefined || predClass === null) return;
    if (!grouped[predClass]) grouped[predClass] = { samples: [] };
    grouped[predClass].samples.push(item);
  });
  return grouped;
};

const ensureChart = () => {
  if (!chartRef.value) return;
  if (chartInstance) return;
  chartInstance = echarts.init(chartRef.value);
  chartInstance.on("click", async (params) => {
    const seriesId = params?.seriesId || "";
    // Only data curves with prefix `sample-` are clickable sample lines.
    if (!seriesId.startsWith("sample-")) return;
    const sampleId = seriesId.slice(7);
    if (!sampleId) return;
    await toggleSampleSelection(sampleId);
  });
};

const renderEmptyChart = () => {
  ensureChart();
  if (!chartInstance) return;
  chartInstance.clear();
};

const trimSequencesToMatrix = (details) => {
  const parsed = details
    .map((detail) => ({
      sampleId: String(detail.sample_id),
      seq: normalizeSequence(detail.sequence),
    }))
    .filter(
      (item) => item.seq.length > 1 && item.seq.every((v) => Number.isFinite(v))
    );

  if (!parsed.length) return null;

  const minLen = parsed.reduce(
    (m, item) => Math.min(m, item.seq.length),
    parsed[0].seq.length
  );
  const sampleIds = parsed.map((item) => item.sampleId);
  const X = parsed.map((item) => item.seq.slice(0, minLen));
  return { sampleIds, X, T: minLen };
};

const mean1d = (arr) => arr.reduce((sum, v) => sum + v, 0) / arr.length;

const std1d = (arr, eps = 1e-8) => {
  const m = mean1d(arr);
  const variance =
    arr.reduce((sum, v) => sum + (v - m) * (v - m), 0) / arr.length;
  return Math.max(Math.sqrt(variance), eps);
};

const quantile1d = (arr, q) => {
  if (!arr.length) return 0;
  if (arr.length === 1) return arr[0];
  const sorted = [...arr].sort((a, b) => a - b);
  const pos = (sorted.length - 1) * q;
  const lo = Math.floor(pos);
  const hi = Math.ceil(pos);
  if (lo === hi) return sorted[lo];
  const frac = pos - lo;
  return sorted[lo] * (1 - frac) + sorted[hi] * frac;
};

const computeSoftDepthAgainstMean = (X, eps = 1e-8) => {
  const N = X.length;
  const T = X[0].length;

  const meanCurve = Array.from({ length: T }, (_, t) =>
    mean1d(X.map((row) => row[t]))
  );
  const sigmaT = Array.from({ length: T }, (_, t) =>
    std1d(
      X.map((row) => row[t]),
      eps
    )
  );

  const W = X.map((row) =>
    row.map((value, t) => {
      const diff = value - meanCurve[t];
      const denom = 2 * sigmaT[t] * sigmaT[t];
      return Math.exp(-(diff * diff) / denom);
    })
  );

  const depth = W.map((row) => mean1d(row));
  return { depth, meanCurve, sigmaT, W };
};

const getCentralBandByDepth = (
  X,
  depth,
  centralRatio = 0.5,
  bandMode = "quantile"
) => {
  const N = X.length;
  const T = X[0].length;
  const k = Math.max(1, Math.ceil(N * centralRatio));

  const orderHighToLow = Array.from({ length: N }, (_, i) => i).sort(
    (a, b) => depth[b] - depth[a]
  );
  const centralIdx = orderHighToLow.slice(0, k);
  const centralCurves = centralIdx.map((idx) => X[idx]);

  const lowerBand = [];
  const upperBand = [];

  for (let t = 0; t < T; t += 1) {
    const values = centralCurves.map((row) => row[t]);
    if (bandMode === "minmax") {
      lowerBand.push(Math.min(...values));
      upperBand.push(Math.max(...values));
    } else {
      lowerBand.push(quantile1d(values, 0.25));
      upperBand.push(quantile1d(values, 0.75));
    }
  }

  return { centralIdx, centralCurves, lowerBand, upperBand, orderHighToLow };
};

const buildDepthState = (clusterKey) => {
  const cache = clusterDetailCache.value[clusterKey];
  if (!cache?.loaded) return null;
  const details = Object.values(cache.detailsById || {}).filter(
    (detail) => detail?.sequence?.length
  );
  const matrixData = trimSequencesToMatrix(details);
  if (!matrixData) return null;

  const { sampleIds, X, T } = matrixData;
  const { depth, meanCurve, W } = computeSoftDepthAgainstMean(X);
  const orderLowToHigh = Array.from({ length: depth.length }, (_, i) => i).sort(
    (a, b) => depth[a] - depth[b]
  );
  const { centralIdx, lowerBand, upperBand, orderHighToLow } =
    getCentralBandByDepth(X, depth, CENTRAL_RATIO, BAND_MODE);

  const depthBySampleId = {};
  sampleIds.forEach((sampleId, idx) => {
    depthBySampleId[sampleId] = depth[idx];
  });

  const centralSet = new Set(centralIdx);
  const outerIndices = Array.from({ length: X.length }, (_, i) => i).filter(
    (idx) => !centralSet.has(idx)
  );
  const outerSortedIndices = outerIndices.sort((a, b) => depth[a] - depth[b]);

  const baseSamples = clusterMap.value[clusterKey]?.samples || [];
  const sampleById = {};
  baseSamples.forEach((sample) => {
    sampleById[String(sample.sample_id)] = sample;
  });
  const sortedSamples = sampleIds
    .map((sampleId) => {
      const base = sampleById[sampleId];
      if (!base) return null;
      return {
        ...base,
        soft_depth: depthBySampleId[sampleId] ?? null,
      };
    })
    .filter(Boolean)
    .sort((a, b) => {
      const da = typeof a.soft_depth === "number" ? a.soft_depth : -Infinity;
      const db = typeof b.soft_depth === "number" ? b.soft_depth : -Infinity;
      return db - da;
    });

  const WSorted = orderHighToLow.map((idx) => W[idx]);

  return {
    sampleIds,
    X,
    T,
    depth,
    meanCurve,
    W,
    WSorted,
    orderLowToHigh,
    orderHighToLow,
    centralIdx,
    lowerBand,
    upperBand,
    outerSortedIndices,
    minDepth: Math.min(...depth),
    maxDepth: Math.max(...depth),
    depthBySampleId,
    sortedSamples,
  };
};

const renderSoftDepthPanels = (clusterKey) => {
  ensureChart();
  if (!chartInstance) return;

  const state = clusterDepthCache.value[clusterKey];
  if (!state) {
    renderEmptyChart();
    return;
  }

  const {
    sampleIds,
    X,
    T,
    meanCurve,
    outerSortedIndices,
    lowerBand,
    upperBand,
  } = state;

  const xAxisData = Array.from({ length: T }, (_, i) => i);
  const selectedSet = new Set(selectedSampleIds.value.map((id) => String(id)));

  const topKValue = Math.max(
    1,
    Math.min(OUTER_CURVE_TOP_K_MAX, Number(outerTopK.value) || 1)
  );
  const outerTopKIndices = outerSortedIndices.slice(0, topKValue);
  const panelSeries = outerTopKIndices.map((idx) => ({
    id: `sample-${sampleIds[idx]}`,
    name: `Outer Sample ${sampleIds[idx]}`,
    type: "line",
    xAxisIndex: 0,
    yAxisIndex: 0,
    data: X[idx],
    showSymbol: false,
    lineStyle: {
      width: 0.9,
      opacity: 0.4,
      color: "#d1d5db",
    },
    z: 1,
  }));

  panelSeries.push(
    {
      id: "p3-band-polygon",
      name: `Central band (top ${Math.ceil(CENTRAL_RATIO * 100)}%)`,
      type: "custom",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: [0],
      silent: true,
      tooltip: { show: false },
      z: 3,
      renderItem: (params, api) => {
        const points = [];
        for (let i = 0; i < xAxisData.length; i += 1) {
          points.push(api.coord([xAxisData[i], upperBand[i]]));
        }
        for (let i = xAxisData.length - 1; i >= 0; i -= 1) {
          points.push(api.coord([xAxisData[i], lowerBand[i]]));
        }
        return {
          type: "polygon",
          shape: { points },
          style: api.style({
            fill: "rgba(249, 115, 22, 0.24)",
            stroke: "none",
          }),
        };
      },
    },
    {
      id: "p3-band-lower-line",
      name: "Lower band",
      type: "line",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: lowerBand,
      showSymbol: false,
      lineStyle: {
        width: 2,
        color: "#f59e0b",
        opacity: 0.95,
      },
      z: 4,
    },
    {
      id: "p3-band-upper-line",
      name: "Upper band",
      type: "line",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: upperBand,
      showSymbol: false,
      lineStyle: {
        width: 2,
        color: "#f59e0b",
        opacity: 0.95,
      },
      z: 4,
    },
    {
      id: "p3-mean",
      name: "Mean curve",
      type: "line",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: meanCurve,
      showSymbol: false,
      lineStyle: {
        width: 3.2,
        color: "#111827",
      },
      z: 5,
    },
    {
      id: "p3-selected-anchor",
      name: "Selected anchor",
      type: "line",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: [],
      showSymbol: false,
      lineStyle: {
        width: 0,
        opacity: 0,
      },
      z: 6,
    }
  );

  const selectedPalette = [
    "#2563eb",
    "#1d4ed8",
    "#0ea5e9",
    "#0284c7",
    "#0891b2",
    "#3b82f6",
  ];
  const selectedSeries = sampleIds
    .map((sampleId, idx) => ({ sampleId, idx }))
    .filter((item) => selectedSet.has(item.sampleId))
    .map((item, i) => ({
      id: `selected-${item.sampleId}`,
      name: `Selected ${item.sampleId}`,
      type: "line",
      xAxisIndex: 0,
      yAxisIndex: 0,
      data: X[item.idx],
      showSymbol: false,
      lineStyle: {
        width: 2.8,
        color: selectedPalette[i % selectedPalette.length],
        opacity: 1,
      },
      z: 7,
    }));
  panelSeries.push(...selectedSeries);

  chartInstance.setOption(
    {
      animation: false,
      title: {
        text: `(Soft-Depth Panel)\nMean+Central Band +Selected+Top-${topKValue} Outer Curves`,
        left: 30,
        top: 6,
        textStyle: { fontSize: 12, fontWeight: 600, color: "#1f2937" },
      },
      grid: { left: 35, right: 20, top: 58, bottom: 30 },
      xAxis: [
        {
          type: "category",
          data: xAxisData,
          gridIndex: 0,
          boundaryGap: false,
          name: "Time",
          axisLabel: { fontSize: 10 },
        },
      ],
      yAxis: [
        {
          type: "value",
          gridIndex: 0,
          scale: true,
          name: "Value",
          axisLabel: { fontSize: 10 },
        },
      ],
      tooltip: {
        trigger: "axis",
        formatter: function (params) {
          let res = params[0].axisValue + "<br/>";

          params.forEach((p) => {
            let val = p.data;

            if (typeof val === "number") {
              val = val.toFixed(3);
            }

            res += `${p.marker}${p.seriesName}: ${val}<br/>`;
          });

          return res;
        },
      },
      series: panelSeries,
    },
    true
  );
};

const ensureClusterDetailsLoaded = async (clusterKey) => {
  const key = Number(clusterKey);
  if (Number.isNaN(key) || !clusterMap.value[key]) return;

  const existing = clusterDetailCache.value[key];
  if (existing?.loaded) return;
  if (existing?.loadingPromise) {
    await existing.loadingPromise;
    return;
  }

  const sampleIds = clusterMap.value[key].samples
    .map((s) => s.sample_id)
    .filter(Boolean);
  if (!sampleIds.length) {
    clusterDetailCache.value[key] = { loaded: true, detailsById: {} };
    return;
  }

  const loadingPromise = (async () => {
    const details = await mapWithConcurrency(sampleIds, 8, async (sampleId) =>
      getSampleDetail(sampleId)
    );
    const detailsById = {};
    details.forEach((detail) => {
      if (!detail?.sample_id) return;
      detailsById[detail.sample_id] = detail;
    });
    clusterDetailCache.value[key] = {
      loaded: true,
      detailsById,
      loadingPromise: null,
    };
  })();

  clusterDetailCache.value[key] = {
    loaded: false,
    detailsById: existing?.detailsById || {},
    loadingPromise,
  };

  await loadingPromise;
};

const recomputeAndRenderCluster = async (clusterKey) => {
  const key = Number(clusterKey);
  if (Number.isNaN(key)) {
    renderEmptyChart();
    return;
  }
  const depthState = buildDepthState(key);
  if (!depthState) {
    clusterDepthCache.value[key] = null;
    renderEmptyChart();
    return;
  }
  clusterDepthCache.value[key] = depthState;

  renderSoftDepthPanels(key);
};

const syncTableSelection = async () => {
  if (!sampleTableRef.value) return;
  await nextTick();
  sampleTableRef.value.clearSelection();
  const selectedSet = new Set(selectedSampleIds.value.map((id) => String(id)));
  displayClusterSamples.value.forEach((row) => {
    if (selectedSet.has(String(row.sample_id))) {
      sampleTableRef.value.toggleRowSelection(row, true);
    }
  });
};

const toggleSampleSelection = async (sampleId) => {
  if (!sampleId) return;
  const id = String(sampleId);
  const selectedSet = new Set(selectedSampleIds.value.map((v) => String(v)));
  if (selectedSet.has(id)) {
    selectedSet.delete(id);
  } else {
    selectedSet.add(id);
  }
  selectedSampleIds.value = [...selectedSet];
  await syncTableSelection();
  const key = Number(activeClusterKey.value);
  if (!Number.isNaN(key) && clusterDepthCache.value[key]) {
    renderSoftDepthPanels(key);
  }
};

const onTableSelectionChange = (rows) => {
  selectedSampleIds.value = normalizeSelectedIds(
    rows.map((row) => row.sample_id)
  );
  const key = Number(activeClusterKey.value);
  if (!Number.isNaN(key) && clusterDepthCache.value[key]) {
    renderSoftDepthPanels(key);
  }
};

const onTableRowClick = async (row) => {
  if (!row?.sample_id) return;
  await toggleSampleSelection(row.sample_id);
};

const loadAllSummaries = async () => {
  if (!props.datasetName || !classLabels.value.length) {
    allSummaries.value = [];
    clusterMap.value = {};
    clusterDetailCache.value = {};
    clusterDepthCache.value = {};
    activeClusterKey.value = "";
    selectedSampleIds.value = [];
    renderEmptyChart();
    return;
  }

  const token = summaryLoadToken + 1;
  summaryLoadToken = token;
  summaryLoading.value = true;
  clusterLoading.value = false;
  errorMessage.value = "";
  try {
    const summaries = await getAllSampleSummaries(token);
    if (token !== summaryLoadToken) return;

    const idSet = new Set();
    allSummaries.value = summaries.filter((s) => {
      const id = s?.sample_id;
      if (!id || idSet.has(id)) return false;
      idSet.add(id);
      return true;
    });

    clusterMap.value = buildClusterMapFromSummaries(allSummaries.value);
    clusterDetailCache.value = {};
    clusterDepthCache.value = {};
    selectedSampleIds.value = [];
    const keys = Object.keys(clusterMap.value)
      .map((v) => Number(v))
      .sort((a, b) => a - b);
    activeClusterKey.value = keys.length ? String(keys[0]) : "";
    await nextTick();
    if (!keys.length) renderEmptyChart();
  } catch (error) {
    console.log("error", error);
    errorMessage.value = "Failed to load test-set sample summaries.";
  } finally {
    if (token === summaryLoadToken) summaryLoading.value = false;
  }
};

const loadAndRenderActiveCluster = async () => {
  const key = Number(activeClusterKey.value);
  if (Number.isNaN(key)) {
    renderEmptyChart();
    return;
  }

  clusterLoading.value = true;
  errorMessage.value = "";
  try {
    await ensureClusterDetailsLoaded(key);
    await recomputeAndRenderCluster(key);
  } catch (error) {
    console.log("error", error);
    errorMessage.value =
      "Failed to load soft-depth visualization for the selected cluster.";
  } finally {
    clusterLoading.value = false;
  }
};

watch(
  () => [props.datasetName, classLabels.value.join(",")],
  () => {
    loadAllSummaries();
  },
  { immediate: true }
);

watch(
  () => activeClusterKey.value,
  async () => {
    selectedSampleIds.value = [];
    await nextTick();
    await loadAndRenderActiveCluster();
  }
);

watch(
  () => [
    onlyMisclassified.value,
    activeClusterKey.value,
    activeClusterSamples.value.length,
  ],
  async () => {
    await syncTableSelection();
  }
);

watch(
  () => outerTopK.value,
  () => {
    const normalized = Math.max(
      1,
      Math.min(OUTER_CURVE_TOP_K_MAX, Number(outerTopK.value) || 1)
    );
    if (normalized !== outerTopK.value) {
      outerTopK.value = normalized;
      return;
    }
    if (topKRenderDebounceTimer) clearTimeout(topKRenderDebounceTimer);
    topKRenderDebounceTimer = setTimeout(() => {
      const key = Number(activeClusterKey.value);
      if (!Number.isNaN(key) && clusterDepthCache.value[key]) {
        renderSoftDepthPanels(key);
      }
    }, 250);
  }
);

onBeforeUnmount(() => {
  if (topKRenderDebounceTimer) clearTimeout(topKRenderDebounceTimer);
  if (chartInstance) {
    chartInstance.dispose();
    chartInstance = null;
  }
});
</script>

<style scoped lang="scss">
.samples-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding-right: 16px;

  .header-line {
    display: flex;
    justify-content: space-between;
    align-items: center;
    min-height: 34px;
    .title {
      font-size: 16px;
      font-weight: 600;
    }
    .meta {
      display: flex;
      align-items: center;
      gap: 12px;
      font-size: 13px;
      color: #4b5563;
      flex-wrap: wrap;
      justify-content: flex-end;
    }
  }

  .error-alert {
    margin-top: 8px;
  }

  .cluster-body {
    margin-top: 10px;
    flex: 1;
    display: flex;
    flex-direction: column;
  }

  .cluster-loading {
    margin-bottom: 8px;
    font-size: 13px;
    color: #6b7280;
  }

  .content-line {
    display: flex;
    gap: 10px;
    flex: 1;
    min-height: 0;
  }

  .cluster-chart {
    flex: 1;
    min-width: 0;
    height: 280px;
    border: 1px solid #eaecef;
    border-radius: 6px;
  }

  .sample-list {
    width: 310px;
    height: 260px;
    border: 1px solid #eaecef;
    border-radius: 6px;
    padding: 8px;
    .list-title {
      font-size: 13px;
      font-weight: 600;
      margin-bottom: 6px;
    }
    .list-tools {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      margin-bottom: 8px;
      .topk-control {
        display: flex;
        align-items: center;
        gap: 6px;
        font-size: 12px;
      }
      .selected-count {
        font-size: 12px;
        color: #6b7280;
      }
    }
  }

  .detail-line {
    margin-top: 8px;
    display: flex;
    gap: 16px;
    flex-wrap: wrap;
    font-size: 13px;
    color: #374151;
  }

  .empty-state {
    margin-top: 10px;
    font-size: 13px;
    color: #9ca3af;
  }
}
</style>
