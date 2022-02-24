<template>
  <div>
    <div id="map" class="map"></div>
    <form class="form-inline">
      <label for="type">Measurement type &nbsp;</label>
      <select id="type" @change="selectChange" v-model="typeSelect">
        <option value="length">Length (LineString)</option>
        <option value="area">Area (Polygon)</option>
      </select>
    </form>
  </div>
</template>
 
<script lang="ts">
import "ol/ol.css";
import Draw from "ol/interaction/Draw";
import Map from "ol/Map";
import Overlay from "ol/Overlay";
import View from "ol/View";
import { Circle as CircleStyle, Fill, Stroke, Style } from "ol/style";
import { LineString, Polygon } from "ol/geom";
import { OSM, Vector as VectorSource } from "ol/source";
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer";
import { getArea, getLength } from "ol/sphere";
import { unByKey } from "ol/Observable";

const raster = new TileLayer({
  source: new OSM(),
});

const source = new VectorSource();

const vector = new VectorLayer({
  source: source,
  style: new Style({
    fill: new Fill({
      color: "rgba(255, 255, 255, 0.2)",
    }),
    stroke: new Stroke({
      color: "#ffcc33",
      width: 2,
    }),
    image: new CircleStyle({
      radius: 7,
      fill: new Fill({
        color: "#ffcc33",
      }),
    }),
  }),
});

/**
 * Currently drawn feature.
 * @type {import("../src/ol/Feature.js").default}
 */
let sketch;

/**
 * The help tooltip element.
 * @type {HTMLElement}
 */
let helpTooltipElement;

/**
 * Overlay to show the help messages.
 * @type {Overlay}
 */
let helpTooltip;

/**
 * The measure tooltip element.
 * @type {HTMLElement}
 */
let measureTooltipElement;

/**
 * Overlay to show the measurement.
 * @type {Overlay}
 */
let measureTooltip;

/**
 * Message to show when the user is drawing a polygon.
 * @type {string}
 */
const continuePolygonMsg = "Click to continue drawing the polygon";

/**
 * Message to show when the user is drawing a line.
 * @type {string}
 */
const continueLineMsg = "Click to continue drawing the line";

let draw; // global so we can remove it later

