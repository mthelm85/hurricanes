<template>
  <v-app>
    <v-main style="margin-left: 30px; margin-right: 30px; height: 100%">
      <v-row>
        <v-col style="margin: 40px 0px -10px 0px">
          <v-slider
            v-model="yr"
            :min="1851"
            :max="2021"
            :step="1"
            thumb-label="always"
          ></v-slider>
        </v-col>
      </v-row>
      <v-row :style="{ visibility: stormVisibility }" class="storm-info">
        <v-col>Storm: {{ name }}</v-col>
        <v-col>Dates: {{ dateRange.min }} - {{ dateRange.max }}</v-col>
        <v-col>Max Strength: {{ Math.round(maxStrength * 1.15078) }} mph</v-col>
      </v-row>
      <Map
        :year="yr"
        @select="($e) => updateStorm($e)"
        @noSelect="($e) => noSelection()"
      />
    </v-main>
  </v-app>
</template>

<script>
import Map from "@/components/Map.vue";

export default {
  components: {
    Map,
  },

  data() {
    return {
      yr: 1936,
      name: "",
      dateRange: "",
      maxStrength: "",
      stormVisibility: "hidden",
      lastEvent: "",
    };
  },

  methods: {
    noSelection() {
      if (Math.abs(new Date() - this.lastEvent) >= 250) {
        this.stormVisibility = "hidden";
        this.name = "";
        this.maxStrength = "";
        this.dateRange = "";
      }
    },

    updateStorm(storm) {
      this.stormVisibility = "visible";
      this.name = storm.name;
      this.maxStrength = storm.maxStrength;
      this.dateRange = storm.dateRange;
      this.lastEvent = new Date();
    },
  },

  watch: {
    yr: {
      handler() {
        this.stormVisibility = "hidden";
        this.name = "";
        this.maxStrength = "";
        this.dateRange = "";
      },
    },
  },
};
</script>
<style>
.storm-info {
  margin: 0px 0px 10px 0px;
}
</style>