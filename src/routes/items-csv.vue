<template>
	<v-not-found v-if="notFound" />
	<div v-else class="route-item-listing">
		<v-header
			info-toggle
			:item-detail="false"
			:breadcrumb="breadcrumb"
			:icon="breadcrumbIcon"
			:settings="collection === 'directus_webhooks'"
			:title="$helpers.formatTitle(collection)"
			:icon-link="
				collection === 'directus_webhooks'
					? `/${currentProjectKey}/settings/`
					: null
			"
		>
			<template slot="buttons">
				<v-header-button
					v-if="saveButton && !activity"
					key="save"
					icon="save"
					icon-color="button-primary-text-color"
					background-color="button-primary-background-color"
					:label="$t('new')"
					@click="saveCSVUpload"
				/>
			</template>
		</v-header>

		<div class="items-csv">
			<v-upload
				accept="text/csv"
				:multiple="false"
				@upload="CSVUploaded"
				:disabled="!!CSVData"
			/>
			<div v-if="!!CSVData">
				<div class="items-csv-merge-row">
					<v-checkbox
						class="items-csv-merge-checkbox"
						id="merge"
						:value="shouldMerge"
						:label="$t('Merge with existing items')"
						:inputValue="shouldMerge"
						v-model="shouldMerge"
					/>
				</div>

				<div class="items-csv-merge-row" v-if="shouldMerge">
					<p class="items-csv-merge-description">
						{{
							$t(
								'CSV rows will merge with existing items if they match the following field:'
							)
						}}
					</p>
					<v-simple-select
						class="items-csv-select"
						v-model="mergeField"
					>
						<option
							v-for="field in fields"
							:key="field.field"
							:value="field.field"
							:selected="mergeField === field.field"
						>
							{{ $helpers.formatTitle(field.name) }}
						</option>
					</v-simple-select>
				</div>

				<h2 class="type-heading-medium">
					Field Mappings
					<v-button
						v-if="checkboxes.length"
						@click="uncheckAllFields()"
					>
						{{ $t('Uncheck All') }}
					</v-button>
					<v-button
						v-if="!checkboxes.length"
						@click="checkAllFields()"
					>
						{{ $t('Check All') }}
					</v-button>
				</h2>

				<div
					v-for="field in fields"
					:key="field.field"
					class="items-csv-row"
				>
					<v-checkbox
						class="items-csv-checkbox"
						:id="field.field"
						:value="field.field"
						:label="field.name"
						:inputValue="checkboxes"
						v-model="checkboxes"
					/>

					<v-simple-select
						class="items-csv-select"
						v-model="CSVData.mappedTitles[field.name]"
					>
						<option
							v-for="title in CSVData.titles"
							:key="title"
							:value="title"
							:selected="
								CSVData.mappedTitles[field.name] === title
							"
						>
							{{ $helpers.formatTitle(title) }}
						</option>
					</v-simple-select>
				</div>
			</div>
		</div>

		<v-info-sidebar wide>
			<span class="type-note">No settings</span>
		</v-info-sidebar>
	</div>
</template>

<script>
import shortid from 'shortid';
import store from '../store/';

import VNotFound from './not-found.vue';
import { mapState } from 'vuex';
import { isEqual, isEmpty, isNil, find, findIndex, keyBy } from 'lodash';

import api from '../api';