export default {
  name: "MapGraphic3",
  data() {
    return {
      map: {},
      typeSelect: "length",
    };
  },
  mounted() {
    this.map = new Map({
      layers: [raster, vector],
      target: "map",
      view: new View({
        center: [-11000000, 4600000],
        zoom: 15,
      }),
    });
    this.map.on("pointermove", this.pointerMoveHandler);
    this.map.getViewport().addEventListener("mouseout", function () {
      helpTooltipElement.classList.add("hidden");
    });
    this.addInteraction();
  },

  methods: {
    pointerMoveHandler(evt) {
      if (evt.dragging) {
        return;
      }
      /** @type {string} */
      let helpMsg = "Click to start drawing";

      if (sketch) {
        const geom = sketch.getGeometry();
        if (geom instanceof Polygon) {
          helpMsg = continuePolygonMsg;
        } else if (geom instanceof LineString) {
          helpMsg = continueLineMsg;
        }
      }

      helpTooltipElement.innerHTML = helpMsg;
      helpTooltip.setPosition(evt.coordinate);

      helpTooltipElement.classList.remove("hidden");
    },
    formatArea(polygon) {
      const area = getArea(polygon);
      let output;
      if (area > 10000) {
        output =
          Math.round((area / 1000000) * 100) / 100 + " " + "km<sup>2</sup>";
      } else {
        output = Math.round(area * 100) / 100 + " " + "m<sup>2</sup>";
      }
      return output;
    },
    formatLength(line) {
      const length = getLength(line);
      let output;
      if (length > 100) {
        output = Math.round((length / 1000) * 100) / 100 + " " + "km";
      } else {
        output = Math.round(length * 100) / 100 + " " + "m";
      }
      return output;
    },
    addInteraction() {
      const type = this.typeSelect == "area" ? "Polygon" : "LineString";
      
      draw = new Draw({
        source: source,
        type: type,
        style: new Style({
          fill: new Fill({
            color: "rgba(255, 255, 255, 0.2)",
          }),
          stroke: new Stroke({
            color: "rgba(0, 0, 0, 0.5)",
            lineDash: [10, 10],
            width: 2,
          }),
          image: new CircleStyle({
            radius: 5,
            stroke: new Stroke({
              color: "rgba(0, 0, 0, 0.7)",
            }),
            fill: new Fill({
              color: "rgba(255, 255, 255, 0.2)",
            }),
          }),
        }),
      });
      this.map.addInteraction(draw);

      this.createMeasureTooltip();
      this.createHelpTooltip();
      let that = this;
      let listener;
      draw.on("drawstart", function (evt) {
        // set sketch
        sketch = evt.feature;

        /** @type {import("../src/ol/coordinate.js").Coordinate|undefined} */
        let tooltipCoord = evt.coordinate;

        listener = sketch.getGeometry().on("change", function (evt) {
          const geom = evt.target;
          let output;
          if (geom instanceof Polygon) {
            output = that.formatArea(geom);
            tooltipCoord = geom.getInteriorPoint().getCoordinates();
          } else if (geom instanceof LineString) {
            output = that.formatLength(geom);
            tooltipCoord = geom.getLastCoordinate();
          }
          measureTooltipElement.innerHTML = output;
          measureTooltip.setPosition(tooltipCoord);
        });
      });

      draw.on("drawend", function () {
        measureTooltipElement.className = "ol-tooltip ol-tooltip-static";
        measureTooltip.setOffset([0, -7]);
        // unset sketch
        sketch = null;
        // unset tooltip so that a new one can be created
        measureTooltipElement = null;
        that.createMeasureTooltip();
        unByKey(listener);
      });
    },
    createHelpTooltip() {
      if (helpTooltipElement) {
        helpTooltipElement.parentNode.removeChild(helpTooltipElement);
      }
      helpTooltipElement = document.createElement("div");
      helpTooltipElement.className = "ol-tooltip hidden";
      helpTooltip = new Overlay({
        element: helpTooltipElement,
        offset: [15, 0],
        positioning: "center-left",
      });
      this.map.addOverlay(helpTooltip);
    },
    createMeasureTooltip() {
      if (measureTooltipElement) {
        measureTooltipElement.parentNode.removeChild(measureTooltipElement);
      }
      measureTooltipElement = document.createElement("div");
      measureTooltipElement.className = "ol-tooltip ol-tooltip-measure";
      measureTooltip = new Overlay({
        element: measureTooltipElement,
        offset: [0, -15],
        positioning: "bottom-center",
        stopEvent: false,
        insertFirst: false,
      });
      this.map.addOverlay(measureTooltip);
    },
    selectChange() {
      this.map.removeInteraction(draw);
      this.addInteraction();
    },
  },
};
</script>
<style>
.map {
  width: 100%;
  height: 400px;
}
.ol-tooltip {
  position: relative;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 4px;
  color: white;
  padding: 4px 8px;
  opacity: 0.7;
  white-space: nowrap;
  font-size: 12px;
  cursor: default;
  user-select: none;
}
.ol-tooltip-measure {
  opacity: 1;
  font-weight: bold;
}
.ol-tooltip-static {
  background-color: #ffcc33;
  color: black;
  border: 1px solid white;
}
.ol-tooltip-measure:before,
.ol-tooltip-static:before {
  border-top: 6px solid rgba(0, 0, 0, 0.5);
  border-right: 6px solid transparent;
  border-left: 6px solid transparent;
  content: "";
  position: absolute;
  bottom: -6px;
  margin-left: -7px;
  left: 50%;
}
.ol-tooltip-static:before {
  border-top-color: #ffcc33;
}
</style>