<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-cards" role="tablist" :aria-label="t('tables', 'Application resources')">
		<Transition name="card-flip" mode="out-in">
			<div v-if="flippedIndex === null" key="grid" class="context-resource-cards__grid">
				<div v-for="(resource, index) in resources"
					:key="resource.key"
					class="context-resource-cards__card"
					:class="{ 'context-resource-cards__card--active': index === activeIndex }">
					<button :id="`context-resource-card-${resource.key}`"
						type="button"
						role="tab"
						class="context-resource-cards__card-button"
						:aria-selected="index === activeIndex"
						@click="$emit('update:active-index', index)">
						<h3 class="context-resource-cards__title">
							<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
						</h3>
					</button>

					<button v-if="resource.description"
						type="button"
						class="context-resource-cards__info"
						:aria-label="t('tables', 'Show description')"
						@click.stop="flippedIndex = index">
						<span class="context-resource-cards__info-badge" aria-hidden="true">i</span>
					</button>
				</div>
			</div>

			<div v-else key="detail" class="context-resource-cards__detail">
				<div ref="detailPanel" class="context-resource-cards__detail-panel">
					<div class="context-resource-cards__detail-header">
						<h3 class="context-resource-cards__title">
							<span v-if="resources[flippedIndex].emoji">{{ resources[flippedIndex].emoji }}&nbsp;</span>{{ resources[flippedIndex].title }}
						</h3>
						<button type="button"
							class="context-resource-cards__info context-resource-cards__info--active"
							:aria-label="t('tables', 'Back to cards')"
							@click="flippedIndex = null">
							<span class="context-resource-cards__info-badge" aria-hidden="true">✕</span>
						</button>
					</div>
					<TableDescription class="context-resource-cards__detail-text" :description="resources[flippedIndex].description" :read-only="true" />
				</div>
			</div>
		</Transition>
	</div>
</template>

<script>
import TableDescription from './TableDescription.vue'

export default {
	name: 'ContextResourceCards',

	components: {
		TableDescription,
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
			flippedIndex: null,
		}
	},

	watch: {
		// Switching which resource is open below should drop back to the grid.
		activeIndex() {
			this.flippedIndex = null
		},
		flippedIndex(newVal) {
			if (newVal !== null) {
				// Defer so the click that opened the panel isn't immediately
				// seen as an "outside" click by this same listener.
				this.$nextTick(() => {
					document.addEventListener('click', this.handleOutsideClick, true)
				})
				window.addEventListener('scroll', this.closeDetail, { passive: true })
				document.addEventListener('keydown', this.handleEscape)
			} else {
				document.removeEventListener('click', this.handleOutsideClick, true)
				window.removeEventListener('scroll', this.closeDetail)
				document.removeEventListener('keydown', this.handleEscape)
			}
		},
	},

	beforeUnmount() {
		document.removeEventListener('click', this.handleOutsideClick, true)
		window.removeEventListener('scroll', this.closeDetail)
		document.removeEventListener('keydown', this.handleEscape)
	},

	methods: {
		closeDetail() {
			this.flippedIndex = null
		},
		handleOutsideClick(event) {
			if (this.$refs.detailPanel && !this.$refs.detailPanel.contains(event.target)) {
				this.closeDetail()
			}
		},
		handleEscape(event) {
			if (event.key === 'Escape') {
				this.closeDetail()
			}
		},
	},
}
</script>

<style scoped lang="scss">
.context-resource-cards {
	width: var(--app-content-width, 100%);
	padding: calc(4 * var(--default-grid-baseline, 4px)) 20px;
	background-color: var(--color-main-background);
	border-bottom: 1px solid var(--color-border);
	box-sizing: border-box;

	&__grid {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
		gap: calc(3 * var(--default-grid-baseline, 4px));
	}

	&__card {
		position: relative;
		box-sizing: border-box;
		height: calc(30 * var(--default-grid-baseline, 4px));
		overflow-y: auto;
		overflow-x: hidden;
		background-color: var(--color-main-background);
		border: 2px solid var(--color-border);
		border-radius: var(--border-radius-large, 12px);
		padding: calc(5 * var(--default-grid-baseline, 4px));
		transition: border-color var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

		&:hover {
			border-color: var(--color-primary-element);
		}

		&--active {
			border-color: var(--color-primary-element);
			background-color: var(--color-primary-element-light);
		}
	}

	&__card-button {
		appearance: none;
		box-sizing: border-box;
		width: 100%;
		text-align: start;
		cursor: pointer;
		user-select: none;
		background: transparent;
		border: none;
		padding: 0;
		margin: 0;
		font: inherit;
		color: inherit;

		&:focus {
			outline: none;
		}

		&:focus-visible {
			outline: 2px solid var(--color-primary-element);
			outline-offset: 2px;
		}
	}

	&__title {
		margin: 0;
		padding-inline-end: calc(6 * var(--default-grid-baseline, 4px));
		overflow-wrap: anywhere;
	}

	&__info {
		appearance: none;
		position: absolute;
		top: calc(3 * var(--default-grid-baseline, 4px));
		inset-inline-end: calc(3 * var(--default-grid-baseline, 4px));
		display: flex;
		align-items: center;
		justify-content: center;
		width: 24px;
		height: 24px;
		background: transparent;
		border: none;
		border-radius: 50%;
		cursor: pointer;
		color: var(--color-text-maxcontrast);
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

		// Used as the "back" button inside the detail panel header instead of
		// a floating corner badge.
		&--active {
			position: static;
			flex-shrink: 0;
		}
	}

	&__info-badge {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 18px;
		height: 18px;
		border: 1.5px solid currentColor;
		border-radius: 50%;
		font-size: 11px;
		line-height: 1;
		font-style: italic;
		font-weight: 600;
		font-family: Georgia, 'Times New Roman', serif;
	}

	&__detail {
		width: 100%;
		box-sizing: border-box;
	}

	&__detail-panel {
		width: 100%;
		box-sizing: border-box;
		background-color: var(--color-main-background);
		border: 2px solid var(--color-primary-element);
		border-radius: var(--border-radius-large, 12px);
		padding: calc(5 * var(--default-grid-baseline, 4px));
		overflow-x: hidden;
	}

	&__detail-header {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: calc(3 * var(--default-grid-baseline, 4px));
		margin-bottom: calc(3 * var(--default-grid-baseline, 4px));
	}

	&__detail-text {
		overflow-wrap: anywhere;
	}
}

.card-flip-enter-active,
.card-flip-leave-active {
	transition: opacity var(--animation-quick, 150ms) ease, transform var(--animation-quick, 150ms) ease;
}

.card-flip-enter-from,
.card-flip-leave-to {
	opacity: 0;
	transform: scaleY(0.98);
}
</style>
