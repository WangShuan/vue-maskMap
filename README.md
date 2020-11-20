# vue-mask

## 結構

這次使用 `vue-cli 3` 以上做的 

主要用到的套件有 `axios` `bootstrap` `scss` 地圖部分使用 `leaflet` 通過 CDN 加載

另外抓取了縣市 json 檔 之後要用在 `select` 中

## 重點 API 講解

1. 初始化地圖 `L.map("地圖在頁面中的 id ",{center:[x座標,y座標],zoom:num})`

  * `L`：`leaflet` 的縮寫

  * `center`：顯示時的中心點座標

  * `zoom`：顯示的縮放大小 範圍 `0~18` 推薦範圍 `15~18`

  在 `mounted` 時就初始化地圖 順便獲取遠端的藥局資料

```js

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
  const API = "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";
  vm.$http.get(API).then((res) => {
    vm.data = res.data.features;
  });
},

```

2. 引入 `cityName` 並將其通過 `v-for` 放到 `options` 中

給 `select` 添加 `v-model` 為 `city` 讓每次選擇後執行 `computed` 獲取地區

地區為 `areaName` 通過 `v-for` 放到地區對應的的 `options` 中 並給其 `select` 添加 `v-model` 為 `town`

計算代碼如下：

```js

computed: {
  areaName: function () {
    const vm = this;
    let arr = vm.cityName.find((item) => {
      return item.CityName === vm.select.city;
    });
    return arr.AreaList;
  }
}

```

3. 接下來撰寫一個增加地標的方法 `addMark`

首先創建一個 `newData` 資料 裡面存放當前被篩選到的地區有哪些藥局

通過 `forEach` 方法遍歷 `newData` 將每個藥局建立自己的 `marker` 

再將 `marker` 存放到 `layers` 變數中

接下來創建一個 `myGroup` 變數為 `L.layerGroup(layers)`

最後再通過 `osmMap.addLayer(myGroup)` 方法把 `myGroup` 增加到地圖上

最後把 `layers` 清空 這樣才不會一直添加同個地區的 `marker`

代碼如下：

```js

addMark() {
  const vm = this;
  vm.nowData.forEach((item) => {
    const icon = item.mask_adult || item.mask_child ? icons.green : icons.grey;
    let layer = new L.marker([item.x, item.y],{ icon }).bindPopup(`以下略`);
    vm.layers.push(layer);
    vm.myGroup = L.layerGroup(vm.layers);
    osmMap.addLayer(vm.myGroup);
  });
  vm.layers = [];
},

```

4. 增加一個移除所有地標的方法 `removeMarker`

該方法直接將 `addMarker` 的 `myGroup` 刪除即可

  * 這裡要注意 一開始 `myGroup` 會是空的 所以我們要加上判斷 `myGroup` 是否為空再執行

代碼如下：

```js

if(this.myGroup){
  this.myGroup.clearLayers();
}

```

5. 切換地圖中心點的方法 `penTo`

`penTo` 要接收一個藥局的資料 這裏我們默認傳入當前選中地區的第一筆資料

首先通過 `L.popup()` 方法創建一個 `Popup` 工具

然後使用 `map.setView([x,y],num)` 方法將地圖的中心點重設

代碼如下：

```js

penTo(item) {
  L.popup()
    .setLatLng([item.x + 0.001,item.y,])
    .setContent(`以下略`)
    .openOn(osmMap);

  osmMap.setView(
    [item.x, item.y],
    15
  );
},

```

6. 最後使用 `computed` 計算當前顯示的藥局清單

添加一個 `filterData` 當每次下拉選單的選項改變時就會觸發

裡面使用了剛才增加的 `addMarker` `penTo` `removeMarker` 方法

先使用 `filter` 方法判斷當前的縣市與地區 然後篩選出當前地區的所有藥局

接下來將所有藥局傳入 `nowData` 中 並執行 `removeMarker` 方法

再執行 `addMarker` 方法 接下來執行 `penTo` 方法

最後 `return nowData` 為 `filterData` 的結果

  * 這裡要先執行刪除 再執行添加是因為要先把上一個地區的地標全部移除