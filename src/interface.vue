<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
<!-- Original work Copyright (C) 2020 Adrian Dimitrov (https://github.com/dimitrov-adrian/directus-extension-masked-interface) -->
<!-- Modifications Copyright (C) 2026 Ribertec — updated for Directus 10 and 11 compatibility -->

<template>
	<div ref="containerRef" class="masked-interface" :class="[font, { invalid }]">
		<v-input
			:disabled="disabled"
			:placeholder="placeholder"
			:icon-left="iconLeft"
			:icon-right="iconRight"
		/>
	</div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';
import Inputmask from 'inputmask';

const props = withDefaults(
	defineProps<{
		value?: string | null;
		required?: boolean;
		disabled?: boolean;
		placeholder?: string | null;
		iconLeft?: string | null;
		iconRight?: string | null;
		font?: 'sans-serif' | 'serif' | 'monospace';
		storeMasked?: boolean;
		transform?: '' | 'lower' | 'upper' | 'title';
		templateType?: 'url' | 'email' | 'ip' | 'mac' | 'vin' | 'ssn' | 'mask' | 'regex';
		template?: string;
	}>(),
	{
		value: null,
		placeholder: null,
		iconLeft: null,
		iconRight: null,
		font: 'sans-serif',
		storeMasked: false,
		transform: '',
		templateType: 'mask',
		template: '',
	}
);

const emit = defineEmits<{ (e: 'input', value: string | null): void }>();

const containerRef = ref<HTMLDivElement | null>(null);
const inputMaskInstance = ref<Inputmask.Instance | null>(null);
const invalid = ref(false);

const aliasType = ['regex', 'mask'].includes(props.templateType) ? null : props.templateType;
const customArgs: Inputmask.Options = {};
if (['regex', 'mask'].includes(props.templateType)) {
	customArgs[props.templateType] = props.template || '';
}

onMounted(() => {
	const nativeInput = containerRef.value?.querySelector('input');
	if (!nativeInput) return;

	inputMaskInstance.value = Inputmask(aliasType, {
		...customArgs,
		showMaskOnHover: true,
		showMaskOnFocus: true,
		jitMasking: false,
		casing: props.transform as Inputmask.Casing,
		importDataAttributes: false,
		autoUnmask: false,
		clearIncomplete: false,
		tabThrough: true,
		nullable: true,
		oncleared: onClear,
	}).mask(nativeInput);

	setValue(props.value);

	nativeInput.addEventListener('input', propagateInput);
	nativeInput.addEventListener('blur', onBlur);
});

onUnmounted(() => {
	inputMaskInstance.value?.remove();
});

watch(
	() => props.value,
	(newValue) => {
		if (!inputMaskInstance.value) return;
		inputMaskInstance.value.setValue(newValue || '');
		invalid.value = !!(newValue && !inputMaskInstance.value.isValid(newValue));
	}
);

function setValue(value: string | null) {
	if (!inputMaskInstance.value) return;
	inputMaskInstance.value.setValue(value || '');
	invalid.value = inputMaskInstance.value.hasMaskedValue() && !inputMaskInstance.value.isComplete();
}

function onBlur() {
	if (!inputMaskInstance.value) return;
	if (inputMaskInstance.value.isComplete()) return;
	setValue(props.value);
}

function onClear() {
	invalid.value = false;
	emitValue(null);
}

function propagateInput() {
	if (!inputMaskInstance.value) return;
	if (inputMaskInstance.value.isComplete()) {
		invalid.value = false;
		emitValue(getValue());
	} else {
		invalid.value = true;
		emitValue(props.value);
	}
}

function emitValue(value: string | null) {
	if (props.disabled) return;
	if (value === props.value) return;
	emit('input', value ?? null);
}

function getValue() {
	if (!inputMaskInstance.value) return null;
	const unmaskedvalue = inputMaskInstance.value.unmaskedvalue();
	if (!unmaskedvalue) return null;
	if (props.storeMasked) return inputMaskInstance.value.format(unmaskedvalue);
	return unmaskedvalue;
}
</script>

<style lang="scss" scoped>
.masked-interface {
	&.monospace {
		--v-input-font-family: var(--family-monospace);
	}

	&.serif {
		--v-input-font-family: var(--family-serif);
	}

	&.sans-serif {
		--v-input-font-family: var(--family-sans-serif);
	}

	&.invalid :deep(.v-input input) {
		color: var(--danger);
	}
}
</style>
