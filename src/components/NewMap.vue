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
import 'leaflet.markercluster/dist/leaflet.markercluster.js';
import 'leaflet.markercluster/dist/MarkerCluster.css'
import 'leaflet.markercluster/dist/MarkerCluster.Default.css'
import "leaflet-rotatedmarker/leaflet.rotatedMarker.js"
import routeMap from "../routeMap.json";
import { LineChart } from 'vue-chart-3';
import { Chart, registerables } from 'chart.js';
import Centrifuge from "centrifuge"
var YOUR_TOKEN = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdGFydGRlbW8iLCJleHAiOjE2NDM5NTkwNDh9.OBdfOPKD4VwhqLqmYO-0jz2AwyL6_9pRIO7zLXdjQHg'
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
      marker: [],
      chartData: {
        datasets: [{
          data: [],
          backgroundColor: '#d80739',
          label: 'Speed m/s'
        }],
        labels: []
      },
      colors: ['#007bff', '#37c936', '#868e96', '#ff8f00', '#d81900'],
      clusteredPoints: null,
      markers: {}

    }
  },

  mounted () {
    this.loadPolyline()
    this.createMap()
    this.centrifugeTest()
  },

  beforeDestroy () {
    clearInterval(this.driving);
  },

  methods: {
    centrifugeTest () {
      var centrifuge = new Centrifuge('ws://206.189.42.13:8000/connection/websocket');
      centrifuge.setToken(YOUR_TOKEN);

      var callbacks = {
        "publish": (message) => {
          // See below description of message format
          console.log("publish", message);
          this.createMarker(message)
        },
        "join": function (message) {
          // See below description of join message format
          console.log("join", message);
        },
        "leave": function (message) {
          // See below description of leave message format
          console.log("leave", message);
        },
        "subscribe": (context) => {
          // See below description of subscribe callback context format
          console.log("subscribe", context);
        },
        "error": function (errContext) {
          // See below description of subscribe error callback context format
          console.log("error", errContext);
        },
        "disconnect": function (errContext) {
          // See below description of subscribe error callback context format
          console.log("disconnect", errContext);
        },
        "connect": function (context) {
          console.log("connect", context);
        },
        "unsubscribe": function (context) {
          // See below description of unsubscribe event callback context format
          console.log("unsubscribe", context);
        }
      }
      centrifuge.on('disconnect', function (context) {
        // do whatever you need in case of disconnect from server
        console.log("disconnect", context)
      });
      centrifuge.subscribe("positions:startdemo", callbacks);
      centrifuge.connect();

    },
    createMap () {
      this.map = L.map('mapContainer').setView(this.center, this.zoom, { zoomControl: true, zoomAnimation: false, fadeAnimation: true, markerZoomAnimation: true });
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(this.map);
      L.polyline(this.polyLine, { color: 'green' }).addTo(this.map)
      let carIcon = L.icon({ iconUrl: this.iconOption.iconUrl, iconSize: this.iconOption.iconSize });
      vehicle = L.marker(this.center, {
        icon: carIcon,
        rotationAngle: 0,
        rotationOrigin: "center"
      }).addTo(this.map);



    },
    iconCar (status) {
      let indicator = 0
      if (status === 'PARK') indicator = 1
      else if (status === 'DRIVING') indicator = 2
      else if (status === 'STOP') indicator = 3
      else if (status === 'INACTIVE') indicator = 4
      else if (status === 'ALERT') indicator = 5
      let url = `https://start-dev.oss-ap-southeast-5.aliyuncs.com/files/icon-images/sedan0${indicator}_64.png`
      return L.icon({ iconUrl: url, iconSize: [28, 28], });
    },

    createMarker (data) {
      this.clusteredPoints = L.markerClusterGroup({
        spiderfyOnMaxZoom: true,
        showCoverageOnHover: true,
        removeOutsideVisibleBounds: true,
        iconCreateFunction: (cluster) => {
          let option = {
            mag1: 0,
            mag2: 0,
            mag3: 0,
            mag4: 0,
            mag5: 0
          }
          cluster.getAllChildMarkers().map(el => {
            if (el.options.information.status === 'DRIVING') option.mag1++
            else if (el.options.information.status === 'STOP') option.mag2++
            else if (el.options.information.status === 'PARK') option.mag3++
            else if (el.options.information.status === 'INACTIVE') option.mag4++
            else if (el.options.information.status === 'ALERT') option.mag5++
          })
          return L.divIcon({
            html: this.createDonutChart(option)
          });
        }
      });

      let customMarker = L.Marker.extend({
        options: {
          information: {},
          setInformation: function (context) { this.information = context },
          getInformation: function () { return this.information }
        }
      });

      data.data.map(el => {
        if (!this.markers[el.gps_id]) {
          this.markers[el.gps_id] = new customMarker([el.latitude, el.longitude], {
            title: el.gps_id,
            rotationOrigin: "center"
          })
          this.clusteredPoints.addLayer(this.markers[el.gps_id]);
        }

        this.markers[el.gps_id].setRotationAngle(el.course)
        this.markers[el.gps_id].setIcon(this.iconCar(el.status))
        this.markers[el.gps_id].setLatLng([el.latitude, el.longitude])
        this.markers[el.gps_id].options.setInformation(el)
        // this.clusteredPoints.refreshClusters()
      })
      this.map.addLayer(this.clusteredPoints)

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
      if (!index) {
        this.resetChartData()
        vehicle.setRotationAngle(0)
        vehicle.setLatLng(this.center)
      }
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
    },


    // code for creating an SVG donut chart from feature properties
    createDonutChart (props) {
      const offsets = [];
      const counts = [
        props.mag1,
        props.mag2,
        props.mag3,
        props.mag4,
        props.mag5
      ];
      let total = 0;
      for (const count of counts) {
        offsets.push(total);
        total += count;
      }
      const fontSize =
        total >= 1000 ? 22 : total >= 100 ? 20 : total >= 10 ? 18 : 16;
      const r =
        total >= 1000 ? 50 : total >= 100 ? 32 : total >= 10 ? 24 : 18;
      const r0 = Math.round(r * 0.6);
      const w = r * 2;

      let html = `<div>
      <svg width="${w}" height="${w}" viewbox="0 0 ${w} ${w}" text-anchor="middle" style="font: ${fontSize}px sans-serif; display: block">`;

      for (let i = 0; i < counts.length; i++) {
        html += this.donutSegment(
          offsets[i] / total,
          (offsets[i] + counts[i]) / total,
          r,
          r0,
          this.colors[i]
        );
      }
      html += `<circle cx="${r}" cy="${r}" r="${r0}" fill="white" />
      <text dominant-baseline="central" transform="translate(${r}, ${r})">
      ${total.toLocaleString()}
      </text>
      </svg>
      </div>`;

      const el = document.createElement('div');
      el.innerHTML = html;
      return el.firstChild;
    },

    donutSegment (start, end, r, r0, color) {
      if (end - start === 1) end -= 0.00001;
      const a0 = 2 * Math.PI * (start - 0.25);
      const a1 = 2 * Math.PI * (end - 0.25);
      const x0 = Math.cos(a0),
        y0 = Math.sin(a0);
      const x1 = Math.cos(a1),
        y1 = Math.sin(a1);
      const largeArc = end - start > 0.5 ? 1 : 0;

      // draw an SVG path
      return `<path d="M ${r + r0 * x0} ${r + r0 * y0} L ${r + r * x0} ${r + r * y0
        } A ${r} ${r} 0 ${largeArc} 1 ${r + r * x1} ${r + r * y1} L ${r + r0 * x1
        } ${r + r0 * y1} A ${r0} ${r0} 0 ${largeArc} 0 ${r + r0 * x0} ${r + r0 * y0
        }" fill="${color}" />`;
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