<style scoped>
td {
	cursor: pointer;
}

th.checkbox {
	width: 1%;
}

.loading-cursor {
	cursor: wait;
}
.loading-cursor td {
	cursor: wait;
}

.img_gcode{
transform-origin: 0% 50%;
transition: transform .5s;
margin: 0 5px;
}

.img_gcode:hover{
	transform: scale(5);
}
</style>

<template>
	<div>
		<v-btn-toggle class="v-btn-toggle--only-child v-btn-toggle--selected">
			<v-btn id="v-btn--list" class="v-btn--large v-btn--depressed " v-on:click="for(dis in displayMode) displayMode[dis] = false; displayMode.asList=true; computeRowsCols();"><v-icon  class="mr-1"> view_list </v-icon></v-btn>
			<v-btn id="v-btn--mini" class="v-btn--large v-btn--depressed " v-on:click="for(dis in displayMode) displayMode[dis] = false; displayMode.asMini=true; computeRowsCols();"><v-icon  class="mr-1"> view_module </v-icon></v-btn>
		</v-btn-toggle>
		<v-data-table v-model="innerValue" v-bind="$props" :items="innerFilelist" :loading="loading || innerLoading" :custom-sort="sort" :pagination.sync="innerPagination" select-all hide-actions item-key="name" class="elevation-3" :class="{ 'empty-table-fix' : innerFilelist.length === 0, 'loading-cursor' : (loading || innerLoading || doingFileOperation || innerDoingFileOperation) }">
			<template slot="progress">
				<slot name="progress">
				<v-progress-linear indeterminate></v-progress-linear>
				</slot>
			</template>

			<template slot="no-data">
				<slot name="no-data">
					<v-alert :value="true" type="info" class="ma-0" @contextmenu.prevent="">No Files</v-alert>
				</slot>
			</template>

			<template slot="headers" slot-scope="props" v-if="displayMode.asList">
				<tr>
					<th class="checkbox pr-0">
						<v-checkbox :input-value="props.all" :indeterminate="props.indeterminate" primary hide-details @click="toggleAll"></v-checkbox>
					</th>
					<th v-for="header in props.headers" :key="header.text" :class="['text-xs-left column sortable', innerPagination.descending ? 'desc' : 'asc', header.value === innerPagination.sortBy ? 'active' : '']" @click="changeSort(header.value)" v-tab-control>
						{{ header.text }}
						<v-icon small>arrow_upward</v-icon>
					</th>
				</tr>
			</template>

			<template slot="items" slot-scope="props">
				<tr v-if="displayMode.asList" :active="props.selected" @click="props.selected = !props.selected" @dblclick="itemClicked(props.item)" @contextmenu.prevent="itemContextmenu(props, $event)" :data-filename="(props.item.isDirectory ? '*' : '') + props.item.name" draggable="true" @dragstart="dragStart(props.item, $event)" @dragover="dragOver(props.item, $event)" @drop.prevent="dragDrop(props.item, $event)" v-tab-control.contextmenu.dblclick @keydown.space="props.selected = !props.selected">
					<td class="pr-0">
						<v-checkbox :input-value="props.selected" primary hide-details></v-checkbox>
					</td>
					<template v-for="header in headers">
						<td v-if="header.value === 'name'" :key="header.value">
							<a href="#" @click.prevent.stop="itemClicked(props.item)" tabindex="-1">
								<v-layout row align-center>
									<img :src="props.item.ico" v-if="props.item.ico" class="img_gcode" width="30px"/>
									<v-icon class="mr-1" v-if="!props.item.ico"> {{ (props.item.isDirectory ? 'folder' : 'assignment') }} </v-icon>
									<span>{{ props.item.name }}</span>
								</v-layout>
							</a>
						</td>
						<td v-else-if="header.unit === 'bytes'" :key="header.value">
							{{ (props.item[header.value] !== null) ? $displaySize(props.item[header.value]) : '' }}
						</td>
						<td v-else-if="header.unit === 'date'" :key="header.value">
							{{ props.item.lastModified ? props.item.lastModified.toLocaleString() : $t('generic.novalue') }}
						</td>
						<td v-else-if="header.unit === 'filaments'" :key="header.value">
							<v-tooltip bottom :disabled="!props.item[header.value] || props.item[header.value].length <= 1">
								<span slot="activator">
									{{ displayLoadingValue(props.item, header.value, 1, 'mm') }}
								</span>
								<span>
									{{ $display(props.item[header.value], 1, 'mm') }}
								</span>
							</v-tooltip>
						</td>
						<td v-else-if="header.unit === 'time'" :key="header.value">
							{{ displayTimeValue(props.item, header.value) }}
						</td>
						<td v-else :key="header.value">
							{{ displayLoadingValue(props.item, header.value, header.precision, header.unit) }}
						</td>
					</template>
				</tr>
				<tr v-else-if="displayMode.asMini && props.item.name == filelist[0].name" v-for="row in rows">
					<td v-for="col in cols" :active="filelist[col+row*rows.length].selected" @click="filelist[col+row*rows.length].selected = !filelist[col+row*rows.length].selected" @dblclick="itemClicked(filelist[col+row*rows.length])" @contextmenu.prevent="itemContextmenu(filelist[col+row*rows.length], $event)" :data-filename="(filelist[col+row*rows.length].isDirectory ? '*' : '') + filelist[col+row*rows.length].name" draggable="true" @dragstart="dragStart(filelist[col+row*rows.length], $event)" @dragover="dragOver(filelist[col+row*rows.length], $event)" @drop.prevent="dragDrop(filelist[col+row*rows.length], $event)" v-tab-control.contextmenu.dblclick @keydown.space="filelist[col+row*rows.length].selected = !filelist[col+row*rows.length].selected">
						<v-tooltip bottom class="pr-0">
							<a slot="activator" href="#" @click.prevent.stop="itemClicked(filelist[col+row*rows.length])" tabindex="-1">
								<v-layout row align-center>
									<div style="height: 200px">
										<img :src="filelist[col+row*rows.length].ico" v-if="filelist[col+row*rows.length].ico" class="img_gcode_miniature" width="150px"/>
										<img src="http://192.168.1.40/img/folder.svg" v-if="filelist[col+row*rows.length].isDirectory" class="a-gcode-miniature" width="150px"/>
										<v-icon class="mr-1" v-if="!filelist[col+row*rows.length].ico && !filelist[col+row*rows.length].isDirectory"> {{ (filelist[col+row*rows.length].isDirectory ? 'folder' : 'assignment') }} </v-icon>
									</div>
									<div style="position: absolute;margin-top: 70px; margin-left: 0; width:150px;">
										{{ (8*row+col) + ': ' + filelist[col+row*rows.length].name.replace(/_/g,' ') }}
									</div>
								</v-layout>
							</a>
							<span v-if="!filelist[col+row*rows.length].isDirectory">
								<template v-for="header in headers">
									<p v-if="header.unit === 'bytes' && (filelist[col+row*rows.length][header.value] !== null)" :key="header.value" style="margin: 0px; padding: 0px">
										{{header.text}}: {{ (filelist[col+row*rows.length][header.value] !== null) ? $displaySize(filelist[col+row*rows.length][header.value]) : '' }}
									</p>
									<p v-else-if="header.unit === 'bytes' && filelist[col+row*rows.length].lastModified" :key="header.value" style="margin: 0px; padding: 0px">
										{{header.text}}: {{ filelist[col+row*rows.length].lastModified ? filelist[col+row*rows.length].lastModified.toLocaleString() : $t('generic.novalue') }}
									</p>
									<p v-else-if="header.unit === 'filaments'" :key="header.value" style="margin: 0px; padding: 0px">
										<span v-if="filelist[col+row*rows.length].filament || filelist[col+row*rows.length].filament.length > 1" :key="header.value" style="margin: 0px; padding: 0px">
												{{header.text}}: {{ $display(filelist[col+row*rows.length].filament, 1, 'mm') }}
										</span>
										<span v-else style="margin: 0px; padding: 0px">
												{{header.text}}: {{ displayLoadingValue(filelist[col+row*rows.length], 'filament', 1, 'mm') }}
										</span>
									</p>
									<p v-else-if="header.unit === 'time' && filelist[col+row*rows.length]" :key="header.value" style="margin: 0px; padding: 0px">
										{{header.text}}: {{ displayTimeValue(filelist[col+row*rows.length], header.value) }}
									</p>
									<p v-else-if="filelist[col+row*rows.length]" :key="header.value" style="margin: 0px; padding: 0px">
										{{header.text}}: {{ displayLoadingValue(filelist[col+row*rows.length], header.value, header.precision, header.unit) }}
									</p>
								</template>
							</span>
						</v-tooltip>
					</td>
				</tr>
			</template>
		</v-data-table>

		<v-menu v-model="contextMenu.shown" :position-x="contextMenu.x" :position-y="contextMenu.y" absolute offset-y v-tab-control.contextmenu>
			<v-list>
				<slot name="context-menu"></slot>

				<v-list-tile v-show="!noDownload && innerValue.length === 1 && filesSelected" @click="download">
					<v-icon class="mr-1">cloud_download</v-icon> Download File
				</v-list-tile>
				<v-list-tile v-show="!noEdit && innerValue.length === 1 && filesSelected" :disabled="!canEditFile" @click="edit">
					<v-icon class="mr-1">edit</v-icon> Edit File
				</v-list-tile>
				<v-list-tile v-show="!noRename && innerValue.length === 1" @click="rename">
					<v-icon class="mr-1">short_text</v-icon> Rename
				</v-list-tile>
				<v-list-tile v-show="!noDelete" @click="remove">
					<v-icon class="mr-1">delete</v-icon> Delete
				</v-list-tile>
				<v-list-tile v-show="!foldersSelected && innerValue.length > 1" @click="downloadZIP">
					<v-icon class="mr-1">archive</v-icon> Download as ZIP
				</v-list-tile>
			</v-list>
		</v-menu>

		<file-edit-dialog :shown.sync="editDialog.shown" :filename="editDialog.filename" v-model="editDialog.content" @editComplete="$emit('fileEdited', $event)"></file-edit-dialog>
		<input-dialog :shown.sync="renameDialog.shown" title="Rename File or Directory" prompt="Please enter a new name:" :preset="renameDialog.item && renameDialog.item.name" @confirmed="renameCallback"></input-dialog>
	</div>
