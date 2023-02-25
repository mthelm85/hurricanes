<template>
  <v-row style="margin-left: 0px;">
    <v-col style="margin-top: -30px; z-index: 999;">
      <svg height="40px" width="100%">
        <text
          v-for="(n, i) in generateRange(20, 100, 8)"
          :key="`t${i}`"
          :x ="i * 30 + 2.5"
          y="15"
          fill="white"
          text-anchor="start"
          font-size="12px"
        >
        {{  Math.round(n)  }}{{ n == 100 ? '+' : '' }}
        </text>
        <text fill="white" font-size="12px" :x="9 * 27" y="15">mph</text>
        <rect
          v-for="(n, i) in generateRange(20, 100, 80)"
          :key="i"
          width="3px"
          height="20px"
          :fill="getColor(n)"
          :x="i * 3"
          y="20">
        </rect>
      </svg>
    </v-col>
  </v-row>
  <div ref="map" class="map-container"></div>
</template>

<script>
import "ol/ol.css";
import { Map, View } from "ol";
import TileLayer from "ol/layer/Tile";
import OSM from "ol/source/OSM";
import Feature from "ol/Feature.js";
import { useGeographic } from "ol/proj";
import VectorLayer from "ol/layer/Vector";
import VectorSource from "ol/source/Vector";
import { Fill, Stroke, Style } from "ol/style";
import { LineString } from "ol/geom";
import { click } from "ol/events/condition.js";
import Select from "ol/interaction/Select.js";
import hurdat from "@/assets/hurdat.json";
import { scaleLinear } from "d3-scale";
import { interpolateOranges } from "d3-scale-chromatic";

useGeographic();

export default {
  name: "OpenLayersMap",
  emits: ["select", "noSelect"],
  props: {
    year: Number,
  },

  data() {
    return {
      map: null,
      storms: JSON.parse(hurdat),
    };
  },

  methods: {
    addSelectInteraction() {
      this.selectInteraction.on("select", (e) => {
        const feature = e.selected[0];
        if (feature) {
          this.$emit("select", {
            name: feature.get("name"),
            maxStrength: feature.get("maxStrength"),
            dateRange: feature.get("dateRange"),
          });
        } else {
          this.$emit("noSelect");
        }
      });
    },

    generateRange(start, end, length) {
      const step = (end - start) / (length - 1);
      return Array.from({ length }, (_, i) => start + i * step);
    },

    getColor(maxWind) {
      return scaleLinear()
        .domain([20, 40, 100])
        .range(["lightgreen", "yellow", "red"])(maxWind);
    },

    getMinMaxDates(dates) {
      const parsedDates = dates.map((dateString) => new Date(dateString));
      const minDate = new Date(Math.min(...parsedDates)).toLocaleDateString(
        "en-US",
        {
          month: "2-digit",
          day: "2-digit",
          year: "2-digit",
        }
      );
      const maxDate = new Date(Math.max(...parsedDates)).toLocaleDateString(
        "en-US",
        {
          month: "2-digit",
          day: "2-digit",
          year: "2-digit",
        }
      );
      return { min: minDate, max: maxDate };
    },

    getStyle(storm) {
      const max_wind = Math.max(...storm.max_wind);
      return new Style({
        fill: new Fill({
          color: this.getColor(max_wind),
        }),
        stroke: new Stroke({
          color: this.getColor(max_wind),
          width: 6,
          lineDash: [2, 10],
        }),
      });
    },

    splitObjectByType(obj) {
      const result = [];
      let currentType = obj.storm_type[0];
      let currentDateTime = [obj.date_time[0]];
      let currentLat = [obj.lat[0]];
      let currentLon = [obj.lon[0]];
      let currentMaxWind = [obj.max_wind[0]];

      for (let i = 1; i < obj.storm_type.length; i++) {
        if (obj.storm_type[i] !== currentType) {
          result.push({
            name: obj.name,
            date_time: currentDateTime,
            lat: currentLat,
            lon: currentLon,
            max_wind: currentMaxWind,
            storm_type: [currentType],
          });
          currentType = obj.storm_type[i];
          currentDateTime = [obj.date_time[i - 1], obj.date_time[i]];
          currentLat = [obj.lat[i - 1], obj.lat[i]];
          currentLon = [obj.lon[i - 1], obj.lon[i]];
          currentMaxWind = [obj.max_wind[i - 1], obj.max_wind[i]];
        } else {
          currentDateTime.push(obj.date_time[i]);
          currentLat.push(obj.lat[i]);
          currentLon.push(obj.lon[i]);
          currentMaxWind.push(obj.max_wind[i]);
        }
      }

      result.push({
        name: obj.name,
        date_time: currentDateTime,
        lat: currentLat,
        lon: currentLon,
        max_wind: currentMaxWind,
        storm_type: [currentType],
      });

      return result;
    },

    stormPath(obj) {
      let path = [];
      for (let i = 0; i < obj.lat.length; i++) {
        path.push([obj.lon[i], obj.lat[i]]);
      }
      return path;
    },
  },

  computed: {
    currentStorms() {
      return this.storms.filter((obj) => {
        const dateTime = obj.date_time[0];
        const objYear = new Date(dateTime).getFullYear();
        return objYear === this.year;
      });
    },

    pathLayer() {
      return new VectorLayer({
        name: "stormPaths",
        source: new VectorSource({
          features: this.paths,
        }),
      });
    },

    paths() {
      let feats = [];
      for (let i = 0; i < this.currentStorms.length; i++) {
        const stormComponents = this.splitObjectByType(this.currentStorms[i]);
        for (let j = 0; j < stormComponents.length; j++) {
          let f = new Feature({
            geometry: new LineString(this.stormPath(stormComponents[j])),
            name: stormComponents[j].name,
            maxStrength: Math.max(...stormComponents[j].max_wind),
            dateRange: this.getMinMaxDates(stormComponents[j].date_time),
          });
          f.setStyle(this.getStyle(stormComponents[j]));
          feats.push(f);
        }
      }
      return feats;
    },

    selectInteraction() {
      return new Select({
        condition: click,
        layers: [this.pathLayer],
        filter: function (feature, layer) {
          return feature.getGeometry().getType() === "LineString";
        },
      });
    },
  },

  mounted() {
    this.map = new Map({
      target: this.$refs.map,
      layers: [
        new TileLayer({
          source: new OSM(),
        }),
      ],
      view: new View({
        center: [-81.5158, 27.6648],
        zoom: 5,
      }),
    });

    this.map.addLayer(this.pathLayer);
    this.map.addInteraction(this.selectInteraction);
    this.addSelectInteraction();
  },

  watch: {
    year: {
      handler() {
        this.map.getLayers().forEach((layer) => {
          if (layer && layer.get("name") === "stormPaths") {
            this.map.removeLayer(layer);
          }
        });
        this.map.addLayer(this.pathLayer);
        this.map.addInteraction(this.selectInteraction);
        this.addSelectInteraction();
      },
    },
  },
};
</script>

<style scoped>
.map-container {
  height: 80%;
  margin-left: 12px;
  margin-right: 12px;
}
</style>