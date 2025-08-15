<template>
  <default-field
    :field="currentField"
    :errors="allErrors"
    :full-width-content="true"
    :show-help-text="!isReadonly && showHelpText"
  >
    <template #field>
      <div
        class="z-10 p-0 w-full form-control form-input-bordered overflow-hidden relative"
        :class="mapErrorClasses"
        :style="{ height: currentField.mapHeight + 'px' }"
      >
        <point-form-field
          v-if="mapType === 'POINT'"
          ref="mapField"
          v-model="fieldValue"
          :field="currentField"
          :resource-id="resourceId"
          :resource-name="resourceName"
          :readonly="currentField.readonly"
        />

        <polygon-form-field
          v-else-if="mapType === 'POLYGON'"
          ref="mapField"
          v-model="fieldValue"
          :field="currentField"
          :resource-id="resourceId"
          :resource-name="resourceName"
          :readonly="currentField.readonly"
        />

        <multi-polygon-form-field
          v-else-if="mapType === 'MULTI_POLYGON'"
          ref="mapField"
          v-model="fieldValue"
          :field="currentField"
          :resource-id="resourceId"
          :resource-name="resourceName"
          :readonly="currentField.readonly"
        />
      </div>

      <map-export
        v-if="currentField?.capture?.enabled"
        v-model="image"
        :field="currentField"
        :field-value="fieldValue"
      />
    </template>
  </default-field>
</template>

<script>
import PointFormField from "./form-fields/PointFormField";
import PolygonFormField from "./form-fields/PolygonFormField";
import MultiPolygonFormField from "./form-fields/MultiPolygonFormField";
import MapExport from "./other/MapExport";

export default {
  props: ["resourceName", "resourceId", "field", "errors"],
  components: {
    MapExport,
    PolygonFormField,
    MultiPolygonFormField,
    PointFormField,
  },
  data() {
    return {
      fieldValue: "",
      image: null,
    };
  },
  watch: {
    fieldValue(value) {
      this.emitFieldValueChange(this.currentField.attribute, value);
    },
  },
  computed: {
    currentField() {
      return this.field;
    },

    allErrors() {
      const attribute = this.currentField.attribute;
      const valueKey = attribute + ".value";
      const imageKey = attribute + ".image";

      const errors = {};

      errors[valueKey] = [
        ...(this.errors && this.errors.has && this.errors.has(valueKey)
          ? this.errors.get(valueKey)
          : []),
        ...(this.errors && this.errors.has && this.errors.has(imageKey)
          ? this.errors.get(imageKey)
          : []),
      ];

      return errors;
    },

    mapType() {
      return this.currentField.mapType;
    },

    hasLocationError() {
      return (
        this.errors &&
        this.errors.has &&
        this.errors.has(this.currentField.attribute)
      );
    },

    mapErrorClasses() {
      return this.hasLocationError ? this.errorClass : "";
    },

    errorClass() {
      return "form-input-border-error";
    },

    isReadonly() {
      return this.currentField.readonly || false;
    },

    showHelpText() {
      return (
        this.currentField.helpText && this.currentField.helpText.length > 0
      );
    },
  },
  methods: {
    emitFieldValueChange(attribute, value) {
      this.$emit("field-changed");
    },

    fill(formData) {
      if (this.currentField.visible) {
        formData.append(
          this.currentField.attribute + "[value]",
          this.fieldValue || ""
        );
        formData.append(
          this.currentField.attribute + "[image]",
          this.image || ""
        );
      }
    },
  },
};
</script>
