<template>
  <v-app>
    <v-main style="height: 100%">
      <v-row>
        <v-col style="margin: 40px 10px -35px 5px">
          <v-slider
            v-model="yr"
            :min="1954"
            :max="2021"
            :step="1"
            thumb-label="always"
            label="Year"
          ></v-slider>
        </v-col>
      </v-row>
      <v-row class="storm-info">
        <v-col>
          <v-card variant="tonal">
            <v-card-title>Storm Info <span class="text-subtitle-2" :style="{ visibility: subTitleVisibility }">(click a line segment to view)</span></v-card-title>
            <v-card-text :style="{ visibility: stormVisibility }">
              <v-row>
                <v-col>Storm: {{ name }}</v-col>
                <v-col>Dates: {{ dateRange.min }} - {{ dateRange.max }}</v-col>
                <v-col>Max Strength: {{ Math.round(maxStrength * 1.15078) }} mph</v-col
                >
              </v-row>
            </v-card-text>
          </v-card>
        </v-col>
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
      yr: 1987,
      name: "",
      dateRange: "",
      maxStrength: "",
      stormVisibility: "hidden",
      subTitleVisibility: "visible",
      lastEvent: "",
    };
  },

  methods: {
    noSelection() {
      if (Math.abs(new Date() - this.lastEvent) >= 250) {
        this.stormVisibility = "hidden";
        this.subTitleVisibility = "visible";
        this.name = "";
        this.maxStrength = "";
        this.dateRange = "";
      }
    },

    updateStorm(storm) {
      this.stormVisibility = "visible";
      this.subTitleVisibility = "hidden";
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
        this.subTitleVisibility = "visible";
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