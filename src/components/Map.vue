<template>
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
import { scaleQuantize } from "d3-scale";
import { schemeRdYlBu } from "d3-scale-chromatic";

useGeographic();

export default {
  name: "OpenLayersMap",

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

    getColor(maxWind) {
      return scaleQuantize()
        .domain([30, 96])
        .range([
          "#c4c4c4",
          "#ffda05",
          "#ff9900",
          "#ff5900",
          "#ff2a00",
          "#ff006f",
        ])(maxWind);
    },

    getMinMaxDates(dates) {
      const parsedDates = dates.map((dateString) => new Date(dateString));
      const minDate = new Date(Math.min(...parsedDates)).toLocaleDateString(
        "en-US",
        {
          month: "2-digit",
          day: "2-digit",
          year: "numeric",
        }
      );
      const maxDate = new Date(Math.max(...parsedDates)).toLocaleDateString(
        "en-US",
        {
          month: "2-digit",
          day: "2-digit",
          year: "numeric",
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
  height: 85%;
}
</style>