<template>
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
      <div class="gallery">
        <div v-for="item in items" class="card" :key="item.shapelet_id">
          <div :id="'chart-' + item.shapelet_id"></div>
        </div>
      </div>
    </el-card>
    <div class="router-buttons">
      <el-button @click="toPartA" type="primary"
        >Show The Data Panel</el-button
      >
    </div>
  </div>
</template>

<script setup>
import axios from "@/scripts/axios.js";
import { onMounted, ref } from "vue";
import { useRouter } from "vue-router";
import * as echarts from "echarts";
const router = useRouter();
const dataSetList = ref([]);
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
const toPartA = () => {
  router.push({name: "DataPanel"})
}
onMounted(() => {
  getDatasetList();
});
</script>
<style scoped lang="scss">
.info-leftbar {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
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
  .router-buttons {
    display: flex;
    justify-content: end;
    height: 5vh;
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
