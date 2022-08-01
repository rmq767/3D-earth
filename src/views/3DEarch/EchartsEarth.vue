<template>
  <div class="earth-container">
    <div class="handle-container">
      <el-form :inline="true">
        <el-form-item label="切换3D/2D地图">
          <el-switch
            v-model="config.is3D"
            :active-value="true"
            :inactive-value="false"
            @change="change3Dor2D"
          >
          </el-switch>
        </el-form-item>
        <el-form-item label="是否需要地球背景图">
          <el-switch
            v-model="config.hasEarthImg"
            :active-value="true"
            :inactive-value="false"
            @change="changeEarthImg"
          >
          </el-switch>
        </el-form-item>
        <el-form-item label="切换柱状/热力图">
          <el-switch
            v-model="config.dataType"
            :active-value="0"
            :inactive-value="1"
            @change="changeDataType"
          >
          </el-switch>
        </el-form-item>
        <el-form-item label="地球背景色">
          <el-color-picker
            v-model="config.earthBackground"
            show-alpha
            @change="changeBackground"
          />
        </el-form-item>
        <el-form-item label="国家背景色">
          <el-color-picker
            v-model="config.countryBackground"
            show-alpha
            @change="changeCountryBackground"
          />
        </el-form-item>
      </el-form>
    </div>
    <div id="earth"></div>
  </div>
  <el-dialog v-model="message.visible" title="详细信息" width="30%">
    <span>{{ message.name }}:{{ message.value }}</span>
  </el-dialog>
</template>

<script lang="ts" setup>
import changeConfig from "./config";
import { EarthDataType, EarthData } from "./interface";
import * as echarts from "echarts";
import "echarts-gl";
import { onBeforeUnmount, onMounted, reactive, watch } from "vue";
let earth: echarts.ECharts;
let map: echarts.ECharts;
let data: EarthData[] = [
  {
    name: "China",
    lng: "116.417492",
    lat: "39.920469",
    value: 10,
  },
  {
    name: "Russia",
    lng: "105.779255",
    lat: "61.910247",
    value: 20,
  },
  {
    name: "Japan",
    lng: "139.777459",
    lat: "35.764312",
    value: 30,
  },
  {
    name: "France",
    lng: "2.349652",
    lat: "48.876294",
    value: 40,
  },
  {
    name: "United States",
    lng: "-100.601621",
    lat: "40.667118",
    value: 50,
  },
];
const message = reactive({
  name: "",
  value: "" as string | number,
  visible: false,
});
const config = reactive({
  is3D: true,
  hasEarthImg: true,
  earthBackground: "",
  countryBackground: "",
  dataType: 0 as EarthDataType,
});
const select = reactive({
  selectData: [1, 10],
  selectDataCopy: [1, 10],
});
const common = reactive({
  isHover: false,
  hoverTimer: 0,
  rotaionTime: 36, //默认一圈36秒
  autoRotateAfterStill: 2, //在鼠标静止操作后恢复自动旋转的时间间隔
});

watch(
  () => select.selectData,
  (value) => {
    select.selectDataCopy = value;
  },
  { deep: true }
);
watch(
  () => common.isHover,
  (value) => {
    if (value) {
      // hover停止旋转
      if (common.hoverTimer) {
        window.clearTimeout(common.hoverTimer);
      }
      earth.setOption({
        globe: {
          viewControl: {
            autoRotate: false,
          },
        },
      });
    } else {
      // 移出旋转
      common.hoverTimer = window.setTimeout(() => {
        earth.setOption({
          globe: {
            viewControl: {
              autoRotate: true,
            },
          },
        });
      }, common.autoRotateAfterStill * 1000);
    }
  },
  { deep: true }
);

/**
 * @description 获取筛选范围
 */
const getSelectData = () => {
  let dataValue = data.map((item) => item.value);
  select.selectData[1] = Math.max(...dataValue);
};
/**
 * @description 改变地球背景图
 */
const changeEarthImg = (value: boolean) => {
  changeConfig.changeEarthImg(earth, value);
};
/**
 * @description 改变地球背景
 */
const changeBackground = (value: string) => {
  changeConfig.changeEarthColor(map, value);
};
/**
 * @description 改变国家颜色
 */
const changeCountryBackground = (value: string) => {
  changeConfig.changeCountryColor(map, value);
};
/**
 * @description 切换2/3d
 */
const change3Dor2D = () => {
  map.dispose();
  if (config.is3D) {
    initEarth();
  } else {
    earth.dispose();
    map = null as unknown as echarts.ECharts;
    const chartDom = document.getElementById("earth") as HTMLElement;
    map = echarts.init(chartDom);
    map.setOption({
      tooltip: {
        trigger: "item",
      },
      visualMap: {
        min: select.selectData[0],
        max: select.selectData[1],
        calculable: true, //是否显示拖拽用的手柄
        realtime: false, //拖拽时，是否实时更新。
      },
      series: [
        {
          type: "map",
          map: "world",
          // 绘制完整尺寸的 echarts 实例
          top: 0,
          left: 0,
          right: 0,
          bottom: 0,
          boundingCoords: [
            [-180, 90],
            [180, -90],
          ],
          roam: true, //是否开启鼠标缩放和平移漫游。
          label: {
            show: true, //展示国家名
          },
          itemStyle: {
            areaColor: "transparent",
          },
          // hover样式
          emphasis: {},
          // select样式
          select: {
            disabled: true,
          },
          data: data.map((item) => ({ name: item.name, value: item.value })),
        },
      ],
    });
  }
};
/**
 * @description 筛选数据
 */
