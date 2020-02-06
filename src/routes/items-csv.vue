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
				collection === 'directus_webhooks' ? `/${currentProjectKey}/settings/` : null
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
					:to="createLink"
					@click="saveCSVUpload"
				/>
			</template>
		</v-header>

		<div class="items-csv">
			<label for="csv-select">Select a csv:</label>
			<input type="file" id="csv-select" name="csv-select" accept=".csv" />
		</div>

		<!-- <v-info-sidebar v-if="preferences">
			<template slot="system">
				<div class="layout-picker">
					<select
						:value="viewType"
						@input="updatePreferences('view_type', $event.target.value)"
					>
						<option v-for="(name, val) in layoutNames" :key="val" :value="val">
							{{ name }}
						</option>
					</select>
					<div class="preview">
						<v-icon :name="layoutIcons[viewType]" color="--sidebar-text-color" />
						<span class="label">{{ layoutNames[viewType] }}</span>
						<v-icon name="expand_more" color="--sidebar-text-color" />
					</div>
				</div>
			</template>
			<v-ext-layout-options
				:key="`${collection}-${viewType}`"
				class="layout-options"
				:type="viewType"
				:collection="collection"
				:fields="keyBy(fields, 'field')"
				:view-options="viewOptions"
				:view-query="viewQuery"
				:selection="selection"
				:primary-key-field="primaryKeyField"
				link="__link__"
				@query="setViewQuery"
				@options="setViewOptions"
			/>
		</v-info-sidebar> -->
		<v-info-sidebar wide>
			<span class="type-note">No settings</span>
		</v-info-sidebar>

		<!-- <portal v-if="confirmRemove" to="modal">
			<v-confirm
				:message="
					$tc('batch_delete_confirm', selection.length, {
						count: selection.length
					})
				"
				color="danger"
				:confirm-text="$t('delete')"
				@cancel="confirmRemove = false"
				@confirm="remove"
			/>
		</portal> -->
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
			notFound: false
		};
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
		createLink() {
			if (this.collection === 'directus_webhooks') {
				return `/${this.currentProjectKey}/settings/webhooks/+`;
			}

			if (this.collection.startsWith('directus_')) {
				return `/${this.currentProjectKey}/${this.collection.substr(9)}/+`;
			}

			return `/${this.currentProjectKey}/collections/${this.collection}/+`;
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
						name: this.$helpers.formatTitle(this.collection.substr(9)),
						path: `/${this.currentProjectKey}/${this.collection.substring(9)}`
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
			const fields = this.$store.state.collections[this.collection].fields;
			const fieldsArray = Object.values(fields).map(field => ({
				...field,
				name: this.$helpers.formatField(field.field, field.collection)
			}));

			//Filter out hidden_browser items.
			let filteredFields = fieldsArray.filter(field => field.hidden_browse !== true);

			return filteredFields;
		},
		batchURL() {
			return `/${this.currentProjectKey}/collections/${this.collection}/${this.selection
				.map(item => item[this.primaryKeyField])
				.join(',')}`;
		},

		collection() {
			if (this.$route.path.endsWith('webhooks')) return 'directus_webhooks';
			return this.$route.params.collection;
		},
		collectionInfo() {
			return this.$store.state.collections[this.collection];
		},
		emptyCollection() {
			return (this.meta && this.meta.total_count === 0) || false;
		},
		// searchQuery() {
		// 	if (!this.preferences) return '';
		// 	return this.preferences.search_query || '';
		// },
		// viewType() {
		// 	if (!this.preferences) return 'tabular';
		// 	return this.preferences.view_type || 'tabular';
		// },
		// viewQuery() {
		// 	if (!this.preferences) return {};

		// 	// `Fields` computed property return the fields which need to displayed. Here we want all fields.
		// 	let fields = this.$store.state.collections[this.collection].fields;
		// 	fields = Object.values(fields).map(field => ({
		// 		...field,
		// 		name: this.$helpers.formatField(field.field, field.collection)
		// 	}));

		// 	const viewQuery =
		// 		(this.preferences.view_query && this.preferences.view_query[this.viewType]) || {};

		// 	// Filter out the fieldnames of fields that don't exist anymore
		// 	// Sorting / querying fields that don't exist anymore will return
		// 	// a 422 in the API and brick the app

		// 	const collectionFieldNames = fields.map(f => f.field);

		// 	if (viewQuery.fields) {
		// 		viewQuery.fields = viewQuery.fields
		// 			.split(',')
		// 			.filter(fieldName => collectionFieldNames.includes(fieldName))
		// 			.join(',');
		// 	}

		// 	if (viewQuery.sort) {
		// 		// If the sort is descending, the fieldname starts with -
		// 		// The fieldnames in the array of collection field names don't have this
		// 		// which is why we have to take it out.
		// 		const sortFieldName = viewQuery.sort.startsWith('-')
		// 			? viewQuery.sort.substring(1)
		// 			: viewQuery.sort;

		// 		if (collectionFieldNames.includes(sortFieldName) === false) {
		// 			viewQuery.sort = this.primaryKeyField;
		// 		}
		// 	}

		// 	return viewQuery;
		// },
		// viewOptions() {
		// 	if (!this.preferences) return {};
		// 	return (
		// 		(this.preferences.view_options && this.preferences.view_options[this.viewType]) ||
		// 		{}
		// 	);
		// },

		filterableFieldNames() {
			return this.fields.filter(field => field.datatype).map(field => field.field);
		},
		layoutNames() {
			if (!this.$store.state.extensions.layouts) return {};
			const translatedNames = {};
			Object.keys(this.$store.state.extensions.layouts).forEach(id => {
				translatedNames[id] = this.$store.state.extensions.layouts[id].name;
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
			const fields = this.$store.state.collections[this.collection].fields;
			if (!fields) return null;
			let fieldsObj = find(fields, { type: 'status' });
			return fieldsObj && fieldsObj.field ? fieldsObj.field : null;
		},

		// Get the status name of the value that's marked as soft delete
		// This will make the delete button update the item to the hidden status
		// instead of deleting it completely from the database
		softDeleteStatus() {
			if (!this.collectionInfo.status_mapping || !this.statusField) return null;

			const statusKeys = Object.keys(this.collectionInfo.status_mapping);
			const index = findIndex(Object.values(this.collectionInfo.status_mapping), {
				soft_delete: true
			});
			return statusKeys[index];
		},

		userCreatedField() {
			if (!this.fields) return null;

			return (
				find(Object.values(this.fields), field => field.type.toLowerCase() === 'owner') ||
				{}
			).field;
		},
		primaryKeyField() {
			const fields = this.$store.state.collections[this.collection].fields;
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
			console.log('save CSV data to collection');
		},
		keyBy: keyBy,
		setMeta(meta) {
			this.meta = meta;
		},
		editCollection() {
			if (!this.$store.state.currentUser.admin) return;
			this.$router.push(`/${this.currentProjectKey}/settings/collections/${this.collection}`);
		},
		// setViewQuery(query) {
		// 	const newViewQuery = {
		// 		...this.preferences.view_query,
		// 		[this.viewType]: {
		// 			...this.viewQuery,
		// 			...query
		// 		}
		// 	};
		// 	this.updatePreferences('view_query', newViewQuery);
		// },
		// setViewOptions(options) {
		// 	const newViewOptions = {
		// 		...this.preferences.view_options,
		// 		[this.viewType]: {
		// 			...this.viewOptions,
		// 			...options
		// 		}
		// 	};
		// 	this.updatePreferences('view_options', newViewOptions);
		// },
		// updatePreferences(key, value, combine = false) {
		// 	if (combine) {
		// 		value = {
		// 			...this.preferences[key],
		// 			...value
		// 		};
		// 	}
		// 	this.$set(this.preferences, key, value);

		// 	// user vs role vs collection level preferences, == checks both null and undefined
		// 	const isPreferenceFallback = this.preferences.user == null;
		// 	if (isPreferenceFallback) {
		// 		return this.createCollectionPreset();
		// 	}

		// 	const id = this.$helpers.shortid.generate();
		// 	this.$store.dispatch('loadingStart', { id });

		// 	return this.$api
		// 		.updateCollectionPreset(this.preferences.id, {
		// 			[key]: value
		// 		})
		// 		.then(() => {
		// 			this.$store.dispatch('loadingFinished', id);
		// 		})
		// 		.catch(error => {
		// 			this.$store.dispatch('loadingFinished', id);
		// 			this.$events.emit('error', {
		// 				notify: this.$t('something_went_wrong_body'),
		// 				error
		// 			});
		// 		});
		// },
		// createCollectionPreset() {
		// 	const id = this.$helpers.shortid.generate();
		// 	this.$store.dispatch('loadingStart', { id });

		// 	const preferences = { ...this.preferences };
		// 	delete preferences.id;

		// 	return this.$api
		// 		.createCollectionPreset({
		// 			...preferences,
		// 			collection: this.collection,
		// 			user: this.$store.state.currentUser.id
		// 		})
		// 		.then(({ data }) => {
		// 			this.$store.dispatch('loadingFinished', id);
		// 			this.$set(this.preferences, 'id', data.id);
		// 			this.$set(this.preferences, 'user', data.user);
		// 		})
		// 		.catch(error => {
		// 			this.$store.dispatch('loadingFinished', id);
		// 			this.$events.emit('error', {
		// 				notify: this.$t('something_went_wrong_body'),
		// 				error
		// 			});
		// 		});
		// },
		remove() {
			const id = this.$helpers.shortid.generate();
			this.$store.dispatch('loadingStart', { id });

			let request;

			const itemKeys = this.selection.map(item => item[this.primaryKeyField]);

			if (this.softDeleteStatus) {
				request = this.$api.updateItem(this.collection, itemKeys.join(','), {
					[this.statusField]: this.softDeleteStatus
				});
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

		if (collection.startsWith('directus_') === false && collectionInfo === null) {
			return next(vm => (vm.notFound = true));
		}

		if (collectionInfo && collectionInfo.single) {
			return next(`/${store.state.currentProjectKey}/collections/${collection}/1`);
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

		const collectionInfo = this.$store.state.collections[collection] || null;

		if (collection.startsWith('directus_') === false && collectionInfo === null) {
			this.notFound = true;
			return next();
		}

		if (collectionInfo && collectionInfo.single) {
			return next(`/${this.$store.state.currentProjectKey}/collections/${collection}/1`);
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
.items-csv {
	padding: var(--page-padding-top-table) var(--page-padding) var(--page-padding-bottom);
}
</style>