export default {
	name: 'ItemsCSV',
	metaInfo() {
		return {
			title: this.$helpers.formatTitle('Upload CSV')
		};
	},
	components: {
		VNotFound
	},
	data() {
		return {
			selection: [],
			meta: null,
			// preferences: null,
			confirmRemove: false,
			notFound: false,
			checkboxes: [],
			CSVData: null,
			shouldMerge: false,
			mergeField: null
		};
	},
	created() {
		this.checkboxes = this.fields.map(({ field }) => field);
	},
	computed: {
		...mapState(['currentProjectKey']),
		activity() {
			return this.collection === 'directus_activity';
		},
		breadcrumbIcon() {
			if (this.collection === 'directus_webhooks') return 'arrow_back';
			return this.collectionInfo?.icon || 'box';
		},
		breadcrumb() {
			if (this.collection === 'directus_users') {
				return [
					{
						name: this.$t('user_directory'),
						path: `/${this.currentProjectKey}/users`
					}
				];
			}

			if (this.collection === 'directus_webhooks') {
				return [
					{
						name: this.$t('settings'),
						path: `/${this.currentProjectKey}/settings`
					},
					{
						name: this.$t('settings_webhooks'),
						path: `/${this.currentProjectKey}/settings/webhooks`
					}
				];
			}

			if (this.collection === 'directus_files') {
				return [
					{
						name: this.$t('file_library'),
						path: `/${this.currentProjectKey}/files`
					}
				];
			}

			if (this.collection.startsWith('directus_')) {
				return [
					{
						name: this.$helpers.formatTitle(
							this.collection.substr(9)
						),
						path: `/${
							this.currentProjectKey
						}/${this.collection.substring(9)}`
					}
				];
			} else {
				return [
					{
						name: this.$tc('collection', 2),
						path: `/${this.currentProjectKey}/collections`
					},
					{
						name: this.$helpers.formatCollection(this.collection),
						path: `/${this.currentProjectKey}/collections/${this.collection}`
					},
					{
						name: this.$t('Upload CSV'),
						path: `/${this.currentProjectKey}/collections/${this.collection}/csv`
					}
				];
			}
		},
		fields() {
			const fields = this.$store.state.collections[this.collection]
				.fields;
			const fieldsArray = Object.values(fields).map(field => ({
				...field,
				name: this.$helpers.formatField(field.field, field.collection)
			}));

			//Filter out hidden_browser items.
			let filteredFields = fieldsArray.filter(
				field => !field.hidden_browse && !field.readonly
			);
			return filteredFields;
		},
		batchURL() {
			return `/${this.currentProjectKey}/collections/${
				this.collection
			}/${this.selection
				.map(item => item[this.primaryKeyField])
				.join(',')}`;
		},

		collection() {
			if (this.$route.path.endsWith('webhooks'))
				return 'directus_webhooks';
			return this.$route.params.collection;
		},
		collectionInfo() {
			return this.$store.state.collections[this.collection];
		},
		emptyCollection() {
			return (this.meta && this.meta.total_count === 0) || false;
		},

		filterableFieldNames() {
			return this.fields
				.filter(field => field.datatype)
				.map(field => field.field);
		},
		layoutNames() {
			if (!this.$store.state.extensions.layouts) return {};
			const translatedNames = {};
			Object.keys(this.$store.state.extensions.layouts).forEach(id => {
				translatedNames[id] = this.$store.state.extensions.layouts[
					id
				].name;
			});
			return translatedNames;
		},
		layoutIcons() {
			if (!this.$store.state.extensions.layouts) return {};
			const icons = {};
			Object.keys(this.$store.state.extensions.layouts).forEach(id => {
				icons[id] = this.$store.state.extensions.layouts[id].icon;
			});
			return icons;
		},
		statusField() {
			const fields = this.$store.state.collections[this.collection]
				.fields;
			if (!fields) return null;
			let fieldsObj = find(fields, { type: 'status' });
			return fieldsObj && fieldsObj.field ? fieldsObj.field : null;
		},

		// Get the status name of the value that's marked as soft delete
		// This will make the delete button update the item to the hidden status
		// instead of deleting it completely from the database
		softDeleteStatus() {
			if (!this.collectionInfo.status_mapping || !this.statusField)
				return null;

			const statusKeys = Object.keys(this.collectionInfo.status_mapping);
			const index = findIndex(
				Object.values(this.collectionInfo.status_mapping),
				{
					soft_delete: true
				}
			);
			return statusKeys[index];
		},

		userCreatedField() {
			if (!this.fields) return null;
			const fields =
				find(
					Object.values(this.fields),
					field => field.type.toLowerCase() === 'owner'
				) || {};

			return fields.field;
		},
		primaryKeyField() {
			const fields = this.$store.state.collections[this.collection]
				.fields;
			if (!fields) return null;
			let fieldsObj = find(fields, { primary_key: true });
			return fieldsObj && fieldsObj.field ? fieldsObj.field : null;
		},
		permissions() {
			return this.$store.state.permissions;
		},
		permission() {
			return this.permissions[this.collection];
		},
		saveButton() {
			if (this.$store.state.currentUser.admin) return true;

			if (this.statusField) {
				return (
					Object.values(this.permission.statuses).some(
						permission => permission.create === 'full'
					) || this.permission.$create.create === 'full'
				);
			}

			return this.permission.create === 'full';
		}
	},
	watch: {
		$route() {
			if (this.$route.query.b) {
				this.$router.replace({
					path: this.$route.path
				});
			}
		}
	},
	methods: {
		saveCSVUpload() {
			//TODO: figure out how to save the CSV data to the collection

			const id = this.$helpers.shortid.generate();
			this.$store.dispatch('loadingStart', { id });

			return this.saveCSVData().then(savedValues => {
				this.$store.dispatch('loadingFinished', id);
				this.saving = false;
				this.$notify({
					title: this.$tc('item_saved', savedValues.length, {
						count: savedValues.length
					}),
					color: 'green',
					iconMain: 'check'
				});

				if (this.collection === 'directus_webhooks') {
					return this.$router.push(
						`/${this.currentProjectKey}/settings/webhooks`
					);
				}

				if (this.collection.startsWith('directus_')) {
					return this.$router.push(
						`/${this.currentProjectKey}/${this.collection.substring(
							9
						)}`
					);
				}

				return this.$router.push(
					`/${this.currentProjectKey}/collections/${this.collection}`
				);
			});
		},
		saveCSVData() {
			const mappedValues = this.fields.reduce((acc, field) => {
				if (
					this.checkboxes.includes(field.field) &&
					!!this.CSVData.mappedTitles[field.name]
				) {
					acc[field.name] = this.CSVData.dataMap[
						this.CSVData.mappedTitles[field.name]
					];
				}

				return acc;
			}, {});

			const create = new Array(this.CSVData.dataLength)
				.fill({})
				.map((_, index) => {
					return this.fields.reduce((acc, field) => {
						if (mappedValues[field.name])
							acc[field.field] = mappedValues[field.name][index];
						return acc;
					}, {});
				});
			return this.$api
				.createItems(this.collection, create)
				.then(res => res.data);
		},
		async CSVUploaded(fileData) {
			const {
				data: {
					data: {
						id: fileId,
						data: { full_url }
					}
				}
			} = fileData;
			const response = await fetch(full_url);
			const text = await response.text();
			const CSVData = this.$helpers.parseCSV(text);
			this.CSVData = this.formatCSVData(CSVData);
			const CSVTitles = CSVData[0];

			const id = this.$helpers.shortid.generate();
			this.$store.dispatch('loadingStart', { id });

			this.$api
				.deleteItem('directus_files', fileId)
				.then(() => {
					this.$store.dispatch('loadingFinished', id);
				})
				.catch(error => {
					this.$store.dispatch('loadingFinished', id);
					this.$events.emit('error', {
						notify: this.$t('something_went_wrong_body'),
						error
					});
				});
		},
		formatCSVData(data) {
			const titles = data[0];
			const dataLength = data.length - 1;
			const dataMap = data.reduce((acc, dataArray, index) => {
				if (!dataArray.length) return acc;
				if (index === 0) {
					dataArray.forEach(title => (acc[title] = []));
					return acc;
				}
				dataArray.forEach((datum, datumIndex) =>
					acc[titles[datumIndex]].push(datum)
				);
				return acc;
			}, {});
			const matchingCSVTitles = this.mapMatchingCSVTitles(titles);
			const mappedTitles = this.fields.reduce((acc, field) => {
				if (matchingCSVTitles.has(field.name))
					acc[field.name] = field.name;
				else acc[field.name] = null;
				return acc;
			}, {});

			return {
				titles,
				dataMap,
				mappedTitles,
				dataLength
			};
		},
		mapMatchingCSVTitles(titles) {
			const formTitleIndexes = this.fields.reduce((acc, field, index) => {
				acc.set(field.name, index);
				return acc;
			}, new Map());

			return titles.reduce((acc, title) => {
				const formTitleIndex = formTitleIndexes.get(title);
				if (Number.isInteger(formTitleIndex))
					acc.set(title, formTitleIndex);
				return acc;
			}, new Map());
		},
		checkAllFields() {
			this.checkboxes = this.fields.map(({ field }) => field);
		},
		uncheckAllFields() {
			this.checkboxes = [];
		},
		keyBy: keyBy,
		setMeta(meta) {
			this.meta = meta;
		},
		editCollection() {
			if (!this.$store.state.currentUser.admin) return;
			this.$router.push(
				`/${this.currentProjectKey}/settings/collections/${this.collection}`
			);
		},
		remove() {
			const id = this.$helpers.shortid.generate();
			this.$store.dispatch('loadingStart', { id });

			let request;

			const itemKeys = this.selection.map(
				item => item[this.primaryKeyField]
			);

			if (this.softDeleteStatus) {
				request = this.$api.updateItem(
					this.collection,
					itemKeys.join(','),
					{
						[this.statusField]: this.softDeleteStatus
					}
				);
			} else {
				request = this.$api.deleteItems(
					this.collection,
					this.selection.map(item => item[this.primaryKeyField])
				);
			}

			request
				.then(() => {
					this.$store.dispatch('loadingFinished', id);
					this.$refs.listing.getItems();
					this.selection = [];
					this.confirmRemove = false;
				})
				.catch(error => {
					this.$store.dispatch('loadingFinished', id);
					this.$events.emit('error', {
						notify: this.$t('something_went_wrong_body'),
						error
					});
				});
		}
	},
	beforeRouteEnter(to, from, next) {
		let { collection } = to.params;

		if (to.path.endsWith('webhooks')) collection = 'directus_webhooks';

		const collectionInfo = store.state.collections[collection] || null;

		if (
			collection.startsWith('directus_') === false &&
			collectionInfo === null
		) {
			return next(vm => (vm.notFound = true));
		}

		if (collectionInfo && collectionInfo.single) {
			return next(
				`/${store.state.currentProjectKey}/collections/${collection}/1`
			);
		}

		const id = shortid.generate();
		store.dispatch('loadingStart', { id });

		return api
			.getMyListingPreferences(collection)
			.then(preferences => {
				store.dispatch('loadingFinished', id);
				next(vm => {
					vm.$data.preferences = preferences;
				});
			})
			.catch(error => {
				store.dispatch('loadingFinished', id);
				this.$events.emit('error', {
					notify: this.$t('something_went_wrong_body'),
					error
				});
			});
	},
	beforeRouteUpdate(to, from, next) {
		const { collection } = to.params;

		this.preferences = null;
		this.selection = [];
		this.meta = {};
		this.notFound = false;

		const collectionInfo =
			this.$store.state.collections[collection] || null;

		if (
			collection.startsWith('directus_') === false &&
			collectionInfo === null
		) {
			this.notFound = true;
			return next();
		}

		if (collectionInfo && collectionInfo.single) {
			return next(
				`/${this.$store.state.currentProjectKey}/collections/${collection}/1`
			);
		}

		const id = this.$helpers.shortid.generate();
		this.$store.dispatch('loadingStart', { id });

		return api
			.getMyListingPreferences(collection)
			.then(preferences => {
				this.$store.dispatch('loadingFinished', id);
				this.preferences = preferences;
				next();
			})
			.catch(error => {
				this.$store.dispatch('loadingFinished', id);
				this.$events.emit('error', {
					notify: this.$t('something_went_wrong_body'),
					error
				});
			});
	}
};
</script>

<style lang="scss" scoped>
.v-upload.disabled {
	opacity: 0.3;
}

.items-csv {
	padding: var(--page-padding-top-table) var(--page-padding)
		var(--page-padding-bottom);

	h2 {
		margin-top: 32px;
		display: flex;
		justify-content: space-between;
	}

	&-row {
		width: 100%;
		margin-top: 16px;
		display: flex;
		align-items: center;
	}

	&-merge {
		&-checkbox {
			display: block;
			width: 100%;
		}
		&-row {
			width: 100%;
			margin-top: 8px;
		}
		&-description {
			display: block;
			margin-bottom: 8px;
		}
	}

	&-checkbox {
		width: 200px;
	}

	&-select {
		width: 200px;
	}
}
</style>