const filterData = () => {
  return data.filter(
    (item) =>
      item.value >= select.selectDataCopy[0] &&
      item.value <= select.selectDataCopy[1]
  );
};
/**
 * @description 切换柱状图或热力图
 */
const changeDataType = () => {
  if (config.dataType) {
    // 热力图
    dataSeriesUpdate(map, filterData());
    dataSeriesUpdate(earth);
  } else {
    // 柱状图
    dataSeriesUpdate(map);
    dataSeriesUpdate(
      earth,
      data.map((item) => [item.lng, item.lat, item.value])
    );
  }
};
/**
 * @description 配置地图
 */
const initMap = () => {
  // 使用 echarts 绘制世界地图的实例作为纹理
  var canvas = document.createElement("canvas");
  map = echarts.init(canvas, undefined, {
    width: 4096,
    height: 2048,
  });
  map.setOption({
    series: [
      {
        type: "map",
        map: "world",
        // 绘制完整尺寸的 echarts 实例
        top: 0,
        left: 0,
        right: 0,
        bottom: 0,
        boundingCoords: [
          [-180, 90],
          [180, -90],
        ],
        label: {
          show: true, //展示国家名
        },
        itemStyle: {
          areaColor: "transparent",
        },
        // hover样式
        emphasis: {},
        // select样式
        select: {
          disabled: true,
        },
        data: data.map((item) => ({ name: item.name, value: item.value })),
      },
    ],
  });
  /**
   * @description 点击国家展示信息
   */
  map.on("click", (event: any) => {
    message.name = event.name;
    let messageData = data.find((item) => item.name === event.name);
    message.value = messageData ? messageData.value : "暂无信息";
    message.visible = true;
  });
};
/**
 * @description 配置3d地球
 */
const initEarth = () => {
  const chartDom = document.getElementById("earth") as HTMLElement;
  earth = echarts.init(chartDom);
  echarts.registerMap("world", {
    geoJSON: require("@/utils/world.json"),
    specialAreas: {},
  });

  initMap();

  const earthOption = {
    backgroundColor: "transparent",
    globe: {
      baseTexture: require("@/assets/earth.jpg"),
      shading: "color",
      environment: require("@/assets/starfield.jpg"),
      layers: [
        {
          type: "blend",
          texture: map, //将map和3d地球纹理混合
        },
      ],
      viewControl: {
        autoRotate: true, //是否开启视角绕物体的自动旋转查看
        autoRotateAfterStill: common.autoRotateAfterStill, //在鼠标静止操作后恢复自动旋转的时间间隔
        animation: false, // 非常重要！！！ 解决setOption时停顿一下的问题
      },
    },
    visualMap: {
      min: select.selectData[0],
      max: select.selectData[1],
      calculable: true, //是否显示拖拽用的手柄
      realtime: false, //拖拽时，是否实时更新。
    },
    series: [
      {
        type: "bar3D",
        name: "earthBar",
        coordinateSystem: "globe", // 使用的坐标系
        barSize: 1, //柱子大小
        minHeight: 1, //最小高度
        silent: false, //不触发任何事件
        data: [
          [0, 0, 0],
          ...data.map((item) => [item.lng, item.lat, item.value]),
        ], //这里必须要加一个数据，否则数据展示不完整
      },
    ],
  };
  earth.setOption(earthOption);

  earth.on("datarangeselected", (event: any) => {
    select.selectDataCopy = event.selected;
    if (config.dataType) {
      dataSeriesUpdate(map, filterData());
    }
  });
  map.getZr().on("mousemove", () => {
    if (config.is3D) {
      common.isHover = true;
    }
  });
  map.getZr().on("mouseout", () => {
    if (config.is3D) {
      common.isHover = false;
    }
  });
};

/**
 * @description 更新实例的数据
 */
const dataSeriesUpdate = (
  instance: echarts.ECharts,
  data: Array<EarthData> | (string | number)[][] = []
) => {
  let visualMap = {
    min: select.selectData[0],
    max: select.selectData[1],
  };
  if (instance === earth) {
    instance.setOption({
      visualMap,
      series: [
        {
          data: [[0, 0, 0], ...data], //当数据大于2个时，最后一个元素无法展示
        },
      ],
    });
  } else {
    instance.setOption({
      visualMap,
      series: [
        {
          data, //当数据大于2个时，最后一个元素无法展示
        },
      ],
    });
  }
};

onMounted(() => {
  getSelectData();
  initEarth();
});
onBeforeUnmount(() => {
  if (common.hoverTimer) {
    window.clearTimeout(common.hoverTimer);
  }
});
</script>

<style lang="scss" scoped>
.earth-container {
  width: 100%;
  height: 100%;
  .handle-container {
    height: 100px;
  }
  #earth {
    width: 100%;
    height: calc(100vh - 100px);
  }
}
</style>
