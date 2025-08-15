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
  name: 'NovaMapFormField',
  props: ["resourceName", "resourceId", "field", "errors"],
  components: {
    MapExport,
    PolygonFormField,
    MultiPolygonFormField,
    PointFormField,
  },
  expose: ["fill"], // Expose the fill method
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
    image(value) {
      // Also emit when image changes
      this.emitFieldValueChange(this.currentField.attribute, this.fieldValue);
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
  created() {
    // Set fill method early in lifecycle
    console.log('FormField created');
  },
  mounted() {
    // The key insight: Nova calls field.fill(), not component.fill()
    // So we need to add the fill method to the FIELD OBJECT itself
    if (this.currentField) {
      // Bind our fill method to the field object
      this.currentField.fill = (formData) => {
        console.log("Field.fill() called via field object!", {
          visible: this.currentField.visible,
          attribute: this.currentField.attribute,
          fieldValue: this.fieldValue,
          image: this.image,
        });

        if (this.currentField.visible) {
          formData.append(
            this.currentField.attribute + "[value]",
            this.fieldValue || ""
          );

          if (this.image) {
            formData.append(
              this.currentField.attribute + "[image]",
              this.image || ""
            );
          }
        }
      };
      
      console.log('FormField mounted, added fill method to field object');
      console.log('Field object now has fill method:', typeof this.currentField.fill);
    }
    
    // Debug: Check if the method is available
    console.log('FormField mounted, component fill method available:', typeof this.fill);
    console.log('Current field object:', this.currentField);
  },
  methods: {
    emitFieldValueChange(attribute, value) {
      this.$emit("field-changed");
      // Ensure Vue knows about the data changes
      this.$nextTick(() => {
        // Force reactivity update
        this.$forceUpdate();
      });
    },

    // Keep this as a backup method on the component itself
    fill(formData) {
      console.log("Component.fill() called as backup!", {
        visible: this.currentField.visible,
        attribute: this.currentField.attribute,
        fieldValue: this.fieldValue,
        image: this.image,
      });

      if (this.currentField.visible) {
        formData.append(
          this.currentField.attribute + "[value]",
          this.fieldValue || ""
        );

        if (this.image) {
          formData.append(
            this.currentField.attribute + "[image]",
            this.image || ""
          );
        }
      }
    },
  },
  
  beforeUnmount() {
    // Clean up - remove the fill method from the field object
    if (this.currentField && this.currentField.fill) {
      delete this.currentField.fill;
    }
  },
};
</script>
