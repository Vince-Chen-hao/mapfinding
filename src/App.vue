<template>
  <div id="app">
    <div class="home row no-gutters">
      <div class="col-md-3">
        <div class="toolbox">
          <div class="d-flex titlename">
          <i class="fas fa-street-view mt-1"></i>
            <div>口罩查詢地圖</div>
          </div>
          <div class="shadow-sm">
            <div class="date d-flex justify-content-between">
              <div>
                <h2 class="date-week ml-1">{{date.getDay() | convertToChineseDay}}</h2>
                <!-- covert函數會套用另一數值當作參數 -->
              </div>
              <div class="date-description">
                <div class="calendar d-flex justify-content-end align-items-baseline">
                  <i class="far fa-calendar-alt mr-2"></i>
                  <div class="date-today">{{date | convertToDateString}}</div>
                </div>
                <div class="date-id">
                  身分證末碼是
                  <span class="date-highlight">{{date.getDay() % 2 === 0 ? "雙號" : "單號"}}</span>
                  可購買
                </div>
              </div>
            </div>

            <div class="search-bar p-2">
              <div class="form-group d-flex">
                <label for="city" class="mr-2 col-form-label">縣市</label>
                <div class="flex-fill">
                  <select class="form-control"
                    id="city"
                    v-model="select.city"
                    @change="select.area = ''"
                  >
                  <!-- 當選擇其他選項時，就會觸發change裡的指令 -->
                    <option value>-- 請選擇縣市 --</option>
                    <option
                      v-for="item in cityName"
                      :value="item.CityName"
                      :key="item.CityName"
                    >{{ item.CityName }}</option>
                  </select>
                </div>
              </div>
              <div class="form-group d-flex">
                <label for="area" class="mr-2 col-form-label">地區</label>
                <div class="flex-fill">
                  <select class="form-control"
                    id="area"
                    v-if="select.city.length"
                    v-model="select.area"
                    @change="updateSelect" 
                  >
                    <option value>-- 請選擇地區 --</option>
                    <option
                      :value="area.AreaName"
                      v-for="area in cityName.find((city) => city.CityName === select.city).AreaList"
                      :key="area.AreaName"
                    >{{ area.AreaName }}</option>
                  </select>
                </div>
              </div>
              <div class="d-flex text-muted justify-content-end">
                <i class="fa fa-bullhorn mr-2"></i>
                <p class="mb-0 small">兩週每人限購九片口罩</p>
              </div>
            </div>
          </div>

          <ul class="list-group text-dark">
            <template v-for="(item, key) in data">
              
                <!-- v-if這裡的用法是指“顯示” 兩個選擇列與data裡資料相同的，其他就隱藏 -->
              <a class="list-group-item"
                :key="key" 
                v-if="item.properties.county === select.city && item.properties.town === select.area"
                :class="{ 'mask-highlight': !item.properties.mask_adult && !item.properties.mask_child}"
                @click="penTo(item)"
              >
                <div class="d-flex align-items-baseline">
                  <i class="fas fa-hospital-symbol mr-1" style="font-size:18px;color:orange"></i>
                  <div class="info_title">{{ item.properties.name }}</div>
                </div>
                <div class="info_address">
                  <p class="mb-0">
                    地址：
                    <a :href="`https://www.google.com.tw/maps/place/${item.properties.address}`"
                      title="Google Map"
                    >{{ item.properties.address }}</a>
                  </p>
                </div>
                <div class="info_phone">
                  <p>電話：{{ item.properties.phone }}</p>
                </div>
                <div class="d-flex justify-content-between my-2">
                  <div class="pharmacy-mask"
                    :class="{ 'selled': item.properties.mask_adult === 0}">
                    <div class="pharmacy-mask-title">成人口罩: {{item.properties.mask_adult}}個</div>
                  </div>
                  <div class="pharmacy-mask"
                    :class="{ 'selled': item.properties.mask_child === 0}">
                    <div class="pharmacy-mask-title">兒童口罩: {{item.properties.mask_child}}個</div>
                  </div>
                </div>
              </a>
            </template>
          </ul>
        </div>
      </div>
      <div class="col-md-9">
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
import L from "leaflet";
import cityName from "./assets/taiwanArea.json";

console.log(L); // 載leaflet近來需要使用consolelog

//流程取得API資料後，呼叫updateMarker取得select選項的對應資料，匯入map顯示地圖座標

let osmap = {};

const iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
};

const icons = {
  orange: new L.Icon({
    iconUrl: 'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-orange.png',
    ...iconsConfig,
  }),
  grey: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png',
    ...iconsConfig, 
  }),
};



