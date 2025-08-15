<template>
  <div class="map-container">
    <ol-map
      ref="map"
      :load-tiles-while-animating="true"
      :load-tiles-while-interacting="true"
      :style="mapStyles"
      :controls="[]"
      @click="onMapClick"
    >
      <ol-view
        :center="center"
        :rotation="rotation"
        :zoom="zoom"
        :projection="projection"
      />

      <ol-tile-layer>
        <ol-source-osm :url="field.templateUrl" />
      </ol-tile-layer>

      <ol-vector-layer>
        <ol-source-vector ref="source" :projection="projection">
          <ol-feature v-if="hasInitValue">
            <ol-geom-point
              :coordinates="[initValue.longitude, initValue.latitude]"
            />
          </ol-feature>

          <template v-if="!isReadonly && !exportable">
            <ol-interaction-modify v-if="isDirty" @modifyend="onModifyEnd" />
            <ol-interaction-draw v-else type="Point" @drawend="onDrawEnd" />
          </template>
        </ol-source-vector>

        <ol-style>
          <ol-style-icon :src="markerIcon" :anchor="[0.5, 1]" :scale="0.1" />
        </ol-style>
      </ol-vector-layer>

      <ol-zoom-control v-if="withZoomControl && !exportable" />
      <ol-zoomslider-control v-if="withZoomSlider && !exportable" />
      <ol-fullscreen-control v-if="withFullScreenControl && !exportable" />
    </ol-map>
  </div>
</template>

<script>
import { toLonLat, fromLonLat } from "ol/proj";
import HasMap from "../../mixins/HasMap";
import HasSearchBox from "../../mixins/HasSearchBox";
import ExportsMap from "../../mixins/ExportsMap";

export default {
  mixins: [HasMap, HasSearchBox, ExportsMap],
  props: ["field", "readonly"],
  expose: ["initCenter", "capture", "isDirty"],
  data() {
    return {
      isDirty: false,
      selectPointOnSearch: true,
      initValue: {
        longitude: null,
        latitude: null,
      },
      fieldValue: "",
    };
  },
  computed: {
    isReadonly() {
      return this.readonly === true;
    },

    hasInitValue() {
      return !!this.initValue.latitude && !!this.initValue.longitude;
    },
  },
  methods: {
    initCenter() {
      let value = {};

      if (this.field.value) {
        value = JSON.parse(this.field.value);
      }

      if (value.latitude && value.longitude) {
        const lonLat = fromLonLat([value.longitude, value.latitude]);

        this.center = lonLat;
        this.initValue = {
          longitude: lonLat[0],
          latitude: lonLat[1],
        };

        this.setValue(lonLat[1], lonLat[0]);
      } else {
        this.center = fromLonLat([
          this.field.defaultLongitude,
          this.field.defaultLatitude,
        ]);
      }
    },

    onDrawEnd(evt) {
      console.log("🎯 onDrawEnd triggered", {
        evt,
        coordinate: evt.feature?.getGeometry().getCoordinates(),
      });
      const coordinate = evt.feature?.getGeometry().getCoordinates();

      if (Array.isArray(coordinate)) {
        console.log("✅ Setting point from draw:", {
          latitude: coordinate[1],
          longitude: coordinate[0],
        });
        this.setValue(coordinate[1], coordinate[0]);
      }
    },

    onModifyEnd(evt) {
      console.log("✏️ onModifyEnd triggered", { evt });
      const coordinate = evt.features
        ?.getArray()[0]
        .getGeometry()
        .getCoordinates();

      console.log("📍 Coordinate from modify:", coordinate);
      if (Array.isArray(coordinate)) {
        console.log("✅ Setting point from modify:", {
          latitude: coordinate[1],
          longitude: coordinate[0],
        });
        this.setValue(coordinate[1], coordinate[0]);
      }
    },

    onMapClick(evt) {
      console.log("🖱️ Map clicked!", {
        coordinate: evt.coordinate,
        isDirty: this.isDirty,
        isReadonly: this.isReadonly,
        exportable: this.exportable,
      });

      // Only handle clicks when we already have a point (Google Maps-like behavior)
      if (this.isDirty && !this.isReadonly && !this.exportable) {
        console.log("🎯 Handling map click for point movement");
        const coordinate = evt.coordinate;

        if (Array.isArray(coordinate)) {
          console.log("✅ Moving point to new location:", {
            latitude: coordinate[1],
            longitude: coordinate[0],
          });
          // Update the existing point's position
          this.updatePointPosition(coordinate);
          this.setValue(coordinate[1], coordinate[0]);
        }
      } else {
        console.log("❌ Map click ignored - conditions not met");
      }
    },

    updatePointPosition(coordinate) {
      console.log("🔄 Updating point position:", coordinate);
      // Get the current feature and update its geometry
      const source = this.$refs.source?.source;
      if (source) {
        const features = source.getFeatures();
        console.log("📊 Found features:", features.length);
        if (features.length > 0) {
          const pointFeature = features[0];
          pointFeature.getGeometry().setCoordinates(coordinate);
          console.log("✅ Point geometry updated");

          // Update initValue to reflect the new position
          this.initValue = {
            longitude: coordinate[0],
            latitude: coordinate[1],
          };
          console.log("✅ initValue updated:", this.initValue);
        }
      } else {
        console.log("❌ No source found");
      }
    },

    setDirty() {
      this.isDirty = true;
    },

    setValue(latitude, longitude) {
      const lonLat = toLonLat([longitude, latitude]);
      const data = {
        longitude: lonLat[0],
        latitude: lonLat[1],
      };

      this.setDirty();
      this.fieldValue = JSON.stringify(data);
    },
  },
};
</script>
