<!--
 - @copyright 2022 Carl Schwan <carl@carlschwan.eu>
 - @license AGPL-3.0-or-later
 -
 - This program is free software: you can redistribute it and/or modify
 - it under the terms of the GNU Affero General Public License as
 - published by the Free Software Foundation, either version 3 of the
 - License, or (at your option) any later version.
 -
 - This program is distributed in the hope that it will be useful,
 - but WITHOUT ANY WARRANTY; without even the implied warranty of
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -->
<template>
	<NcListItem class="version"
		:name="versionLabel"
		:force-display-actions="true"
		data-files-versions-version
		@click="click">
		<template #icon>
			<div v-if="!(loadPreview || previewLoaded)" class="version__image" />
			<img v-else-if="(isCurrent || version.hasPreview) && !previewErrored"
				:src="version.previewUrl"
				alt=""
				decoding="async"
				fetchpriority="low"
				loading="lazy"
				class="version__image"
				@load="previewLoaded = true"
				@error="previewErrored = true">
			<div v-else
				class="version__image">
				<ImageOffOutline :size="20" />
			</div>
		</template>
		<template #subname>
			<div class="version__info">
				<span :title="formattedDate">{{ version.mtime | humanDateFromNow }}</span>
				<!-- Separate dot to improve alignement -->
				<span class="version__info__size">•</span>
				<span class="version__info__size">{{ version.size | humanReadableSize }}</span>
			</div>
		</template>
		<template #actions>
			<NcActionButton v-if="enableLabeling"
				:close-after-click="true"
				@click="labelUpdate">
				<template #icon>
					<Pencil :size="22" />
				</template>
				{{ version.label === '' ? t('files_versions', 'Name this version') : t('files_versions', 'Edit version name') }}
			</NcActionButton>
			<NcActionButton v-if="!isCurrent && canView && canCompare"
				:close-after-click="true"
				@click="compareVersion">
				<template #icon>
					<FileCompare :size="22" />
				</template>
				{{ t('files_versions', 'Compare to current version') }}
			</NcActionButton>
			<NcActionButton v-if="!isCurrent"
				:close-after-click="true"
				@click="restoreVersion">
				<template #icon>
					<BackupRestore :size="22" />
				</template>
				{{ t('files_versions', 'Restore version') }}
			</NcActionButton>
			<NcActionLink :href="downloadURL"
				:close-after-click="true"
				:download="downloadURL">
				<template #icon>
					<Download :size="22" />
				</template>
				{{ t('files_versions', 'Download version') }}
			</NcActionLink>
			<NcActionButton v-if="!isCurrent && enableDeletion"
				:close-after-click="true"
				@click="deleteVersion">
				<template #icon>
					<Delete :size="22" />
				</template>
				{{ t('files_versions', 'Delete version') }}
			</NcActionButton>
		</template>
	</NcListItem>
</template>

<script>
import BackupRestore from 'vue-material-design-icons/BackupRestore.vue'
import Download from 'vue-material-design-icons/Download.vue'
import FileCompare from 'vue-material-design-icons/FileCompare.vue'
import Pencil from 'vue-material-design-icons/Pencil.vue'
import Delete from 'vue-material-design-icons/Delete.vue'
import ImageOffOutline from 'vue-material-design-icons/ImageOffOutline.vue'
import NcActionButton from '@nextcloud/vue/dist/Components/NcActionButton.js'
import NcActionLink from '@nextcloud/vue/dist/Components/NcActionLink.js'
import NcListItem from '@nextcloud/vue/dist/Components/NcListItem.js'
import Tooltip from '@nextcloud/vue/dist/Directives/Tooltip.js'
import moment from '@nextcloud/moment'
import { translate as t } from '@nextcloud/l10n'
import { joinPaths } from '@nextcloud/paths'
import { getRootUrl } from '@nextcloud/router'
import { loadState } from '@nextcloud/initial-state'

export default {
	name: 'Version',
	components: {
		NcActionLink,
		NcActionButton,
		NcListItem,
		BackupRestore,
		Download,
		FileCompare,
		Pencil,
		Delete,
		ImageOffOutline,
	},
	directives: {
		tooltip: Tooltip,
	},
	filters: {
		/**
		 * @param {number} bytes
		 * @return {string}
		 */
		humanReadableSize(bytes) {
			return OC.Util.humanFileSize(bytes)
		},
		/**
		 * @param {number} timestamp
		 * @return {string}
		 */
		humanDateFromNow(timestamp) {
			return moment(timestamp).fromNow()
		},
	},
	props: {
		/** @type {Vue.PropOptions<import('../utils/versions.ts').Version>} */
		version: {
			type: Object,
			required: true,
		},
		fileInfo: {
			type: Object,
			required: true,
		},
		isCurrent: {
			type: Boolean,
			default: false,
		},
		isFirstVersion: {
			type: Boolean,
			default: false,
		},
		loadPreview: {
			type: Boolean,
			default: false,
		},
		canView: {
			type: Boolean,
			default: false,
		},
		canCompare: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		return {
			previewLoaded: false,
			previewErrored: false,
			capabilities: loadState('core', 'capabilities', { files: { version_labeling: false, version_deletion: false } }),
		}
	},
	computed: {
		/**
		 * @return {string}
		 */
		versionLabel() {
			const label = this.version.label ?? ''

			if (this.isCurrent) {
				if (label === '') {
					return t('files_versions', 'Current version')
				} else {
					return `${label} (${t('files_versions', 'Current version')})`
				}
			}

			if (this.isFirstVersion && label === '') {
				return t('files_versions', 'Initial version')
			}

			return label
		},

		/**
		 * @return {string}
		 */
		downloadURL() {
			if (this.isCurrent) {
				return getRootUrl() + joinPaths('/remote.php/webdav', this.fileInfo.path, this.fileInfo.name)
			} else {
				return getRootUrl() + this.version.url
			}
		},

		/** @return {string} */
		formattedDate() {
			return moment(this.version.mtime).format('LLL')
		},

		/** @return {boolean} */
		enableLabeling() {
			return this.capabilities.files.version_labeling === true
		},

		/** @return {boolean} */
		enableDeletion() {
			return this.capabilities.files.version_deletion === true
		},
	},
	methods: {
		labelUpdate() {
			this.$emit('label-update-request')
		},

		restoreVersion() {
			this.$emit('restore', this.version)
		},

		deleteVersion() {
			this.$emit('delete', this.version)
		},

		click() {
			if (!this.canView) {
				window.location = this.downloadURL
				return
			}
			this.$emit('click', { version: this.version })
		},

		compareVersion() {
			if (!this.canView) {
				throw new Error('Cannot compare version of this file')
			}
			this.$emit('compare', { version: this.version })
		},

		t,
	},
}
</script>

<style scoped lang="scss">
.version {
	display: flex;
	flex-direction: row;

	&__info {
		display: flex;
		flex-direction: row;
		align-items: center;
		gap: 0.5rem;

		&__size {
			color: var(--color-text-lighter);
		}
	}

	&__image {
		width: 3rem;
		height: 3rem;
		border: 1px solid var(--color-border);
		border-radius: var(--border-radius-large);

		// Useful to display no preview icon.
		display: flex;
		justify-content: center;
		color: var(--color-text-light);
	}
}
</style>