export default {
  name: "App",
  data: () => ({
    data: [],
    cityName, //導入json縣市區域後須啟動，而且可在瀏覽器看到資料
    select: {
      city: "臺北市",
      area: "萬華區"
    },
    date: new Date()
  }),

  methods: {
    updateMarker() {
      //取出區域
      const pharmacies = this.data.filter(pharmacy => {
        //如果data裡的area沒有資訊，則找出與data裡的city相同的縣市
        if (!this.select.area) {
          return pharmacy.properties.county === this.select.city;
        }
        //找出與data裡的area相同的區域
        return pharmacy.properties.town === this.select.area;
      });

      console.log(pharmacies);
      //利用forEach執行動作，取得座標＋顯示內容
      pharmacies.forEach(pharmacy => {
        const { properties, geometry } = pharmacy;
        const icon = properties.mask_adult || properties.mask_child ? icons.orange : icons.grey;
        L.marker([
          geometry.coordinates[1],geometry.coordinates[0],properties],{icon})
          .addTo(osmap).bindPopup(`<h5>${properties.name}</h5>
            電話： ${properties.phone}<br>
            地址：<a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br> 
            <small>資料更新時間：${properties.updated}</small> 
            <p>備註： ${properties.note}</p>
            <hr>  
            <mark>口罩數量：<b>成人<ins> 
            ${properties.mask_adult} </ins>個 / 兒童 <ins> ${properties.mask_child}</ins> 個 
            </b></mark><br>
            `);
      });
      //上面執行的forEach是顯示初步取得（select選項）的地圖資訊，而非點擊後的

      this.penTo(pharmacies[0]); //回傳入第一個藥局進入penTo下方的語法
      
    },
    // 當select切換區域時(@change))，就會觸發移除指標
    updateSelect() {
      osmap.eachLayer(layer => {
        if (layer instanceof L.Marker) {
          osmap.removeLayer(layer);
        }
      });
      this.updateMarker();
    },

    penTo(item) {
      const { properties, geometry } = item;
      const icon = properties.mask_adult || properties.mask_child ? icons.orange : icons.grey;

      //panTo的功用是點擊後會移動至指定位置並聚焦放大
      osmap.panTo([
        geometry.coordinates[1],
        geometry.coordinates[0],
        properties
      ]); 
      
      //取得指定區域的位置
      L.marker([geometry.coordinates[1], geometry.coordinates[0]],{icon})
        .addTo(osmap).bindPopup(
          `<h5>${properties.name}</h5>
            電話：${properties.phone}<br>
            地址： <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br> 
            <small>資料更新時間：${properties.updated}</small> 
            <p>備註：${properties.note}</p>
            <hr>  
            <mark>口罩數量：<b>成人有<ins> 
            ${properties.mask_adult} </ins>個 / 兒童有 <ins> ${properties.mask_child}</ins> 個 
            </b></mark><br>
            `
        )
        .openPopup(); 
        //彈跳鼠標藥局的“資訊文字”
        //這裡會顯示聚焦後(點擊後會放大)的資訊
    }
  },

  filters: {
    convertToChineseDay(day) {
      const chineseDay = ["日", "一", "二", "三", "四", "五", "六"];
      return `星期${chineseDay[day]}`;
      //此時的參數(day)))是套用在Ｉ前方的數值，自動套入無需宣告
      //取得getday()取得星期，星期日＝0 星期一 = 1....
      //https://cythilya.github.io/2017/05/23/vue-filter/
    },
    convertToDateString(date) {
      const tensdigit = n => {
        return n < 10 ? `0${n}` : n; //例如：A ? B : C ，如果 A 為真，那走 B，否則 C
      };

      return `${date.getFullYear()}-${tensdigit(
        date.getMonth() + 1 //month[0] = 一月，真實月份需再加1
      )}-${tensdigit(date.getDate())}`;
    }
  },

  mounted() {
    const url =
      "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";
    this.$http.get(url).then(response => {
      this.data = response.data.features;
      console.log("<data>"+JSON.stringify(this.data));
      this.updateMarker(); //取得資料後執行更新
    });

    osmap = L.map("map", {
      center: [25.03, 121.55],
      zoom: 16
    });

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png?{foo}", {
      foo: "bar",
      attribution:
        'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>',
      maxZoom: 18
    }).addTo(osmap);

    L.marker([25.03, 121.55]).addTo(osmap);
  }
};
</script>

<style lang="scss">
@import "bootstrap/scss/bootstrap";

#map {
  height: 110vh;
}

.home {
  position: relative;
}

.titlename {
  font-size: 20px;
  color: rgba(245, 248, 250, 0.973);
  justify-content: center;
  display: flex;
  background-color: rgb(116, 171, 192);
  font-family: Roboto;
  letter-spacing: 8px;
}

.mask-highlight {
  background: #ffe3e3;
}

.toolbox {
  background-color: rgb(143, 212, 195);
  color: rgba(245, 248, 250, 0.973);
  a {
    cursor: pointer;
  }
}

.list-group {
  overflow-y: auto;
  height: 75vh;
}

.list-group-item {
  &:hover {
    background-color: #d3f0e9;
  }
}

.date {
  padding: 16px 16px 8px;
  justify-content: space-between;
  align-items: center;
}
.date-week {
  font-size: 34px;
  font-weight: normal;
  margin: 0;
}

.date-id {
  font-size: 14px;
}
.date-highlight {
  color: #fff126;
  font-size: 16px;
  font-weight: bold;
}

.info_title {
  font-size: 18px;
  font-weight: bold;
  color: #555454;
}

.info_address {
  color: #666666;
  font-size: 14px;
  line-height: 19px;
  font-family: Roboto
}

.info_phone {
  color: #666666;
  font-size: 14px;
  line-height: 19px;
  font-family: Roboto
}


.pharmacy-mask {
  height: 35px;
  padding: 8px 16px;
  border-radius: 10px;
  background-color: rgb(122, 211, 189);
  color: rgb(252, 246, 246);
  //在此父層的指定class元素
  &.selled {
    background-color: #e2e2e2;
    color: #9c9c9c;
  }
}

.pharmacy-mask-title {
  font-size: 15px;
  line-height: 17px;
  font-family: Roboto;
}


</style>
