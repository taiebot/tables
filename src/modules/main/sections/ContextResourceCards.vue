<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-tabs">
		<div class="context-resource-tabs__bar">
			<div class="context-resource-tabs__list"
				role="tablist"
				:aria-label="t('tables', 'Application resources')"
				@keydown="onKeydown">
				<div v-for="(resource, index) in resources"
					:key="resource.key"
					class="context-resource-tabs__item"
					:class="{ 'context-resource-tabs__item--active': index === activeIndex }"
					role="presentation"
					@mouseenter="showTooltip($event, index)"
					@mouseleave="hideTooltip">
					<button :id="`context-resource-tab-${resource.key}`"
						type="button"
						role="tab"
						class="context-resource-tabs__tab"
						:class="{
							'context-resource-tabs__tab--active': index === activeIndex,
							'context-resource-tabs__tab--with-info': hasDescription(resource),
						}"
						:aria-selected="index === activeIndex ? 'true' : 'false'"
						:tabindex="index === activeIndex ? 0 : -1"
						@focus="onTabFocus($event, index)"
						@blur="hideTooltip"
						@click="select(index)">
						<span class="context-resource-tabs__label">
							<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
						</span>
					</button>

					<!-- Only on tabs whose table has a description -->
					<button v-if="hasDescription(resource)"
						type="button"
						class="context-resource-tabs__info"
						:class="{ 'context-resource-tabs__info--open': descriptionOpen && index === activeIndex }"
						:tabindex="index === activeIndex ? 0 : -1"
						:aria-label="descriptionOpen && index === activeIndex ? t('tables', 'Hide description') : t('tables', 'Show description')"
						:aria-expanded="index === activeIndex ? (descriptionOpen ? 'true' : 'false') : undefined"
						:aria-controls="index === activeIndex ? 'context-resource-description' : undefined"
						@click.stop="toggleDescription(index)">
						<InformationOutline :size="16" />
					</button>
				</div>
			</div>
		</div>

		<!-- Positioned from the hovered tab, outside the tab list -->
		<div v-if="tooltip"
			class="context-resource-tabs__tooltip"
			role="tooltip"
			:style="{ left: tooltip.left + 'px', top: tooltip.top + 'px' }">
			<span class="context-resource-tabs__tooltip-title">{{ tooltip.title }}</span>
			<p v-if="tooltip.preview" class="context-resource-tabs__tooltip-preview">
				{{ tooltip.preview }}
			</p>
		</div>

		<Transition name="tabs-panel">
			<div v-if="descriptionOpen && activeHasDescription"
				id="context-resource-description"
				class="context-resource-tabs__panel">
				<div class="context-resource-tabs__panel-header">
					<h3 class="context-resource-tabs__panel-title">
						<span v-if="activeResource.emoji">{{ activeResource.emoji }}&nbsp;</span>{{ activeResource.title }}
					</h3>
					<button type="button"
						class="context-resource-tabs__close"
						:aria-label="t('tables', 'Hide description')"
						@click="descriptionOpen = false">
						<Close :size="20" />
					</button>
				</div>
				<TableDescription :key="activeResource.key"
					class="context-resource-tabs__panel-text"
					:description="activeResource.description"
					:read-only="true" />
			</div>
		</Transition>
	</div>
</template>

<script>
import TableDescription from './TableDescription.vue'
import InformationOutline from 'vue-material-design-icons/InformationOutline.vue'
import Close from 'vue-material-design-icons/Close.vue'

const TOOLTIP_DELAY = 400
const TOOLTIP_MAX_WIDTH = 320
const TOOLTIP_GAP = 4
const PREVIEW_MAX_CHARS = 180

