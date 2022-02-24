<template>
  <div class="MapGraphic">
    <div class="mapWrapper">
      <div id="map" class="map" :style="{ width, height }"></div>
      <div class="box">
        <p>我是标题</p>
      </div>
    </div>
    <label for="speed">
      speed:&nbsp;
      <input id="speed" type="range" min="10" max="999" step="10" value="60" />
    </label>
    <button id="start-animation">Start Animation</button>
  </div>
</template>

<script>
import "ol/ol.css";
import Feature from "ol/Feature";
import Map from "ol/Map";
import Point from "ol/geom/Point";
import Polyline from "ol/format/Polyline";
import VectorSource from "ol/source/Vector";
import View from "ol/View";
import XYZ from "ol/source/XYZ";
import { Circle as CircleStyle, Fill, Icon, Stroke, Style } from "ol/style";
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer";
import { getVectorContext } from "ol/render";
import axios from 'axios'

export default {
  name: "MapGraphic",
  props: {
    width: String,
    height: String,
  },
  data() {
    return {
      map: null,
    };
  },
  mounted() {
    this.mapInit();
  },
  methods: {
    mapInit() {
      const key = "e3nh8NYw5bj3jJtehyLu";
      const attributions =
        '<a href="https://www.maptiler.com/copyright/" target="_blank">&copy; MapTiler</a> ' +
        '<a href="https://www.openstreetmap.org/copyright" target="_blank">&copy; OpenStreetMap contributors</a>';

      const center = [-5639523.95, -3501274.52];
      const map = new Map({
        target: document.getElementById("map"),
        view: new View({
          center: center,
          zoom: 10,
          minZoom: 2,
          maxZoom: 19,
        }),
        layers: [
          new TileLayer({
            source: new XYZ({
              attributions: attributions,
              url:
                "https://api.maptiler.com/maps/hybrid/{z}/{x}/{y}.jpg?key=" +
                key,
              tileSize: 512,
            }),
          }),
        ],
      });
      
      axios.get(
        "https://openlayers.org/en/latest/examples/data/polyline/route.json"
      )
      // fetch(
      //   "https://openlayers.org/en/latest/examples/data/polyline/route.json"
      //   // "@/assets/polyline/route.json"
      // )
      .then(function (response) {
        console.log('response',response);
        // response.json().then(function (result) {
          // console.log("result", result);
          const polyline = response.data.routes[0].geometry;

          const route = new Polyline({
            factor: 1e6,
          }).readGeometry(polyline, {
            dataProjection: "EPSG:4326",
            featureProjection: "EPSG:3857",
          });

          const routeFeature = new Feature({
            type: "route",
            geometry: route,
          });
          const startMarker = new Feature({
            type: "icon",
            geometry: new Point(route.getFirstCoordinate()),
          });
          const endMarker = new Feature({
            type: "icon",
            geometry: new Point(route.getLastCoordinate()),
          });
          const position = startMarker.getGeometry().clone();
          const geoMarker = new Feature({
            type: "geoMarker",
            geometry: position,
          });

          const styles = {
            route: new Style({
              stroke: new Stroke({
                width: 6,
                color: [237, 212, 0, 0.8],
              }),
            }),
            icon: new Style({
              image: new Icon({
                anchor: [0.5, 1],
                src: require("@/assets/icon.png"),
              }),
            }),
            // geoMarker: new Style({
            //   image: new CircleStyle({
            //     radius: 7,
            //     fill: new Fill({ color: "black" }),
            //     stroke: new Stroke({
            //       color: "white",
            //       width: 2,
            //     }),
            //   }),
            // }),
            geoMarker: new Style({
              image: new Icon({
                anchor: [0.5, 0.5],
                src: require("@/assets/UAV.png"),
              }),
            }),
          };

          const vectorLayer = new VectorLayer({
            source: new VectorSource({
              features: [routeFeature, geoMarker, startMarker, endMarker],
            }),
            style: function (feature) {
              return styles[feature.get("type")];
            },
          });

          map.addLayer(vectorLayer);

          const speedInput = document.getElementById("speed");
          const startButton = document.getElementById("start-animation");
          let animating = false;
          let distance = 0;
          let lastTime;

          function moveFeature(event) {
            const speed = Number(speedInput.value);
            const time = event.frameState.time;
            const elapsedTime = time - lastTime;
            distance = (distance + (speed * elapsedTime) / 1e6) % 2;
            lastTime = time;

            const currentCoordinate = route.getCoordinateAt(
              distance > 1 ? 2 - distance : distance
            );
            position.setCoordinates(currentCoordinate);
            const vectorContext = getVectorContext(event);
            vectorContext.setStyle(styles.geoMarker);
            vectorContext.drawGeometry(position);
            // tell OpenLayers to continue the postrender animation
            map.render();
          }

          function startAnimation() {
            animating = true;
            lastTime = Date.now();
            startButton.textContent = "Stop Animation";
            vectorLayer.on("postrender", moveFeature);
            // hide geoMarker and trigger map render through change event
            geoMarker.setGeometry(null);
          }

          function stopAnimation() {
            animating = false;
            startButton.textContent = "Start Animation";

            // Keep marker at current animation position
            geoMarker.setGeometry(position);
            vectorLayer.un("postrender", moveFeature);
          }

          startButton.addEventListener("click", function () {
            if (animating) {
              stopAnimation();
            } else {
              startAnimation();
            }
          });
        // });
      
      });
    },
  },
};
</script>

<style>
.MapGraphic {
  text-align: left;
}
.mapWrapper {
  position: relative;
}
.box {
  display: none;
  width: 800px;
  height: 450px;
  border: 2px solid #fff;
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  margin: auto;
  box-sizing: border-box;
  text-align:center;
  color: #fff;
}
</style>