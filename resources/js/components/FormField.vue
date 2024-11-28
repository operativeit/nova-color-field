<template>
  <DefaultField :field="currentField" :errors="errors" :show-help-text="showHelpText">
    <template #field>
      <div
        class="o1-inline-flex o1-mb-2 o1-overflow-hidden color-picker form-input form-control form-control-bordered o1-px-0"
        ref="inputArea"
        v-click-outside="handleClickOutside"
      >
        <div class="o1-bg-checkered" style="z-index: 2">
          <div
            class="o1-border-r o1-border-gray-100 dark:o1-border-gray-700 o1-cursor-pointer"
            :style="{ backgroundColor: rgbaValue, width: '36px', height: '36px' }"
            @click="togglePicker"
          />
        </div>

        <input
          :id="currentField.uniqueKey"
          :dusk="field.attribute"
          type="text"
          class="o1-w-25 o1-border-0 w-full form-control form-input form-input-bordered o1-rounded-l-none"
	  :class="errorClasses"
          :placeholder="placeholder"
          :value="displayValue"
          v-on:keydown.enter.prevent="handleEnter"
          @blur="handleRawInput"
          @focus="handleFocus"
          @click="showPicker"
          ref="inputField"
        />
      </div>

      <component
        v-if="shouldShowPicker"
        ref="pickerArea"
        :is="component"
        :id="field.name"
        :class="[
          'nc-picker',
          errorClasses,
          { absolute: currentField.autoHidePicker && currentField.pickerType !== 'slider', 'o1-z-10': true },
        ]"
        :palette="palette"
        :modelValue="vcValue"
        @update:modelValue="setVcValue"
      />

      <p v-if="hasError" class="o1-my-2 text-danger">
        {{ firstError }}
      </p>
    </template>
  </DefaultField>
</template>

<script>
import { DependentFormField, HandlesValidationErrors } from 'laravel-nova';
import { Chrome, Compact, Grayscale, Material, Photoshop, Sketch, Slider, Swatches, Twitter } from '@ckpack/vue-color';
import tinycolor from 'tinycolor2';

import directiveClickOutside from '../utils/directiveClickOutside'

export default {
  name: 'NovaColorField',
  mixins: [HandlesValidationErrors, DependentFormField],

  directives: {
      clickOutside: directiveClickOutside
  },
  components: {
    'chrome-picker': Chrome,
    'compact-picker': Compact,
    'grayscale-picker': Grayscale,
    'material-picker': Material,
    'photoshop-picker': Photoshop,
    'sketch-picker': Sketch,
    'slider-picker': Slider,
    'swatches-picker': Swatches,
    'twitter-picker': Twitter,
  },

  props: ['resourceName', 'resourceId', 'field'],

  data() {
    return {
      shouldShowPicker: !this.field.autoHidePicker,
      vcValue: {},
    };
  },

  beforeDestroy() {
    if (this.shouldShowPicker) {
      document.removeEventListener('click', this.documentClick);
    }
  },

  methods: {
    setInitialValue() {
      const val = this.field.value || null;
      this.value = val ? tinycolor(val) : '';
      this.vcValue = this.value? this.value.toRgbString() : '';
    },

    setVcValue(newValue) {
      this.value = tinycolor(newValue.hex8);
    },

    fill(formData) {
      if (!this.value) {
        formData.append(this.field.attribute, '');
        return;
      }
      formData.append(this.field.attribute, this.saveValue);
    },

    valueUpdated() {
      this.$refs.inputField.dispatchEvent(new Event('input', { bubbles: true }));

      if (this.field) {
        this.emitFieldValueChange(this.field.attribute, this.saveValue);
      }
    },

    handleClickOutside() {
      const inputArea = this.$refs.inputArea;
      inputArea.classList.remove('form-control-focused');
    },

    handleRawInput(event) {

      const inputArea = this.$refs.inputArea;
      const value = event.target.value;

      inputArea.classList.remove('form-control-focused');

      if (!value) {
        this.value = '';
        this.valueUpdated();
        return;
      }

      const color = tinycolor(value);
      if (color._format) {
        this.value = color;
        this.valueUpdated();
      }
    },

    handleFocus(event) {
      const inputArea = this.$refs.inputArea;
      inputArea.classList.add('form-control-focused');
    },

    handleEnter(event) {
      event.target.blur();
      this.hidePicker();
    },

    documentClick(event) {
      const inputArea = this.$refs.inputArea;
      const pickerArea = this.$refs.pickerArea.$el;
      const target = event.target;

      if (target === inputArea || target === pickerArea) return;
      if (inputArea.contains(target) || pickerArea.contains(target)) return;
      this.hidePicker();
    },

    showPicker() {
      if (this.pickerType === 'none') return;
      if (this.currentField.autoHidePicker) {
        if (!this.shouldShowPicker) {
          document.addEventListener('click', this.documentClick);
        }
        this.shouldShowPicker = true;
      }
    },
    togglePicker() {
      const inputArea = this.$refs.inputArea;
      inputArea.classList.add('form-control-focused');

      if (this.shouldShowPicker) {
        this.hidePicker();
      } else {
        this.showPicker();
      }
    },

    hidePicker() {
      document.removeEventListener('click', this.documentClick);
      if (this.shouldShowPicker === true) this.valueUpdated();
      this.shouldShowPicker = false;
    },
  },

  computed: {
    component() {
      return this.currentField.pickerType + '-picker';
    },

    palette() {
      return this.currentField.palette || undefined;
    },

    placeholder() {
      if (this.currentField.extraAttributes === undefined) {
        return this.currentField.name;
      }
      return this.currentField.extraAttributes.placeholder || this.currentField.name;
    },

    displayValue() {
      if (!this.value) return '';
      const displayAs = this.currentField.displayAs || 'hex8';
      const value = typeof this.value === 'object' && this.value.hex8 ? this.value.hex8 : this.value;
      const color = tinycolor(value);
      return ['hex', 'hex8', 'rgb', 'hsl'].includes(displayAs) ? color.toString(displayAs) : color.toHex8String();
    },

    saveValue() {
      if (!this.value) return '';

      const saveAs = this.currentField.saveAs || 'hex8';
      const value = typeof this.value === 'object' && this.value.hex8 ? this.value.hex8 : this.value;
      const color = tinycolor(value);
      return ['hex', 'hex8', 'rgb', 'hsl'].includes(saveAs) ? color.toString(saveAs) : color.toHex8String();
    },

    rgbaValue() {
      return this.value ? this.value.toRgbString() : '';
    },
    
  },
};
</script>

<style scoped>
.nc-picker.absolute {
  position: absolute !important;
}
</style>
