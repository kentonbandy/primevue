<template>
    <div :class="containerClass">
        <span class="p-hidden-accessible" v-if="cancel">
            <label :for="labelFor(0)">{{labelText(0)}}</label>
            <input type="radio" :id="labelFor(0)" value="0" :name="name" :checked="modelValue === 0" :disabled="disabled" :readonly="readonly" @focus="onFocus($event, 0)" @blur="onBlur" @keydown="onKeyDown($event, 0)">
        </span>
        <span :class="['p-rating-icon p-rating-cancel pi pi-ban', {'p-focus': focusIndex === 0}]" v-if="cancel" @click="onCancelClick" @keydown="onKeyDown"></span>

        <template :key="i" v-for="i in stars">
            <span class="p-hidden-accessible">
                <label :for="labelFor(i)">{{labelText(i)}}</label>
                <input type="radio" :id="labelFor(i)" :value="i" :name="name" :checked="modelValue === i" :disabled="disabled" :readonly="readonly" @focus="onFocus($event, i)" @blur="onBlur" @keydown="onKeyDown($event,i)">
            </span>
            <span :class="['p-rating-icon', {'pi pi-star': (i > modelValue), 'pi pi-star-fill': (i <= modelValue), 'p-focus': i === focusIndex}]" @click="onStarClick($event,i)"></span>
        </template>
    </div>
</template>

<script>
export default {
    name: 'Rating',
    emits: ['update:modelValue', 'change', 'focus', 'blur'],
    props: {
        modelValue: {
            type: Number,
            default: null
        },
        name: {
            type: String,
            default: null
        },
		disabled: {
            type: Boolean,
            default: false
        },
		readonly: {
            type: Boolean,
            default: false
        },
        stars: {
            type: Number,
            default: 5
        },
        cancel: {
            type: Boolean,
            default: true
        }
    },
    data() {
        return {
            focusIndex: null
        };
    },
    methods: {
        onStarClick(event, value) {
            if (!this.readonly && !this.disabled) {
                this.updateModel(event, value);
                this.focusIndex = value;
            }
        },
        onKeyDown(event, value) {
            if (event.code === 'Space') {
                this.updateModel(event, value);
            }
            if (event.code === 'Tab') {
                this.focusIndex = null;
            }
        },
        onFocus(event, index) {
            if (!this.readonly) {
                if (this.modelValue === null && this.focusIndex === null) {
                    this.cancel ? this.focusIndex = 0 : this.focusIndex = 1;
                }
                else if (this.modelValue !== null && this.focusIndex === null) {
                    this.focusIndex = this.modelValue;
                    this.updateModel(event, this.modelValue);
                }
                else {
                    this.focusIndex = index;
                    this.updateModel(event, index);
                }

                this.$emit('focus', event);
            }
        },
        onBlur(event) {
            this.$emit('blur', event);
        },
        onCancelClick(event) {
            if (!this.readonly && !this.disabled) {
                this.updateModel(event, null);
            }
        },
        updateModel(event, value) {
            this.$emit('update:modelValue', value);
            this.$emit('change', {
                originalEvent: event,
                value: value
            });
        },
        labelFor(index) {
            return this.name + '_' + index;
        },
        labelText(index) {
            return index === 0 ? this.$primevue.config.locale.aria.noneStars : index === 1
                ? index + ' ' + this.$primevue.config.locale.aria.star : index + ' ' + this.$primevue.config.locale.aria.stars;
        }
    },
    computed: {
        containerClass() {
            return [
                'p-rating',
                {
                    'p-readonly': this.readonly,
                    'p-disabled': this.disabled
                }
            ];
        }
    }
}
</script>

<style>
.p-rating-icon {
    cursor: pointer;
}

.p-rating.p-rating-readonly .p-rating-icon {
    cursor: default;
}

.p-rating:not(.p-disabled) .p-rating-icon.p-focus {
    outline: 0 none;
    outline-offset: 0;
    box-shadow: 0 0 0 0.2rem #BFDBFE;
    border-color: #3B82F6;
}
</style>