</template>

<script>
'use strict'

import JSZip from 'jszip'
import saveAs from 'file-saver'
import VDataTable from 'vuetify/es5/components/VDataTable'

import { mapState, mapGetters, mapActions, mapMutations } from 'vuex'

import { getModifiedDirectory } from '../../store/machine'
import { DisconnectedError, OperationCancelledError } from '../../utils/errors.js'
import Path from '../../utils/path.js'

const bigFileThreshold = 1048576;		// 1 MiB
const maxEditFileSize = 15728640;		// 15 MiB

export default {
	props: {
		headers: {
			type: Array,
			default: () => [
				{
					text: 'Filename',
					value: 'name'
				},
				{
					text: 'Size',
					value: 'size',
					unit: 'bytes'
				},
				{
					text: 'Last Modified',
					value: 'lastModified',
					unit: 'date'
				}
			]
		},
		sortBy: {
			type: String,
			default: 'name'
		},
		descending: {
			type: Boolean,
			default: true
		},
		sortTable: String,
		directory: {
			type: String,
			required: true
		},
		filelist: Array,
		value: Array,
		loading: Boolean,
		doingFileOperation: Boolean,
		noDragDrop: Boolean,
		noDownload: Boolean,
		noEdit: Boolean,
		noRename: Boolean,
		noDelete: Boolean
	},
	computed: {
		...mapState(['selectedMachine']),
		...mapGetters(['isConnected']),
		...mapState('machine/cache', ['sorting']),
		...mapState('machine/model', ['storages']),
		storageIndex() {
			const matches = /^(\d+)/.exec(this.innerDirectory);
			if (matches) {
				return parseInt(matches[1]);
			}
			return 0;
		},
		foldersSelected() {
			return this.innerValue.some(item => item.isDirectory)
		},
		filesSelected() {
			return this.innerValue.some(item => !item.isDirectory)
		},
		canEditFile() {
			return (this.innerValue.length > 0) && (this.innerValue[0].size < maxEditFileSize);
		}
	},
	data() {
		return {
			unsubscribe: undefined,
			wasMounted: false,
			initialDirectory: this.directory,
			innerDirectory: this.directory,
			innerFilelist: [],
			innerLoading: false,
			innerDoingFileOperation: false,
			innerPagination: {
				sortBy: this.sortBy,
				descending: this.descending,
				rowsPerPage: -1
			},
			innerValue: [],
			togglingSelected: false,
			contextMenu: {
				shown: false,
				x: 0,
				y: 0
			},
			editDialog: {
				shown: false,
				filename: '',
				content: ''
			},
			renameDialog: {
				shown: false,
				directory: '',
				item: null
			},
			displayMode:{
				asList: true,
				asMini: false,
			},
			cols: [],
			rows:[],
		}
	},
	extends: VDataTable,
	methods: {
		...mapActions('machine', {
			machineDownload: 'download',
			machineMove: 'move',
			machineDelete: 'delete',
			getFileList: 'getFileList'
		}),
		...mapMutations('machine/cache', ['setSorting']),
		toggleAll() {
			// FIXME For some reason this is called twice when the checkbox is checked
			if (!this.togglingSelected) {
				this.togglingSelected = true;
				this.innerValue = this.innerValue.length ? [] : this.innerFilelist.slice();
			} else {
				this.togglingSelected = false;
			}
		},
		changeSort(column) {
			if (this.innerPagination.sortBy === column) {
				this.innerPagination.descending = !this.innerPagination.descending;
			} else {
				this.innerPagination.sortBy = column;
				this.innerPagination.descending = true;
			}

			this.$emit('update:sortBy', this.innerPagination.sortBy);
			this.$emit('update:descending', this.innerPagination.descending);
			this.$nextTick(function() {
				this.$emit('changedSort');
			});
		},
		sort(items, index, isDescending) {
			// Sort by index
			items.sort(function(a, b) {
				if (a[index] && a[index].constructor === String && b[index] && b[index].constructor === String) {
					return a[index].localeCompare(b[index], undefined, { sensivity: 'base' });
				}
				if (a[index] instanceof Array && b[index] instanceof Array) {
					const reducedA = a[index].length ? a.filament.reduce((a, b) => a + b) : 0;
					const reducedB = b[index].length ? b.filament.reduce((a, b) => a + b) : 0;
					return reducedA - reducedB;
				}
				return a[index] - b[index];
			});

			// Sort by null values
			items.sort((a, b) => (a[index] === b[index]) ? 0 : (a[index] === null ? -1 : 1));

			// Deal with descending order
			if (isDescending) {
				items.reverse();
			}

			// Then make sure directories come first
			items.sort((a, b) => (a.isDirectory === b.isDirectory) ? 0 : (a.isDirectory ? -1 : 1));
			return items;
		},
		async refresh() {
			await this.loadDirectory(this.innerDirectory);
			await this.computeRowsCols();
		},
		async loadDirectory(directory) {
			if (!this.isConnected || this.innerLoading) {
				return;
			}

			if (this.storageIndex >= this.storages.length || !this.storages[this.storageIndex].mounted) {
				this.innerDirectory = (this.storageIndex === 0) ? this.initialDirectory : `${this.storageIndex}:`;
				this.innerFilelist = [];
				return;
			}

			this.innerLoading = true;
			try {
				// Load file list and create missing props
				const files = await this.getFileList(directory);
				files.forEach(function(item) {
					this.headers.forEach(function(header) {
						if (!item.hasOwnProperty(header.value)) {
							item[header.value] = undefined;
						}
					});
				}, this);

				this.innerDirectory = directory;
				this.innerFilelist = files;
				this.innerValue = [];

				this.$nextTick(function() {
					this.$emit('directoryLoaded', directory);
				});
				await this.computeRowsCols();
			} catch (e) {
				if (!(e instanceof DisconnectedError)) {
					console.warn(e);
					this.$makeNotification('error', this.$t('error.filelistRequestFailed'), e.message);
				}
			}
			this.innerLoading = false;
		},
		displayLoadingValue(item, prop, precision, unit = '') {
			if (item.isDirectory) {
				return '';
			}
			if (!item[prop]) {
				return this.$t((item[prop] === undefined) ? 'generic.loading' : 'generic.novalue');
			}

			let displayValue;
			if (item[prop] instanceof Array) {
				if (!item[prop].length) {
					return this.$t('generic.novalue');
				}
				displayValue = item[prop].reduce((a, b) => a + b);
			} else {
				displayValue = item[prop];
			}

			if (precision !== undefined) {
				displayValue = displayValue.toFixed(precision);
			}
			return `${displayValue} ${unit}`;
		},
		displayTimeValue(item, prop) {
			if (item.isDirectory) {
				return '';
			}
			return (item[prop] !== null) ? this.$displayTime(item[prop]) : this.$t('generic.novalue');
		},
		async itemClicked(item) {
			if (item.isDirectory) {
				await this.loadDirectory(Path.combine(this.innerDirectory, item.name));
				await this.computeRowsCols();
			} else {
				this.$emit('fileClicked', item);
			}
		},
		itemContextmenu(props, e) {
			if (!props.selected) {
				props.selected = true;
			}
			e.preventDefault();

			this.contextMenu.shown = false;
			this.contextMenu.x = e.clientX;
			this.contextMenu.y = e.clientY;
			this.$nextTick(() => {
				this.contextMenu.shown = true;
			});
		},
		dragStart(item, e) {
			if (this.noDragDrop) {
				return;
			}

			const itemsToDrag = this.innerValue;
			if (itemsToDrag.indexOf(item) === -1) {
				itemsToDrag.push(item);
			}
			e.dataTransfer.setData('application/json', JSON.stringify({
				type: 'dwcFiles',
				directory: this.innerDirectory,
				items: itemsToDrag
			}));
			e.dataTransfer.effectAllowed = 'move';

			const table = this.$el.querySelector('table'), firstRow = table.tBodies[0].rows[0];
			const tableClone = table.cloneNode(true), itemFilename = (item.isDirectory ? '*' : '') + item.name;
			let offsetY = 0, countingOffset = true;

			tableClone.tHead.remove();
			Array.from(tableClone.tBodies[0].rows).forEach(function(row) {
				const filename = row.dataset.filename;
				if (itemsToDrag.some(item => (item.isDirectory ? '*' : '') + item.name === filename)) {
					Array.from(row.children).forEach((td, index) => td.style.width = `${firstRow.children[index].offsetWidth}px`);
					if (countingOffset) {
						if (filename === itemFilename) {
							countingOffset = false;
						} else {
							offsetY += firstRow.offsetHeight;
						}
					}
				} else {
					row.remove();
				}
			}, this);
			tableClone.style.opacity = 0.5;
			tableClone.style.position = 'absolute';
			tableClone.style.pointerEvents = 'none';
			table.parentNode.append(tableClone);

			const x = e.clientX - table.getClientRects()[0].left;
			const y = e.clientY - e.target.closest('tr').getClientRects()[0].top + offsetY;
			e.dataTransfer.setDragImage(tableClone, x, y);
			this.$nextTick(() => tableClone.remove());
		},
		dragOver(item, e) {
			if (!this.noDragDrop && item.isDirectory) {
				const jsonData = e.dataTransfer.getData('application/json');
				if (jsonData) {
					const data = JSON.parse(jsonData);
					if (data.type === 'dwcFiles' && !data.items.some(dataItem => dataItem.isDirectory && dataItem.name === item.name)) {
						e.preventDefault();
						e.stopPropagation();
					}
				}
			}
		},
		async dragDrop(item, e) {
			const data = JSON.parse(e.dataTransfer.getData('application/json'));
			const directory = this.innerDirectory;
			for (let i = 0; i < data.items.length; i++) {
				const from = Path.combine(data.directory, data.items[i].name);
				const to = Path.combine(directory, item.name, data.items[i].name);
				try {
					await this.machineMove({ from, to });
				} catch (e) {
					this.$makeNotification('error', `Failed to move ${data.items[i].name} to ${directory}`, e.message);
					break;
				}
			}
		},
		async download(item) {
			try {
				const filename = (item && item.name) ? item.name : this.innerValue[0].name;
				const blob = await this.machineDownload({ filename: Path.combine(this.innerDirectory, filename), type: 'blob' });
				saveAs(blob, filename);
			} catch (e) {
				if (!(e instanceof DisconnectedError) && !(e instanceof OperationCancelledError)) {
					// should be handled before we get here
					console.warn(e);
				}
			}
		},
		async edit(item) {
			try {
				const filename = Path.combine(this.innerDirectory, (item && item.name) ? item.name : this.innerValue[0].name);
				const response = await this.machineDownload({ filename, type: 'text', showSuccess: false });
				let notification, showDelay = 0;
				if (response.length > bigFileThreshold) {
					notification = this.$makeNotification('warning', 'Loading file', 'This file is relatively big so it may take a while before it is displayed.', false);
					showDelay = 1000;
				}

				const editDialog = this.editDialog;
				setTimeout(function() {
					editDialog.filename = filename;
					editDialog.content = response;
					editDialog.shown = true;

					if (notification) {
						setTimeout(notification.hide, 1000);
					}
				}, showDelay);
			} catch (e) {
				if (!(e instanceof DisconnectedError) && !(e instanceof OperationCancelledError)) {
					// should be handled before we get here
					console.warn(e);
				}
			}
		},
		async rename(item) {
			this.renameDialog.directory = this.innerDirectory;
			this.renameDialog.item = (item && item.name) ? item : this.innerValue[0];
			this.renameDialog.shown = true;
		},
		async renameCallback(newFilename) {
			const oldFilename = this.renameDialog.item.name;
			if (this.innerDoingFileOperation) {
				return;
			}

			this.innerDoingFileOperation = true;
			try {
				await this.machineMove({
					from: Path.combine(this.renameDialog.directory, oldFilename),
					to: Path.combine(this.renameDialog.directory, newFilename)
				});

				this.innerFilelist.some(function(file) {
					if (file.isDirectory === this.isDirectory && file.name === this.name) {
						file.name = newFilename;
						return true;
					}
					return false;
				}, this.renameDialog.item);

				this.$makeNotification('success', `Successfully renamed ${oldFilename} to ${newFilename}`);
			} catch (e) {
				console.warn(e);
				this.$log('error', `Failed to rename ${oldFilename} to ${newFilename}`, e.message);
			}
			this.innerDoingFileOperation = false;
		},
		async remove(items) {
			if (!items || !(items instanceof Array)) {
				items = this.innerValue.slice();
			}

			if (this.innerDoingFileOperation) {
				return;
			}

			this.innerDoingFileOperation = true;
			const deletedItems = [], directory = this.directory;
			for (let i = 0; i < items.length; i++) {
				try {
					const item = items[i];
					await this.machineDelete(Path.combine(directory, item.name));

					deletedItems.push(items[i]);
					this.innerFilelist = this.innerFilelist.filter(file => file.isDirectory !== item.isDirectory || file.name !== item.name);
					this.innerValue = this.innerValue.filter(file => file.isDirectory !== item.isDirectory || file.name !== item.name);
				} catch (e) {
					this.$makeNotification('error', `Failed to delete ${items[i].name}`, items[i].isDirectory ? 'Please make sure that this directory is empty' : e.message);
				}
			}

			if (deletedItems.length) {
				this.$log('success', (deletedItems.length > 1) ? `Successfully deleted ${deletedItems.length} items` : `Successfully deleted ${deletedItems[0].name}`);
			}
			this.innerDoingFileOperation = false;
		},
		async downloadZIP(items) {
			if (!items || !(items instanceof Array)) { items = this.innerValue.slice(); }
			const directory = this.directory;

			// Download files
			const zip = new JSZip();
			for (let i = 0; i < items.length; i++) {
				try {
					const blob = await this.machineDownload({ filename: Path.combine(directory, items[i].name), type: 'blob', num: i + 1, count: items.length });
					zip.file(items[i].name, blob);
				} catch (e) {
					if (!(e instanceof DisconnectedError) && !(e instanceof OperationCancelledError)) {
						// should be handled before we get here
						console.warn(e);
					}
					return;
				}
			}

			// Compress files and save the new archive
			const notification = this.$makeNotification('info', 'Compressing files...', 'Please stand by while your files are being compressed...');
			try {
				const zipBlob = await zip.generateAsync({ type: 'blob' });
				saveAs(zipBlob, 'download.zip');
			} catch (e) {
				console.warn(e);
				this.$makeNotification('error', 'Failed to compress files', e.message);
			}
			notification.hide();
		},
		async computeRowsCols(items){
			this.rows = [];
			this.cols = [];
			console.log(this.filelist);
			if (!items)
					items = this.filelist;
			if (items)
			{
				var i = 0;
				if(this.displayMode.asList){
				console.log('asList');
					for(i = 0; i < items.length; i++)
						this.rows.push(i);
						this.cols.push(0);
				} else if(this.displayMode.asMini){
				console.log('asMini');
					var rowLength = Math.ceil(items.length/8);
					var colLength = items.length/rowLength;
					for(i = 0; i < rowLength; i++)
					{
						this.rows.push(i);
					}
					for(i = 0; i < colLength; i++ )
					{
						this.cols.push(i);
					}
				}
			}
		}

	},
	mounted() {
		// Perform initial load
		if (this.isConnected) {
			this.wasMounted = (this.storages.length > this.storageIndex) && this.storages[this.storageIndex].mounted;
			this.loadDirectory(this.innerDirectory);
		}

		// Keep track of file changes
		const that = this;
		this.unsubscribe = this.$store.subscribeAction(async function(action, state) {
			if (!that.doingFileOperation && !that.innerDoingFileOperation) {
				const modifiedDirectory = getModifiedDirectory(action, state);
				if (Path.pathAffectsFilelist(modifiedDirectory, that.innerDirectory, that.innerFilelist)) {
					// Refresh when an external operation has caused a change
					await that.refresh();
				}
			}
		});

		// Get sorting from cache
		if (this.sortTable) {
			const column = this.sorting[this.sortTable].column;
			const descending = this.sorting[this.sortTable].descending;
			if (column !== this.innerPagination.sortBy || descending !== this.innerPagination.descending) {
				this.innerPagination.sortBy = column;
				this.innerPagination.descending = descending;
			}
		}
	},
	beforeDestroy() {
		this.unsubscribe();
	},
	watch: {
		selectedMachine() {
			this.innerFilelist = [];
			this.editDialog.shown = false;
			this.renameDialog.shown = false;
		},
		storages: {
			deep: true,
			handler() {
				// Refresh file list when the selected storage is mounted or unmounted
				if (this.storages.length <= this.storageIndex || this.wasMounted !== this.storages[this.storageIndex].mounted) {
					this.loadDirectory(this.wasMounted ? this.initialDirectory : this.innerDirectory);
					this.wasMounted = (this.storages.length > this.storageIndex) && this.storages[this.storageIndex].mounted;
				}
			}
		},
		directory(to) {
			if (to !== this.innerDirectory) {
				this.loadDirectory(to);
			}
		},
		innerDirectory(to) {
			if (this.directory !== to) {
				this.$emit('update:directory', to);
			}
		},
		innerFilelist(to) {
			if (this.filelist !== to) {
				this.$emit('update:filelist', to);
			}
		},
		innerLoading(to) {
			if (this.loading !== to) {
				this.$emit('update:loading', to);
			}
		},
		innerValue(to) {
			if (this.value !== to) {
				this.$emit('input', to);
			}
		},
		innerPagination: {
			deep: true,
			handler(to) {
				if (this.sortTable && (this.sorting[this.sortTable].column !== to.sortBy || this.sorting[this.sortTable].descending !== to.descending)) {
					this.setSorting({ table: this.sortTable, column: to.sortBy, descending: to.descending });
				}
			}
		},
		sorting: {
			deep: true,
			handler(to) {
				const column = to[this.sortTable].column;
				const descending = to[this.sortTable].descending;
				if (column !== this.innerPagination.sortBy || descending !== this.innerPagination.descending) {
					this.innerPagination.sortBy = column;
					this.innerPagination.descending = descending;
				}
			}
		}
	}
}
</script>
