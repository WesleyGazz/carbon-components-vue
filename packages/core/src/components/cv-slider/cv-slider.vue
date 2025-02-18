<template>
  <div :class="`cv-slider ${carbonPrefix}--form-item`">
    <label
      v-show="label"
      :for="uid"
      :class="[`${carbonPrefix}--label`, { [`${carbonPrefix}--label--disabled`]: disabled }]"
      :id="labelId"
    >
      {{ label }}
    </label>
    <div :class="`${carbonPrefix}--slider-container`">
      <span :class="`${carbonPrefix}--slider__range-label`">{{ internalMinLabel }}</span>
      <div
        :class="[`${carbonPrefix}--slider`, { [`${carbonPrefix}--slider--disabled`]: disabled }]"
        data-slider
        data-slider-input-box="#slider-input-box"
      >
        <div :class="`${carbonPrefix}--slider__track`" @click="onTrackClick" ref="track"></div>
        <div :class="`${carbonPrefix}--slider__filled-track`" :style="`width: ${percentage};`"></div>
        <div
          :class="[
            `${carbonPrefix}--slider__thumb`,
            {
              [`${carbonPrefix}--slider__thumb--clicked`]: animateClick,
            },
          ]"
          tabindex="0"
          :aria-labelledby="labelId"
          :style="`left: ${percentage};`"
          ref="thumb"
          @keydown.up.right.prevent="onUp"
          @keydown.down.left.prevent="onDown"
          @mousedown="onStartDrag"
        ></div>
        <input
          :id="uid"
          :class="`${carbonPrefix}--slider__input`"
          type="range"
          :step="step"
          :min="min"
          :max="max"
          ref="range"
        />
      </div>
      <span :class="`${carbonPrefix}--slider__range-label`">{{ internalMaxLabel }}</span>
      <input
        type="number"
        :class="[
          `${carbonPrefix}--text-input ${carbonPrefix}--slider-text-input`,
          { [`${carbonPrefix}--text-input--light`]: isLight },
        ]"
        :placeholder="min"
        v-model="internalValue"
        @change="onChange"
        ref="inputBox"
        @keydown.up.prevent="onUp"
        @keydown.down.prevent="onDown"
        :disabled="disabled"
      />
    </div>
  </div>
</template>

<script>
import { uidMixin, themeMixin, carbonPrefixMixin, methodsMixin } from '../../mixins';

