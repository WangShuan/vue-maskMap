<template>
  <div id="app">
    <div class="row no-gutters">
      <div class="col-sm-3">
        <div class="toolbox">
          <div class="sticky-top bg-white shadow-sm p-2">
            <div class="form-group d-flex">
              <label for="cityName" class="mr-2 col-form-label text-right">縣市</label>
              <div class="flex-fill">
                <select id="cityName" class="form-control" v-model="select.city">
                  <option value>--- 請選擇 ---</option>
                  <option v-for="c in cityName" :key="c.CityName" :value="c.CityName">{{c.CityName}}</option>
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right">地區</label>
              <div class="flex-fill">
                <select id="area" class="form-control" v-model="select.town">
                  <option value>--- 請選擇 ---</option>
                  <option v-for="a in areaName" :key="a.AreaName" :value="a.AreaName">{{a.AreaName}}</option>
                </select>
              </div>
            </div>
            <p class="mb-0 small text-muted text-right">請先選擇區域查看（綠色表示還有口罩）</p>
          </div>
          <ul class="list-group">
            <template v-for="item in filterData">
              <a
                :class="{ 'highlight': item.properties.mask_adult || item.properties.mask_child}"
                class="list-group-item bg-light text-secondary text-left"
                :key="item.properties.id"
                @click="penTo(item)"
              >
                <h3>{{item.properties.name}}</h3>
                <p
                  class="mb-0"
                >成人口罩：{{item.properties.mask_adult}} | 兒童口罩：{{item.properties.mask_child}}</p>
                <p class="mb-0">
                  地址：
                  <a
                    :href="`https://www.google.com.tw/maps/place/${item.properties.address}`"
                    target="_blank"
                    title="Google Map"
                  >{{item.properties.address}}</a>
                </p>
              </a>
            </template>
          </ul>
        </div>
      </div>
      <div class="col-sm-9">
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
import cityName from "./assets/cityName";

let osmMap = {};

const iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
};

const icons = {
  green: new L.Icon({
    iconUrl:
      "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png",
    ...iconsConfig,
  }),
  grey: new L.Icon({
    iconUrl:
      "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png",
    ...iconsConfig,
  }),
};

export default {
  name: "App",
  data() {
    return {
      data: [],
      cityName,
      select: { city: "臺北市", town: "大安區" },
      nowData: [],
      layers: [],
      myGroup: "",
    };
  },
  methods: {
    penTo(item) {
      L.popup()
        .setLatLng([
          item.geometry.coordinates[1] + 0.001,
          item.geometry.coordinates[0],
        ])
        .setContent(
          `
        藥局名稱：${item.properties.name}
        <br>
        地址: <a href="https://www.google.com.tw/maps/place/${
          item.properties.address
        }" target="_blank">${item.properties.address}</a>
        <br>
        電話: ${item.properties.phone}
        <br>
        口罩數量：<strong>成人 ${
          item.properties.mask_adult
            ? `${item.properties.mask_adult} 個`
            : "未取得資料"
        }
        / 兒童 ${
          item.properties.mask_child
            ? `${item.properties.mask_child} 個`
            : "未取得資料"
        }</strong>
        <br>
        <small>最後更新時間: ${item.properties.updated}</small>`
        )
        .openOn(osmMap);

      osmMap.setView(
        [item.geometry.coordinates[1], item.geometry.coordinates[0]],
        15
      );
    },
    removeMarker() {
      if (this.myGroup) {
        this.myGroup.clearLayers();
      }
    },
    addMark() {
      const vm = this;
      vm.nowData.forEach((item) => {
        const icon =
          item.properties.mask_adult || item.properties.mask_child
            ? icons.green
            : icons.grey;
        let layer = new L.marker(
          [item.geometry.coordinates[1], item.geometry.coordinates[0]],
          {
            icon,
          }
        ).bindPopup(
          `
        藥局名稱：${item.properties.name}
        <br>
        地址: <a href="https://www.google.com.tw/maps/place/${
          item.properties.address
        }" target="_blank">${item.properties.address}</a>
        <br>
        電話: ${item.properties.phone}
        <br>
        口罩數量：<strong>成人 ${
          item.properties.mask_adult
            ? `${item.properties.mask_adult} 個`
            : "未取得資料"
        }
        / 兒童 ${
          item.properties.mask_child
            ? `${item.properties.mask_child} 個`
            : "未取得資料"
        }</strong>
        <br>
        <small>最後更新時間: ${item.properties.updated}</small>`
        );
        vm.layers.push(layer);
        vm.myGroup = L.layerGroup(vm.layers);
        osmMap.addLayer(vm.myGroup);
      });
      vm.layers = [];
    },
  },
  computed: {
    areaName: function () {
      const vm = this;
      let arr = vm.cityName.find((item) => {
        return item.CityName === vm.select.city;
      });
      return arr.AreaList;
    },
    filterData() {
      const vm = this;
      let items = vm.data.filter((item) => {
        return (
          item.properties.county === vm.select.city &&
          item.properties.town === vm.select.town
        );
      });
      vm.nowData = items;
      vm.removeMarker();
      vm.addMark();
      if (items[0]) {
        vm.penTo(items[0]);
      }
      return items;
    },
  },
  mounted() {
    osmMap = L.map("map", {
      center: [25.0286424, 121.5385066],
      zoom: 15,
    });
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution:
        '<a target="_blank" href="https://www.openstreetmap.org/">© OpenStreetMap 貢獻者</a>',
      maxZoom: 18,
    }).addTo(osmMap);

    const vm = this;
    const API =
      "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";
    vm.$http.get(API).then((res) => {
      vm.data = res.data.features;
    });
  },
};
</script>

<style lang="scss">
@import "bootstrap/scss/bootstrap";

#map {
  height: 100vh;
}
.highlight {
  background: #e3ffe8 !important;
}
.toolbox {
  height: 100vh;
  overflow-y: auto;
  a {
    cursor: pointer;
  }
}
</style>