export default {
	name: 'ContextResourceCards',

	components: {
		TableDescription,
		InformationOutline,
		Close,
	},

	props: {
		resources: {
			type: Array,
			required: true,
		},
		activeIndex: {
			type: Number,
			default: 0,
		},
	},

	// `height-change`: the tab row wraps onto several lines when there are
	// many resources, so its height varies. The parent uses it as the `top`
	// offset of the table's sticky options bar.
	emits: ['update:active-index', 'height-change'],

	data() {
		return {
			// The description is always the one of the active resource, so a
			// single flag is enough.
			descriptionOpen: false,
			// Set when the (i) of an inactive tab is clicked: the description
			// opens once the table below has switched to that tab.
			openAfterSwitch: false,
			tooltip: null,
			tooltipTimer: null,
		}
	},

	computed: {
		activeResource() {
			return this.resources[this.activeIndex] ?? null
		},

		activeHasDescription() {
			return this.hasDescription(this.activeResource)
		},
	},

	watch: {
		activeIndex() {
			this.descriptionOpen = this.openAfterSwitch && this.hasDescription(this.activeResource)
			this.openAfterSwitch = false
			this.hideTooltip()
		},
	},

	created() {
		// Not reactive on purpose (a ResizeObserver must not be proxied).
		this.resizeObserver = null
	},

	mounted() {
		if (typeof ResizeObserver !== 'undefined') {
			this.resizeObserver = new ResizeObserver(() => this.reportHeight())
			this.resizeObserver.observe(this.$el)
		}
		this.reportHeight()
	},

	// Vue 2 and Vue 3 name this hook differently; stopObserving is idempotent.
	beforeDestroy() {
		this.stopObserving()
	},

	beforeUnmount() {
		this.stopObserving()
	},

	methods: {
		reportHeight() {
			this.$emit('height-change', this.$el.offsetHeight)
		},

		stopObserving() {
			this.resizeObserver?.disconnect()
			this.resizeObserver = null
			clearTimeout(this.tooltipTimer)
		},

		hasDescription(resource) {
			const description = resource?.description
			if (typeof description === 'string') {
				return description.trim() !== ''
			}
			return !!description
		},

		select(index) {
			this.hideTooltip()
			if (index !== this.activeIndex) {
				this.$emit('update:active-index', index)
			}
		},

		toggleDescription(index) {
			this.hideTooltip()
			if (index === this.activeIndex) {
				this.descriptionOpen = !this.descriptionOpen
				return
			}
			// The table below must follow the tab whose description is shown.
			this.openAfterSwitch = true
			this.$emit('update:active-index', index)
			// If the parent did not switch tabs, do not leave the flag armed.
			this.$nextTick(() => {
				this.openAfterSwitch = false
			})
		},

		// Short plain-text excerpt of the (markdown) description.
		previewText(description) {
			const text = String(description)
				.replace(/!?\[([^\]]*)\]\([^)]*\)/g, '$1')
				.replace(/https?:\/\/\S+/g, '')
				.replace(/^\s*[-+*]\s+/gm, '')
				.replace(/[`*_~>#|]/g, '')
				.replace(/\s+/g, ' ')
				.trim()
			return text.length > PREVIEW_MAX_CHARS
				? text.slice(0, PREVIEW_MAX_CHARS).trimEnd() + '…'
				: text
		},

		onTabFocus(event, index) {
			// Mouse clicks also focus the tab: only show it for keyboard focus.
			if (event.currentTarget.matches(':focus-visible')) {
				this.showTooltip(event, index, true)
			}
		},

		// Shown when the title is cut off and/or the table has a description.
		showTooltip(event, index, immediate = false) {
			const item = event.currentTarget.closest('.context-resource-tabs__item')
			const resource = this.resources[index]
			if (!item || !resource) {
				return
			}
			const label = item.querySelector('.context-resource-tabs__label')
			const truncated = !!label && label.scrollWidth > label.clientWidth
			const described = this.hasDescription(resource)
			if (!truncated && !described) {
				this.hideTooltip()
				return
			}

			clearTimeout(this.tooltipTimer)
			this.tooltipTimer = setTimeout(() => {
				const rootRect = this.$el.getBoundingClientRect()
				const itemRect = item.getBoundingClientRect()
				this.tooltip = {
					// Physical coordinates on both sides, so RTL works too.
					left: Math.max(8, Math.min(itemRect.left - rootRect.left, rootRect.width - TOOLTIP_MAX_WIDTH - 8)),
					// Right under the hovered tab, whichever line it is on.
					top: itemRect.bottom - rootRect.top + TOOLTIP_GAP,
					title: `${resource.emoji ? resource.emoji + ' ' : ''}${resource.title}`,
					preview: described ? this.previewText(resource.description) : '',
				}
			}, immediate ? 0 : TOOLTIP_DELAY)
		},

		hideTooltip() {
			clearTimeout(this.tooltipTimer)
			this.tooltip = null
		},

		// Arrow keys move focus (in DOM order, so line by line), Enter/Space
		// select: browsing the tabs does not reload a table on every keypress.
		onKeydown(event) {
			if (event.key === 'Escape') {
				this.hideTooltip()
				return
			}
			if (!['ArrowLeft', 'ArrowRight', 'Home', 'End'].includes(event.key)) {
				return
			}
			const tabs = [...event.currentTarget.querySelectorAll('[role="tab"]')]
			const current = tabs.indexOf(event.target.closest('[role="tab"]'))
			if (current === -1) {
				return
			}
			const rtl = getComputedStyle(event.currentTarget).direction === 'rtl'
			let next = current
			if (event.key === 'Home') {
				next = 0
			} else if (event.key === 'End') {
				next = tabs.length - 1
			} else {
				const forward = (event.key === 'ArrowRight') !== rtl
				next = (current + (forward ? 1 : -1) + tabs.length) % tabs.length
			}
			event.preventDefault()
			tabs[next].focus()
		},
	},
}
</script>

<style scoped lang="scss">
// One line of tabs = 47px + the 1px bottom border of the root = 48px.
$tab-row-h: 47px;

@mixin icon-button($size) {
	appearance: none;
	flex-shrink: 0;
	display: flex;
	align-items: center;
	justify-content: center;
	width: $size;
	height: $size;
	padding: 0;
	cursor: pointer;
	color: var(--color-text-maxcontrast);
	background: transparent;
	border: none;
	border-radius: 50%;
	transition: color var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

	&:hover {
		color: var(--color-main-text);
		background-color: var(--color-background-hover);
	}

	&:focus {
		outline: none;
	}

	&:focus-visible {
		outline: 2px solid var(--color-primary-element);
		outline-offset: 2px;
	}
}

.context-resource-tabs {
	// The height is not fixed: tabs wrap onto extra lines. It is reported to
	// the parent (`height-change`) which exposes it as `--tbl-tabs-h`, the
	// `top` of the table's sticky options bar.
	position: sticky;
	top: 0;
	inset-inline-start: 0; // stays in view when a wide table scrolls sideways
	// Must stay above the table's own sticky rows (title, options bar, header),
	// otherwise the description overlay is painted underneath them.
	z-index: 100;
	width: var(--app-content-width, 100%);
	box-sizing: border-box;
	background-color: var(--color-main-background);
	border-bottom: 1px solid var(--color-border);

	&__bar {
		padding-inline: 20px;
		box-sizing: border-box;
	}

	&__list {
		display: flex;
		flex-wrap: wrap;
		gap: 0 var(--default-grid-baseline, 4px);
	}

	// One tab = the tab button + its (i) button, side by side (a button
	// cannot contain another button). The underline lives on the item.
	&__item {
		flex: 0 0 auto;
		display: flex;
		align-items: center;
		height: $tab-row-h;
		max-width: calc(65 * var(--default-grid-baseline, 4px));
		box-sizing: border-box;
		border-bottom: 3px solid transparent;
		transition: border-color var(--animation-quick, 100ms) ease;

		&:hover {
			border-bottom-color: var(--color-border-maxcontrast);
		}

		&--active,
		&--active:hover {
			border-bottom-color: var(--color-primary-element);
		}
	}

	&__tab {
		appearance: none;
		flex: 1 1 auto;
		min-width: 0;
		box-sizing: border-box;
		display: flex;
		align-items: center;
		align-self: stretch;
		margin: 0;
		padding: 0 calc(4 * var(--default-grid-baseline, 4px));
		font: inherit;
		color: var(--color-text-maxcontrast);
		cursor: pointer;
		user-select: none;
		background: transparent;
		border: none;
		border-radius: 0;
		transition: color var(--animation-quick, 100ms) ease;

		&--with-info {
			padding-inline-end: var(--default-grid-baseline, 4px);
		}

		// Global `button:hover` styles must not tint the tab background.
		&:hover {
			color: var(--color-main-text);
			background: transparent !important;
		}

		&:focus {
			outline: none;
		}

		&:focus:not(:focus-visible) {
			background: transparent !important;
			box-shadow: none !important;
		}

		&:focus-visible {
			outline: 2px solid var(--color-primary-element);
			outline-offset: -2px;
		}

		&--active,
		&--active:hover {
			color: var(--color-main-text);
		}
	}

	&__label {
		min-width: 0;
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}

	// (i) sitting on the tab itself
	&__info {
		@include icon-button(24px);
		margin-inline-end: calc(2 * var(--default-grid-baseline, 4px));

		&--open,
		&--open:hover {
			color: var(--color-primary-element-text);
			background-color: var(--color-primary-element);
		}
	}

	// × inside the description panel
	&__close {
		@include icon-button(32px);
	}

	// Hover / focus tooltip: full title + short description preview.
	&__tooltip {
		position: absolute;
		z-index: 3;
		box-sizing: border-box;
		width: max-content;
		max-width: 320px;
		padding: calc(2 * var(--default-grid-baseline, 4px)) calc(3 * var(--default-grid-baseline, 4px));
		background-color: var(--color-main-background);
		border: 1px solid var(--color-border);
		border-radius: var(--border-radius-large, 12px);
		box-shadow: 0 2px 8px var(--color-box-shadow, rgba(0, 0, 0, 0.2));
		pointer-events: none;
	}

	&__tooltip-title {
		display: block;
		font-weight: bold;
		overflow-wrap: anywhere;
	}

	&__tooltip-preview {
		display: -webkit-box;
		-webkit-box-orient: vertical;
		-webkit-line-clamp: 4;
		line-clamp: 4;
		margin: var(--default-grid-baseline, 4px) 0 0;
		overflow: hidden;
		overflow-wrap: anywhere;
		color: var(--color-text-maxcontrast);
	}

	// Overlays the table instead of pushing it, so the tab row keeps a
	// constant height and the table's sticky bar offset stays valid.
	&__panel {
		position: absolute;
		inset-inline: 0;
		top: 100%;
		z-index: 2;
		box-sizing: border-box;
		max-height: 60vh;
		overflow-x: hidden;
		overflow-y: auto;
		padding: calc(4 * var(--default-grid-baseline, 4px)) 20px calc(5 * var(--default-grid-baseline, 4px));
		background-color: var(--color-main-background);
		border-bottom: 1px solid var(--color-border);
		box-shadow: 0 6px 12px var(--color-box-shadow, rgba(0, 0, 0, 0.15));

		:deep(.element-description) {
			width: 100%;
			max-width: 100%;
			padding-inline: 0 !important;
		}
	}

	&__panel-header {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: calc(3 * var(--default-grid-baseline, 4px));
		margin-bottom: calc(3 * var(--default-grid-baseline, 4px));
	}

	&__panel-title {
		margin: 0;
		overflow-wrap: anywhere;
	}

	&__panel-text {
		overflow-wrap: anywhere;
	}
}

.tabs-panel-enter-active,
.tabs-panel-leave-active {
	transition: opacity var(--animation-quick, 150ms) ease, transform var(--animation-quick, 150ms) ease;
}

.tabs-panel-enter-from,
.tabs-panel-enter,
.tabs-panel-leave-to {
	opacity: 0;
	transform: translateY(-4px);
}

@media (prefers-reduced-motion: reduce) {
	.tabs-panel-enter-active,
	.tabs-panel-leave-active,
	.context-resource-tabs__item,
	.context-resource-tabs__tab,
	.context-resource-tabs__info,
	.context-resource-tabs__close {
		transition: none;
	}
}
</style>
