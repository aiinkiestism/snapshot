<script setup>
const props = defineProps({
  modelValue: [String, Number],
  placeholder: String,
  error: [String, Boolean],
  number: Boolean,
  disabled: Boolean,
  maxlength: [Number, String],
  additionalInputClass: String,
  required: {
    type: Boolean,
    default: true
  }
});

const emit = defineEmits(['update:modelValue']);

function handleInput(e) {
  const input = e.target.value;
  if (props.number) {
    return emit('update:modelValue', !input ? undefined : parseFloat(input));
  }
  emit('update:modelValue', input);
}
</script>

<template>
  <div
    class="border border-skin-border transition-colors bg-transparent text-skin-link rounded-3xl outline-none leading-[46px] text-left w-full mb-2 flex px-3 focus-within:border-skin-link hover:border-skin-link"
    :class="{ '!border-red': error, 'cursor-pointer': $slots.selected }"
  >
    <div class="text-color mr-2 whitespace-nowrap">
      <slot name="label" />
    </div>
    <div
      v-if="$slots.selected"
      class="flex-auto whitespace-nowrap overflow-x-auto"
    >
      <slot name="selected" />
    </div>
    <input
      v-else
      :value="modelValue"
      @input="handleInput"
      :placeholder="placeholder"
      :type="number ? 'number' : 'text'"
      :disabled="disabled"
      class="input flex-auto w-full"
      :class="additionalInputClass"
      :required="required"
      :maxlength="maxlength"
    />
    <slot name="info" />
    <span
      v-if="error"
      v-tippy="{
        content: error
      }"
      class="float-right link-color"
    >
      <Icon name="warning" class="!text-red p-1 block pt-2 mt-[6px] -mr-1" />
    </span>
  </div>
</template>
