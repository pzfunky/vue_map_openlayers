<template>
  <div>
    <div id="map" class="map"></div>
    <button class="zoom-in" @click="setZoomIn">zoom-in</button>&nbsp;
    <button class="zoom-out" @click="setZoomOut">zoom-out</button>
  </div>
</template>
 
<script lang="ts">
import "ol/ol.css";
import GeoJSON from "ol/format/GeoJSON";
import Map from "ol/Map";
import View from "ol/View";
import { Circle as CircleStyle, Fill, Icon, Stroke, Style } from "ol/style";
import { OSM, Vector as VectorSource } from "ol/source";
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer";

import Vue from "vue";

var image = new CircleStyle({
  radius: 5,
  fill: new Fill({
    color: "rgba(255, 0, 0, 1)",
  }),
  stroke: new Stroke({ color: "red", width: 1 }),
});

var styles = {
  Point: new Style({
    image: new Icon({
      anchor: [0.5, 0.5],
      src: require("@/assets/UAV.png"),
    })
  }),
};

var styleFunction = function (feature) {
  return styles[feature.getGeometry().getType()];
};

var geojsonObject = {
  type: "FeatureCollection",
  features: [
    {
      type: "Feature",
      geometry: {
        type: "Point",
        coordinates: [12945510, 4825390],
      },
    },
  ],
};

var vectorSource = new VectorSource({
  features: new GeoJSON().readFeatures(geojsonObject),
});

var vectorLayer = new VectorLayer({
  source: vectorSource,
  style: styleFunction,
});

export default Vue.extend({
  data() {
    return {
      map: {},
    };
  },
  mounted() {
    this.map = new Map({
      layers: [
        new TileLayer({
          source: new OSM(),
        }),
        vectorLayer,
      ],
      target: "map",
      view: new View({
        center: [12945510, 4825390],
        zoom: 5,
      }),
    });

    // setInterval(this.translate, 500);
  },

  methods: {
    translate() {
      vectorSource.forEachFeature(function (f) {
        var x = Math.random() * 1000000;
        var y = Math.random() * 1000000;
        console.log("translate",x,y);
        f.getGeometry().translate(x, y);
      });
    },
    setZoomIn(){
      let view = this.map.getView()
      let zoom = view.getZoom()
      view.setZoom(zoom + 1)
      console.log("setZoomIn",zoom);
    },
    setZoomOut(){
      let view = this.map.getView()
      let zoom = view.getZoom()
      view.setZoom(zoom - 1)
      console.log("setZoomOut",zoom);
    },
  },
});
</script>
<style>
.map {
  width: 100%;
  height: 600px;
}
</style>