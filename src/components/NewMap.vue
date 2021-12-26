<template>
  <div class="main">
    <div class="flex">
      <!-- Map Container -->
      <div id="mapContainer"></div>

      <!-- Display Arena here -->
      <div class="dislpay-arena">
        <div class="display-header">
          <h2>GPS Tracker</h2>
          <template v-if="isArrive">
            <h3>Arrived at Destination</h3>
            <LineChart ref="lineRef" :chartData="chartData" />
          </template>
          <h3 v-if="onTheWay">On the Way..</h3>
          <button
            :disabled="onTheWay"
            :class="{ disabled: onTheWay }"
            type="button"
            class="control-btn"
            @click="startDriving()"
          >
            Start
          </button>
          <button type="button" class="control-btn" @click="pause()">
            Pause
          </button>
          <button type="button" class="control-btn" @click="stop()">
            Stop
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import L from 'leaflet';
import "leaflet/dist/leaflet.css";
import "leaflet-rotatedmarker/leaflet.rotatedMarker.js"
import routeMap from "../routeMap.json";
import { LineChart } from 'vue-chart-3';
import { Chart, registerables } from 'chart.js';

var index = 0
var vehicle = null
Chart.register(...registerables);

export default {
  components: {
    LineChart
  },
  data () {
    return {
      map: null,
      zoom: 13,
      center: [3.575307, 98.684053],
      polyLine: [],
      driving: null,
      isArrive: false,
      onTheWay: false,
      iconOption: {
        iconUrl: 'https://cdn.pixabay.com/photo/2014/04/02/16/28/car-307371_1280.png',
        iconSize: [14, 28],
      },
      chartData: {
        datasets: [{
          data: [],
          backgroundColor: '#d80739',
          label: 'Speed m/s'
        }],
        labels: []
      }
    }
  },

  mounted () {
    this.loadPolyline()
    this.createMap()
  },

  beforeDestroy () {
    clearInterval(this.driving);
  },

  methods: {
    createMap () {
      this.map = L.map('mapContainer').setView(this.center, this.zoom, { animation: true });
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(this.map);
      L.polyline(this.polyLine, { color: 'green' }).addTo(this.map)
      let carIcon = L.icon({ iconUrl: this.iconOption.iconUrl, iconSize: this.iconOption.iconSize });
      vehicle = L.marker(this.center, {
        icon: carIcon,
        rotationAngle: 0,
        rotationOrigin: "center"
      }).addTo(this.map);
    },

    loadPolyline () {
      this.polyLine = []
      routeMap[0].geometry.map(el => {
        this.polyLine.push(el.coordinate)
      })
    },

    startDriving () {
      this.zoom = 15
      this.onTheWay = true
      this.isArrive = false
      this.map.setView(this.center, this.zoom);
      if (!index) this.resetChartData()
      this.driving = setInterval(() => {
        if (index >= (routeMap[0].geometry.length - 1)) {
          index = 0;
          this.isArrive = true;
          this.onTheWay = false
          clearInterval(this.driving);
        } else {
          index++;
          let current = routeMap[0].geometry[index]
          vehicle.setRotationAngle(current.bearing)
          vehicle.setLatLng(current.coordinate)
          this.chartData.datasets[0].data.push(current.speed);
          this.chartData.labels.push(current.timeStamp);
          this.map.panTo(current.coordinate)
        }
      }, 10000);
    },

    pause () {
      clearInterval(this.driving);
      this.onTheWay = false
    },

    stop () {
      clearInterval(this.driving);
      index = 0;
      this.onTheWay = false
      vehicle.setRotationAngle(0)
      vehicle.setLatLng(this.center)
    },

    resetChartData () {
      this.chartData.datasets[0].data = []
      this.chartData.labels = []
    }
  }
}
</script>
<style>
.main {
  padding: 25px;
}
.flex {
  display: flex;
  flex-wrap: wrap;
  flex-direction: row;
  justify-content: space-between;
}
#mapContainer {
  width: 65%;
  height: 80vh;
}
.dislpay-arena {
  background: #ffffff;
  box-shadow: 0px -3px 10px rgba(0, 58, 78, 0.1);
  border-radius: 5px;
  padding: 20px 30px;
  width: 30%;
}
.display-header {
  margin-bottom: 50px;
}
.location-control {
  height: 30px;
  background: #ffffff;
  border: 1px solid rgba(31, 42, 83, 0.25);
  box-shadow: 0px 0px 10px rgba(73, 165, 198, 0.1);
  border-radius: 4px;
  padding: 0px 10px;
  width: 90%;
}
.location-control:focus {
  outline: none;
}
.control-btn {
  margin: 15px 5px 5px 5px;
  padding: 10px 15px;
  background: #d80739;
  box-shadow: 0px 4px 10px rgba(73, 165, 198, 0.1);
  border-radius: 5px;
  border: 0;
  cursor: pointer;
  color: #ffffff;
  font-size: 0.875rem;
  font-weight: 600;
}
.disabled {
  background: #db7990;
  cursor: not-allowed;
}

@media screen and (max-width: 1280px) {
  .dislpay-arena {
    width: 100%;
  }
  #mapContainer {
    width: 100%;
    height: 73vh;
  }
}
</style>