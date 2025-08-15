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

// Simple Errors class to mimic Nova's Errors functionality
class Errors {
  constructor(errors = {}) {
    this.errors = errors;
  }

  has(field) {
    return this.errors.hasOwnProperty(field) && this.errors[field].length > 0;
  }

  get(field) {
    return this.errors.hasOwnProperty(field) ? this.errors[field] : [];
  }

  all() {
    return this.errors;
  }
}

export default {
  mixins: [Nova.mixins.FormField],
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
    value: {
      get() {
        return this.fieldValue;
      },
      set(value) {
        this.fieldValue = value;
      },
    },

    currentField() {
      return this.field;
    },

    allErrors() {
      const attribute = this.currentField.attribute;
      const valueKey = attribute + ".value";
      const imageKey = attribute + ".image";

      const errors = {};

      console.log(this.errors);

      const errorKeys = Object.keys(this.errors || {});

      errors[valueKey] = [
        ...(!errorKeys.length
          ? []
          : this.errors[valueKey]
          ? this.errors[valueKey]
          : []),
        ...(!errorKeys.length
          ? []
          : this.errors[imageKey]
          ? this.errors[imageKey]
          : []),
      ];

      return new Errors(errors);
    },

    mapType() {
      return this.currentField.mapType;
    },

    hasLocationError() {
      if (!this.errors) return false;

      // Handle both Errors object and plain object
      if (typeof this.errors.has === "function") {
        return this.errors.has(this.currentField.attribute);
      } else {
        return (
          Object.keys(this.errors).length > 0 &&
          this.errors[this.currentField.attribute] !== undefined
        );
      }
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

    fill: function (e) {
      this.currentField.visible &&
        (e.append(
          this.currentField.attribute + "[value]",
          this.fieldValue || ""
        ),
        e.append(this.currentField.attribute + "[image]", this.image || ""));
    },
  },
};
</script>
