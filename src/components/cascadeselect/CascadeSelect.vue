<template>
    <div ref="container" :class="containerClass" @click="onClick($event)">
        <div class="p-hidden-accessible">
            <input ref="focusInput" role="combobox" type="text" :id="inputId" :class="inputClass" :style="inputStyle" readonly :disabled="disabled" :tabindex="tabindex" :aria-labelledby="ariaLabelledby" :aria-label="ariaLabel"
                aria-haspopup="listbox" :aria-expanded="overlayVisible" :aria-controls="listId" @focus="onFocus" @blur="onBlur" @keydown="onKeyDown" v-bind="inputProps" />
        </div>
        <span :class="labelClass">
            <slot name="value" :value="modelValue" :placeholder="placeholder">
                {{label}}
            </slot>
        </span>
        <div class="p-cascadeselect-trigger" role="button" aria-haspopup="listbox" :aria-expanded="overlayVisible">
            <slot name="indicator">
                <span :class="dropdownIconClass"></span>
            </slot>
        </div>
        <Portal :appendTo="appendTo">
            <transition name="p-connected-overlay" @enter="onOverlayEnter" @leave="onOverlayLeave" @after-leave="onOverlayAfterLeave">
                <div :ref="overlayRef" :class="panelStyleClass" v-if="overlayVisible" @click="onOverlayClick" v-bind="panelProps">
                    <div class="p-cascadeselect-items-wrapper">
                        <CascadeSelectSub :id="listId" :options="options" :selectionPath="selectionPath"
                            :optionLabel="optionLabel" :optionValue="optionValue" :level="0" :templates="$slots"
                            :optionGroupLabel="optionGroupLabel" :optionGroupChildren="optionGroupChildren"
                            @option-select="onOptionSelect" @optiongroup-select="onOptionGroupSelect" :dirty="dirty" :root="true" />
                    </div>
                </div>
            </transition>
        </Portal>
    </div>
</template>

<script>
import {ConnectedOverlayScrollHandler,ObjectUtils,DomHandler,ZIndexUtils,UniqueComponentId} from 'primevue/utils';
import OverlayEventBus from 'primevue/overlayeventbus';
import CascadeSelectSub from './CascadeSelectSub.vue';
import Portal from 'primevue/portal';

