<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-tabs">
		<div class="context-resource-tabs__bar">
			<div ref="list"
				class="context-resource-tabs__list"
				role="tablist"
				:aria-label="t('tables', 'Application resources')"
				@keydown="onKeydown">
				<button v-for="(resource, index) in resources"
					:id="`context-resource-tab-${resource.key}`"
					:key="resource.key"
					type="button"
					role="tab"
					class="context-resource-tabs__tab"
					:class="{ 'context-resource-tabs__tab--active': index === activeIndex }"
					:aria-selected="index === activeIndex ? 'true' : 'false'"
					:tabindex="index === activeIndex ? 0 : -1"
					@mouseenter="updateTooltip($event, resource)"
					@focus="updateTooltip($event, resource)"
					@click="select(index)">
					<span class="context-resource-tabs__label">
						<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
					</span>
				</button>
			</div>

			<!-- Only shown when the table below has a description -->
			<button v-if="activeHasDescription"
				type="button"
				class="context-resource-tabs__info"
				:class="{ 'context-resource-tabs__info--open': descriptionOpen }"
				:aria-label="descriptionOpen ? t('tables', 'Hide description') : t('tables', 'Show description')"
				:aria-expanded="descriptionOpen ? 'true' : 'false'"
				aria-controls="context-resource-description"
				@click="descriptionOpen = !descriptionOpen">
				<InformationOutline :size="20" />
			</button>
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
						class="context-resource-tabs__info context-resource-tabs__info--open"
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

	emits: ['update:active-index'],

	data() {
		return {
			// The description is always the one of the active resource, so a
			// single flag is enough (no separate index to keep in sync).
			descriptionOpen: false,
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
		// Switching tab closes the description of the previous resource.
		activeIndex() {
			this.descriptionOpen = false
			this.$nextTick(() => this.scrollActiveTabIntoView())
		},
	},

	mounted() {
		this.scrollActiveTabIntoView()
	},

	methods: {
		hasDescription(resource) {
			const description = resource?.description
			if (typeof description === 'string') {
				return description.trim() !== ''
			}
			return !!description
		},

		select(index) {
			if (index !== this.activeIndex) {
				this.$emit('update:active-index', index)
			}
		},

		// Native tooltip, only when the label is actually cut off.
		updateTooltip(event, resource) {
			const tab = event.currentTarget
			const label = tab.querySelector('.context-resource-tabs__label')
			const truncated = !!label && label.scrollWidth > label.clientWidth
			tab.title = truncated
				? `${resource.emoji ? resource.emoji + ' ' : ''}${resource.title}`
				: ''
		},

		// Arrow keys move focus, Enter/Space select (manual activation, so
		// browsing the tabs does not reload a table on every keypress).
		onKeydown(event) {
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

		scrollActiveTabIntoView() {
			const active = this.$refs.list?.querySelector('[role="tab"][aria-selected="true"]')
			active?.scrollIntoView({ block: 'nearest', inline: 'nearest' })
		},
	},
}
</script>

<style scoped lang="scss">
.context-resource-tabs {
	// `--tbl-tabs-h` is defined once in Context.vue (on `.resources`) so the
	// sticky options bar of the table below uses the very same value as `top`.
	position: sticky;
	top: 0;
	inset-inline-start: 0; // stays in view when a wide table scrolls sideways
	z-index: 12;
	width: var(--app-content-width, 100%);
	height: var(--tbl-tabs-h, 48px);
	box-sizing: border-box;
	background-color: var(--color-main-background);
	border-bottom: 1px solid var(--color-border);

	&__bar {
		display: flex;
		align-items: stretch;
		gap: calc(2 * var(--default-grid-baseline, 4px));
		height: 100%;
		padding-inline: 20px;
		box-sizing: border-box;
	}

	&__list {
		display: flex;
		flex: 1 1 auto;
		min-width: 0;
		gap: var(--default-grid-baseline, 4px);
		overflow-x: auto;
		overflow-y: hidden;
		scrollbar-width: thin;
	}

	&__tab {
		appearance: none;
		flex: 0 0 auto;
		box-sizing: border-box;
		display: flex;
		align-items: center;
		max-width: calc(65 * var(--default-grid-baseline, 4px));
		margin: 0;
		padding: 0 calc(4 * var(--default-grid-baseline, 4px));
		font: inherit;
		color: var(--color-text-maxcontrast);
		cursor: pointer;
		user-select: none;
		background: transparent;
		border: none;
		border-bottom: 3px solid transparent;
		border-radius: 0;
		transition: color var(--animation-quick, 100ms) ease, border-color var(--animation-quick, 100ms) ease;

		// Global `button:hover` styles must not tint the tab background.
		&:hover {
			color: var(--color-main-text);
			background: transparent !important;
			border-bottom-color: var(--color-border-maxcontrast);
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
			border-bottom-color: var(--color-primary-element);
		}
	}

	&__label {
		min-width: 0;
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}

	&__info {
		appearance: none;
		align-self: center;
		flex-shrink: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		width: 32px;
		height: 32px;
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

		&--open {
			color: var(--color-primary-element-text);
			background-color: var(--color-primary-element);

			&:hover {
				color: var(--color-primary-element-text);
				background-color: var(--color-primary-element-hover);
			}
		}
	}

	// Overlays the table instead of pushing it, so the tab row keeps a
	// constant height and the table's sticky bar offset stays valid.
	&__panel {
		position: absolute;
		inset-inline: 0;
		top: 100%;
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
	.context-resource-tabs__tab,
	.context-resource-tabs__info {
		transition: none;
	}
}
</style>
