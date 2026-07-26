<!--
  - SPDX-FileCopyrightText: 2023 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div v-if="richObject" class="tables-content-widget" data-cy="contentReferenceWidget">
		<div class="header">
			<h2>
				<NcLoadingIcon v-if="!loaded" :size="30" />
				<span v-else>{{ richObject.emoji }}</span> {{ richObject.title }}
			</h2>
			<Options
				:config="tablePermissions"
				:rows="filteredRows"
				:show-options="true"
				@create-row="createRow"
				@set-search-string="search" />
		</div>
		<div v-if="loaded && rows.length > 0" class="nc-table">
			<NcTable
				:rows="filteredRows"
				:columns="columns"
				:element-id="richObject.id"
				:is-view="isView"
				v-bind="tablePermissions"
				@edit-row="editRow"
				@copy-row="copyRow"
				@delete-row="deleteRow" />
		</div>
		<CreateRow
			:columns="columns"
			:is-view="isView"
			:element-id="richObject.id"
			:show-modal="showCopyRow"
			:prefill-data="copyPrefillData"
			@close="showCopyRow = false; copyPrefillData = null" />
		<DeleteRows
			v-if="rowToDelete !== null"
			:rows-to-delete="[rowToDelete]"
			:element-id="richObject.id"
			:is-view="isView"
			@cancel="rowToDelete = null" />
	</div>
</template>
 
<script>
import NcTable from '../shared/components/ncTable/NcTable.vue'
import Options from '../shared/components/ncTable/sections/Options.vue'
import CreateRow from '../modules/modals/CreateRow.vue'
import DeleteRows from '../modules/modals/DeleteRows.vue'
import permissionsMixin from '../shared/components/ncTable/mixins/permissionsMixin.js'
import { NcLoadingIcon } from '@nextcloud/vue'
import { useResizeObserver } from '@vueuse/core'
import { spawnDialog } from '@nextcloud/vue/functions/dialog'
import { useTablesStore } from '../store/store.js'
import { useDataStore } from '../store/data.js'
 