export default {
    name: 'CascadeSelect',
    emits: ['update:modelValue','change','group-change', 'before-show','before-hide','hide','show','focus','blur'],
    data() {
        return {
            selectionPath: null,
            focused: false,
            overlayVisible: false,
            dirty: false
        };
    },
    props: {
        modelValue: null,
        options: Array,
        optionLabel: String,
        optionValue: String,
        optionGroupLabel: String,
        optionGroupChildren: Array,
        placeholder: String,
		disabled: Boolean,
        dataKey: null,
        tabindex: String,
        appendTo: {
            type: String,
            default: 'body'
        },
        loading: {
            type: Boolean,
            default: false
        },
        loadingIcon: {
            type: String,
            default: 'pi pi-spinner pi-spin'
        },
        inputId: null,
        inputClass: null,
        inputStyle: null,
        inputProps: null,
        panelClass: null,
        panelProps: null,
        'aria-labelledby': {
            type: String,
			default: null
        },
        'aria-label': {
            type: String,
            default: null
        }
    },
    outsideClickListener: null,
    scrollHandler: null,
    resizeListener: null,
    overlay: null,
    beforeUnmount() {
        this.unbindOutsideClickListener();
        this.unbindResizeListener();

        if (this.scrollHandler) {
            this.scrollHandler.destroy();
            this.scrollHandler = null;
        }

        if (this.overlay) {
            ZIndexUtils.clear(this.overlay);
            this.overlay = null;
        }
    },
    mounted() {
        this.updateSelectionPath();
    },
    watch: {
        modelValue() {
            this.updateSelectionPath();
        }
    },
    methods: {
        onOptionSelect(event) {
            this.$emit('update:modelValue', event.value);
            this.$emit('change', event);
            this.hide();
            this.$refs.focusInput.focus();
        },
        onOptionGroupSelect(event) {
            this.dirty = true;
            this.$emit('group-change', event);
        },
        getOptionLabel(option) {
            return this.optionLabel ? ObjectUtils.resolveFieldData(option, this.optionLabel) : option;
        },
        getOptionValue(option) {
            return this.optionValue ? ObjectUtils.resolveFieldData(option, this.optionValue) : option;
        },
        getOptionGroupChildren(optionGroup, level) {
            return ObjectUtils.resolveFieldData(optionGroup, this.optionGroupChildren[level]);
        },
        isOptionGroup(option, level) {
            return Object.prototype.hasOwnProperty.call(option, this.optionGroupChildren[level]);
        },
        updateSelectionPath() {
            let path;
            if (this.modelValue != null && this.options) {
                for (let option of this.options) {
                    path = this.findModelOptionInGroup(option, 0);
                    if (path) {
                        break;
                    }
                }
            }

            this.selectionPath = path;
        },
        findModelOptionInGroup(option, level) {
            if (this.isOptionGroup(option, level)) {
                let selectedOption;
                for (let childOption of this.getOptionGroupChildren(option, level)) {
                    selectedOption = this.findModelOptionInGroup(childOption, level + 1);
                    if (selectedOption) {
                        selectedOption.unshift(option);
                        return selectedOption;
                    }
                }
            }
            else if ((ObjectUtils.equals(this.modelValue, this.getOptionValue(option), this.dataKey))) {
                return [option];
            }

            return null;
        },
        show() {
            this.$emit('before-show');
            this.overlayVisible = true;
        },
        hide() {
            this.$emit('before-hide');
            this.overlayVisible = false;
        },
        onFocus(event) {
            this.focused = true;
            this.$emit('focus', event);
        },
        onBlur(event) {
            this.focused = false;
            this.$emit('blur', event);
        },
        onClick(event) {
            if (this.disabled || this.loading) {
                return;
            }

            if (!this.overlay || !this.overlay.contains(event.target)) {
                if (this.overlayVisible)
                    this.hide();
                else
                    this.show();

                this.$refs.focusInput.focus();
            }
        },
        onOverlayEnter(el) {
            ZIndexUtils.set('overlay', el, this.$primevue.config.zIndex.overlay);
            this.alignOverlay();
            this.bindOutsideClickListener();
            this.bindScrollListener();
            this.bindResizeListener();
            this.$emit('show');
        },
        onOverlayLeave() {
            this.unbindOutsideClickListener();
            this.unbindScrollListener();
            this.unbindResizeListener();
            this.$emit('hide');
            this.overlay = null;
            this.dirty = false;
        },
        onOverlayAfterLeave(el) {
            ZIndexUtils.clear(el);
        },
        alignOverlay() {
            if (this.appendTo === 'self') {
                DomHandler.relativePosition(this.overlay, this.$el);
            }
            else {
                this.overlay.style.minWidth = DomHandler.getOuterWidth(this.$el) + 'px';
                DomHandler.absolutePosition(this.overlay, this.$el);
            }
        },
        bindOutsideClickListener() {
            if (!this.outsideClickListener) {
                this.outsideClickListener = (event) => {
                    if (this.overlayVisible && this.overlay && !this.$el.contains(event.target) && !this.overlay.contains(event.target)) {
                        this.hide();
                    }
                };
                document.addEventListener('click', this.outsideClickListener);
            }
        },
        unbindOutsideClickListener() {
            if (this.outsideClickListener) {
                document.removeEventListener('click', this.outsideClickListener);
                this.outsideClickListener = null;
            }
        },
        bindScrollListener() {
            if (!this.scrollHandler) {
                this.scrollHandler = new ConnectedOverlayScrollHandler(this.$refs.container, () => {
                    if (this.overlayVisible) {
                        this.hide();
                    }
                });
            }

            this.scrollHandler.bindScrollListener();
        },
        unbindScrollListener() {
            if (this.scrollHandler) {
                this.scrollHandler.unbindScrollListener();
            }
        },
        bindResizeListener() {
            if (!this.resizeListener) {
                this.resizeListener = () => {
                    if (this.overlayVisible && !DomHandler.isTouchDevice()) {
                        this.hide();
                    }
                };
                window.addEventListener('resize', this.resizeListener);
            }
        },
        unbindResizeListener() {
            if (this.resizeListener) {
                window.removeEventListener('resize', this.resizeListener);
                this.resizeListener = null;
            }
        },
        overlayRef(el) {
            this.overlay = el;
        },
        onKeyDown(event) {
            if (this.disabled || this.loading) {
                event.preventDefault();
                return;
            }

            switch(event.code) {
                case 'Down':
                case 'ArrowDown':
                    if (this.overlayVisible) {
                        if (DomHandler.findSingle(this.overlay, '.p-highlight')) {
                            DomHandler.findSingle(this.overlay, '.p-highlight').focus();
                        }
                        else DomHandler.findSingle(this.overlay, '.p-cascadeselect-item').children[0].focus();
                    }
                    else {
                        this.show();
                    }

                    event.preventDefault();
                break;

                case 'Space':
                case 'Enter':
                    if (this.overlayVisible) {
                        this.hide();
                    }
                    else {
                        this.show();
                    }

                    event.preventDefault();
                break;

                case 'Escape':
                case 'Tab':
                    if (this.overlayVisible) {
                        this.hide();
                        event.preventDefault();
                    }
                break;

                default:
                break;
            }
        },
        onOverlayClick(event) {
            OverlayEventBus.emit('overlay-click', {
                originalEvent: event,
                target: this.$el
            });
        }
    },
    computed: {
        containerClass() {
            return [
                'p-cascadeselect p-component p-inputwrapper',
                {
                    'p-disabled': this.disabled,
                    'p-focus': this.focused,
                    'p-inputwrapper-filled': this.modelValue,
                    'p-inputwrapper-focus': this.focused || this.overlayVisible
                }
            ];
        },
        labelClass() {
            return [
                'p-cascadeselect-label',
                {
                    'p-placeholder': this.label === this.placeholder,
                    'p-cascadeselect-label-empty': !this.$slots['value'] && (this.label === 'p-emptylabel' || this.label.length === 0)
                }
            ];
        },
        label() {
            if (this.selectionPath)
                return this.getOptionLabel(this.selectionPath[this.selectionPath.length - 1]);
            else
                return this.placeholder||'p-emptylabel';
        },
        panelStyleClass() {
            return ['p-cascadeselect-panel p-component', this.panelClass, {
                'p-input-filled': this.$primevue.config.inputStyle === 'filled',
                'p-ripple-disabled': this.$primevue.config.ripple === false
            }];
        },
        dropdownIconClass() {
            return ['p-cascadeselect-trigger-icon', this.loading ? this.loadingIcon : 'pi pi-chevron-down'];
        },
        listId() {
            return this.overlayVisible ? UniqueComponentId() + '_list' : null;
        }
    },
    components: {
        'CascadeSelectSub': CascadeSelectSub,
        'Portal': Portal
    }
}
</script>