export default {
  name: 'CvSlider',
  mixins: [uidMixin, themeMixin, carbonPrefixMixin, methodsMixin({ thumb: ['blur', 'focus'] })],
  props: {
    disabled: Boolean,
    label: String,
    // NOTE: It is not safe to rely on Numbers for non-integer steps
    min: { type: String, default: '0' },
    max: { type: String, default: '100' },
    step: { type: String, default: '1' },
    stepMultiplier: {
      type: String,
      default: '4',
      validator(val) {
        if (val.length) {
          let intMultiplier = parseInt(val);
          if (isNaN(intMultiplier) || intMultiplier < 1) {
            console.warn('cv-slider: multiplier must be >= 1');
            return false;
          }
        }
        return true;
      },
    },
    value: String,
    minLabel: String,
    maxLabel: String,
  },
  model: {
    prop: 'value',
    event: 'modelEvent',
  },
  data() {
    return {
      internalValue: '',
      animateClick: false,
      isDragging: false,
      dragStartX: 0,
      dragStartValue: 0,
      percentage: '0%',
    };
  },
  computed: {
    labelId() {
      return `${this.uid}-label`;
    },
    internalMinLabel() {
      return this.minLabel !== undefined ? this.minLabel : this.getMin();
    },
    internalMaxLabel() {
      return this.maxLabel !== undefined ? this.maxLabel : this.getMax();
    },
    internalMultiplier() {
      let intMultiplier = parseInt(this.stepMultiplier);
      // default to 4 fro multiplier
      return isNaN(intMultiplier) ? 4 : Math.max(intMultiplier, 1);
    },
  },
  mounted() {
    this.$refs.range.value = this.value;
    this.internalValue = this.$refs.range.value;
    this.percentage = `${((this.internalValue - this.getMin()) * 100) / (this.getMax() - this.getMin())}%`;
  },
  watch: {
    value(val) {
      this.setValue(val);
    },
    min() {
      this.$nextTick(() => {
        this.setValue(this.internalValue);
      });
      //      this.internalMin = val && val.length ? val : '0';
    },
    max() {
      this.$nextTick(() => {
        this.setValue(this.internalValue);
      });
    },
    step() {
      this.$nextTick(() => {
        this.setValue(this.internalValue);
      });
    },
  },
  methods: {
    // NOTE: It is not safe to rely on Numbers for non-integer steps
    getMin() {
      if (this.$refs.range) {
        const val = parseFloat(this.$refs.range.min);
        return isNaN(val) ? 0 : val;
      }
      return 0;
    },
    getMax() {
      if (this.$refs.range) {
        const val = parseFloat(this.$refs.range.max);
        return isNaN(val) ? 100 : val;
      }
      return 100;
    },
    getValue() {
      if (this.$refs.range) {
        const val = parseFloat(this.$refs.range.value);
        return isNaN(val) ? 0 : val;
      }
      return (this.getMax() + this.getMin()) / 2;
    },
    getStep() {
      if (this.$refs.range) {
        const val = parseFloat(this.$refs.range.step);
        return isNaN(val) ? 0 : val;
      }
      return 1;
    },

    setValue(newValue) {
      if (this.disabled) return;

      this.$refs.range.value = newValue;
      this.internalValue = this.$refs.range.value;

      this.percentage = `${((this.internalValue - this.getMin()) * 100) / (this.getMax() - this.getMin())}%`;

      this.$emit('modelEvent', this.$refs.range.value);
      this.$emit('change', this.$refs.range.value);
    },
    onChange() {
      let newValue = this.internalValue.length ? parseFloat(this.internalValue) : this.getMin();
      this.setValue(newValue);
    },
    onStartDrag(ev) {
      document.body.addEventListener('mousemove', this.onDrag);
      document.body.addEventListener('mouseup', this.onStopDrag);

      this.dragStartX = ev.clientX;
      this.dragStartValue = this.getValue();
      this.isDragging = true;
    },
    onDrag(ev) {
      if (this.isDragging) {
        // percentage change
        let newValue = (ev.clientX - this.dragStartX) / this.$refs.track.offsetWidth;
        // uncapped new value
        newValue = this.dragStartValue + (this.getMax() - this.getMin()) * newValue;

        this.setValue(newValue, ev);
      }
    },
    onStopDrag() {
      this.isDragging = false;
      document.body.removeEventListener('mousemove', this.onDrag);
      document.body.removeEventListener('mouseup', this.onStopDrag);
    },
    onTrackClick(ev) {
      const afterAnimate = ev => {
        if (ev.propertyName === 'left') {
          this.animateClick = false;
          this.$refs.thumb.removeEventListener('transitionend', afterAnimate);
        }
      };

      let newValue = ev.offsetX / this.$refs.track.offsetWidth;
      newValue = (this.getMax() - this.getMin()) * newValue + this.getMin();
      this.$refs.thumb.addEventListener('transitionend', afterAnimate);
      this.animateClick = true;

      this.setValue(newValue, ev);
    },
    onUp(ev) {
      let curValue = ev.target.type === 'number' ? parseFloat(ev.target.value) : this.getValue();
      let newValue = curValue + (ev.shiftKey ? this.internalMultiplier * this.getStep() : this.getStep());
      this.setValue(newValue, ev);
    },
    onDown(ev) {
      let curValue = ev.target.type === 'number' ? parseFloat(ev.target.value) : this.getValue();
      let newValue = curValue - (ev.shiftKey ? this.internalMultiplier * this.getStep() : this.getStep());
      this.setValue(newValue, ev);
    },
  },
};
</script>