export default {
 
	components: {
		NcTable,
		Options,
		CreateRow,
		DeleteRows,
		NcLoadingIcon,
	},
 
	mixins: [permissionsMixin],
 
	props: {
		richObjectType: {
			type: String,
			default: '',
		},
		richObject: {
			type: Object,
			default: null,
		},
		accessible: {
			type: Boolean,
			default: true,
		},
	},
 
	data() {
		return {
			searchExp: null,
			showCopyRow: false,
			copyPrefillData: null,
			rowToDelete: null,
			tablesStore: null,
			dataStore: null,
			// True once the initial backend fetch for rows+columns has completed.
			// Nothing in this component reads richObject.rows/richObject.columns
			// as data any more -- they're only ever used for id/type/title/emoji.
			loaded: false,
		}
	},
 
	computed: {
		isView() {
			return Boolean(this.richObject?.type)
		},
		tablePermissions() {
			return {
				canCreateRows: this.canCreateRowInElement(this.richObject),
				canReadRows: true,
				canEditRows: this.canUpdateData(this.richObject),
				canDeleteRows: this.canDeleteData(this.richObject),
				canCreateColumns: false,
				canEditColumns: false,
				canDeleteColumns: false,
				canDeleteTable: false,
				canSelectRows: false,
				canHideColumns: false,
				canFilter: false,
				showActions: this.canCreateRowInElement(this.richObject) || this.canUpdateData(this.richObject) || this.canDeleteData(this.richObject),
			}
		},
		filteredRows() {
			if (this.searchExp) {
				return this.rows.filter(row => {
					return row.data.some(column => {
						const col = String(column.value)
						return col.search(this.searchExp) >= 0
					})
				})
			} else {
				return this.rows
			}
		},
		// Store is the ONLY source of truth for rows/columns. No fallback to
		// richObject.rows / richObject.columns -- those are just a snapshot frozen
		// at reference-resolution time and can be arbitrarily stale (old column
		// order, missing newly created rows, etc.).
		rows() {
			return this.dataStore ? this.dataStore.getRows(this.isView, this.richObject.id) : []
		},
		columns() {
			return this.dataStore ? this.dataStore.getColumns(this.isView, this.richObject.id) : []
		},
	},
 
	watch: {
		// Covers "refresh": if the parent re-resolves the reference and swaps in
		// a new richObject (same or different id) without remounting this
		// component, refetch from the backend rather than trusting whatever
		// richObject now contains.
		richObject: {
			deep: true,
			handler() {
				this.reload()
			},
		},
	},
 
	async mounted() {
		useResizeObserver(this.$el, (entries) => {
			const entry = entries[0]
			const { width } = entry.contentRect
			this.$el.style.setProperty('--widget-content-width', `${width}px`)
		})
 
		this.tablesStore = useTablesStore()
		this.dataStore = useDataStore()
 
		await this.reload()
	},
 
	methods: {
		// Builds the { tableId } or { viewId } payload loadRowsFromBE expects,
		// based on whether richObject is a table or a view.
		elementIdPayload() {
			return this.isView
				? { viewId: this.richObject.id }
				: { tableId: this.richObject.id }
		},
		search(searchString) {
			this.searchExp = (searchString !== '')
				? new RegExp(searchString.trim(), 'ig')
				: null
		},
		async createRow() {
			const { default: CreateRow } = await import('../modules/modals/CreateRow.vue')
			spawnDialog(CreateRow, {
				showModal: true,
				columns: this.columns,
				isView: this.isView,
				elementId: this.richObject.id,
			}, async () => {
				// Reload rows from the backend to get the latest data
				await this.dataStore.loadRowsFromBE(this.elementIdPayload())
			})
		},
		async editRow(rowId) {
			const { default: EditRow } = await import('../modules/modals/EditRow.vue')
			spawnDialog(EditRow, {
				showModal: true,
				columns: this.columns,
				row: this.getRow(rowId),
				isView: this.isView,
				element: this.richObject,
			}, async () => {
				await this.dataStore.loadRowsFromBE(this.elementIdPayload())
			})
		},
		copyRow(rowId) {
			this.copyPrefillData = this.getRow(rowId)?.data
			this.showCopyRow = true
		},
		deleteRow(rowId) {
			this.rowToDelete = rowId
		},
		getRow(rowId) {
			return this.rows.find(row => row.id === rowId)
		},
		async loadRows() {
			if (!this.dataStore) return
			try {
				await this.dataStore.loadRowsFromBE(this.elementIdPayload())
			} catch (error) {
				console.error('Error loading rows:', error)
			}
		},
		async loadColumns() {
			if (!this.dataStore) return
			try {
				if (this.isView) {
					// Best-effort: loadColumnsFromBE only needs view.id for the
					// fetch itself; it uses view.columnSettings (if present) purely
					// to order/filter columns for that view. richObject won't carry
					// columnSettings, so ordering here relies on the backend's own
					// findAllByView() order.
					await this.dataStore.loadColumnsFromBE({ view: this.richObject })
				} else {
					await this.dataStore.loadColumnsFromBE({ tableId: this.richObject.id })
				}
			} catch (error) {
				console.error('Error loading columns:', error)
			}
		},
		async reload() {
			this.loaded = false
			await Promise.all([this.loadRows(), this.loadColumns()])
			this.loaded = true
		},
	},
}
</script>
<style lang="scss" scoped>

	.tables-content-widget {
		min-height: max(50vh, 200px);
		height: 50vh;
		overflow: scroll;

		& .header {
			position: sticky;
			top: 0;
			inset-inline-start: 0;
			z-index: 1;

			:where(.options) {
				position: sticky;
				top: 57px;
				z-index: 1;
				padding-bottom: 10px;
				background-color: var(--color-main-background);
			}

			h2 {
				position: sticky;
				top: 0;
				min-width: var(--widget-content-width);
				z-index: 1;
				background-color: var(--color-main-background);
				margin: 0 !important;
				padding: calc(var(--default-grid-baseline) * 4);

				& .loading-icon {
					display: inline-block;
					vertical-align: middle;
				}
			}
		}

		.nc-table {
			min-width: var(--widget-content-width);

			:where(.options.row) {
				display: none;
			}

			:where(thead) {
				position: sticky;
				top: 117px;

				:where(.cell-wrapper) {
					min-width: 150px;
					max-width: 200px;
				}

				:where(.sticky) {
					background: transparent !important;
				}
			}
		}

		& :deep(.options.row) {
			width: calc(var(--widget-content-width, 100%) - 12px);
		}

		& :deep(td) {
			vertical-align: middle !important;
		}
	}

</style>