<style>
.p-cascadeselect {
    display: inline-flex;
    cursor: pointer;
    position: relative;
    user-select: none;
}

.p-cascadeselect-trigger {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
}

.p-cascadeselect-label {
    display: block;
    white-space: nowrap;
    overflow: hidden;
    flex: 1 1 auto;
    width: 1%;
    text-overflow: ellipsis;
    cursor: pointer;
}

.p-cascadeselect-label-empty {
    overflow: hidden;
    visibility: hidden;
}

.p-cascadeselect .p-cascadeselect-panel {
    min-width: 100%;
}

.p-cascadeselect-panel {
    position: absolute;
    top: 0;
    left: 0;
}

.p-cascadeselect-item {
    cursor: pointer;
    font-weight: normal;
    white-space: nowrap;
}

.p-cascadeselect-item-content {
    display: flex;
    align-items: center;
    overflow: hidden;
    position: relative;
}

.p-cascadeselect-group-icon {
    margin-left: auto;
}

.p-cascadeselect-items {
    margin: 0;
    padding: 0;
    list-style-type: none;
    min-width: 100%;
}

.p-fluid .p-cascadeselect {
    display: flex;
}

.p-fluid .p-cascadeselect .p-cascadeselect-label {
    width: 1%;
}

.p-cascadeselect-sublist {
    position: absolute;
    min-width: 100%;
    z-index: 1;
    display: none;
}

.p-cascadeselect-item-active {
    overflow: visible !important;
}

.p-cascadeselect-item-active > .p-cascadeselect-sublist {
    display: block;
    left: 100%;
    top: 0;
}
</style>
