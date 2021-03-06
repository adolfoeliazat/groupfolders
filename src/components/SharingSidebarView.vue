<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->
<template>
	<div>
		<div  class="groupfolder-entry">
			<div class="avatar icon-group-white"></div>
			<span class="username">{{ t('groupfolders', 'Groupfolder') }} {{ groupFolderId }} </span>
		</div>
		<table v-if="!loading">
			<thead>
				<tr>
					<th></th>
					<th></th>
					<th>{{ t('groupfolders', 'Read') }}</th>
					<th>{{ t('groupfolders', 'Write') }}</th>
					<th v-if="model.type === 'dir'">{{ t('groupfolders', 'Create') }}</th>
					<th>{{ t('groupfolders', 'Delete') }}</th>
					<th>{{ t('groupfolders', 'Share') }}</th>
					<th></th>
				</tr>
			</thead>
			<tbody>
				<tr v-if="!isAdmin">
					<td><avatar user="admin" :size="24"></avatar></td>
					<td class="username">{{ t('groupfolders', 'You') }}</td>
					<td><AclStateButton :state="getState(OC.PERMISSION_READ, model.permissions, 1)" :read-only="true" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_UPDATE, model.permissions, 1)" :read-only="true" /></td>
					<td v-if="model.type === 'dir'"><AclStateButton :state="getState(OC.PERMISSION_CREATE, model.permissions, 1)" :read-only="true" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_DELETE, model.permissions, 1)" :read-only="true" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_SHARE, model.permissions, 1)" :read-only="true" /></td>
				</tr>
				<tr v-if="isAdmin" v-for="item in list">
					<td>
						<avatar :user="item.mappingId" :size="24"></avatar>
					</td>
					<td class="username">
						{{ item.mappingId }}
						<span v-if="item.mappingType === 'group'"> {{ t('groupfolders', '(Group)') }}</span>
					</td>
					<td><AclStateButton :state="getState(OC.PERMISSION_READ, item.permissions, item.mask)" @update="changePermission(item, OC.PERMISSION_READ, $event)" :disabled="loading" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_UPDATE, item.permissions, item.mask)" @update="changePermission(item, OC.PERMISSION_UPDATE, $event)" :disabled="loading" /></td>
					<td v-if="model.type === 'dir'"><AclStateButton :state="getState(OC.PERMISSION_CREATE, item.permissions, item.mask)" @update="changePermission(item, OC.PERMISSION_CREATE, $event)" :disabled="loading" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_DELETE, item.permissions, item.mask)" @update="changePermission(item, OC.PERMISSION_DELETE, $event)" :disabled="loading" /></td>
					<td><AclStateButton :state="getState(OC.PERMISSION_SHARE, item.permissions, item.mask)" @update="changePermission(item, OC.PERMISSION_SHARE, $event)" :disabled="loading" /></td>
					<td><a class="icon-close" v-tooltip="t('groupfolders', 'Remove access rule')" @click="removeAcl(item)"></a></td>
				</tr>
			</tbody>
		</table>
		<button v-if="isAdmin && !loading && !showAclCreate" @click="toggleAclCreate"><span class="icon-add"></span> {{ t('groupfolders', 'Add advanced permission rule') }}</button>
		<multiselect v-if="isAdmin && !loading" v-show="showAclCreate" ref="select"
					 v-model="value" :options="options" @select="createAcl" :reset-after="true" @search-change="searchMappings"
					 :internal-search="false"
			track-by="unique">
			<template slot="singleLabel" slot-scope="props">
				<avatar :user="props.option.id" :isNoUser="props.option.type !== 'user'"/> {{ props.option.id }}
				<span v-if="props.option.type === 'group'">{{ t('groupfolders', '(Group)') }}</span>
			</template>
			<template slot="option" slot-scope="props">
				<avatar :user="props.option.id" :isNoUser="props.option.type !== 'user'"/> {{ props.option.id }}
				<span v-if="props.option.type === 'group'">{{ t('groupfolders', '(Group)') }}</span>
			</template>
		</multiselect>
	</div>
</template>

<script>
	import Vue from 'vue'
	import axios from 'nextcloud-axios';
	import { Avatar, Multiselect } from 'nextcloud-vue';
	import AclStateButton from './AclStateButton'
	import Rule from './../model/Rule'
	import BinaryTools from './../BinaryTools'
	import client from './../client'



	export default {
		name: 'SharingSidebarView',
		props: ['fileModel'],
		components: {
			Avatar, Multiselect, AclStateButton
		},
		beforeMount() {
			this.loading = true;
			this.model = JSON.parse(JSON.stringify(this.fileModel))
			client.propFind(this.model).then((data) => {
				if (data.acls) {
					this.list = data.acls;
				}
				this.groupFolderId = data.groupFolderId;
				this.loading = false;
				this.searchMappings('')
			})
		},
		data() {
			return {
				showAclCreate: false,
				groupFolderId: null,
				loading: false,
				options: [],
				value: null,
				model: null,
				list: [],
			}
		},
		computed: {
			isAdmin() {
				return OC.isUserAdmin()
			},
			isInherited() {
				return (permission, permissions, mask) => {
					return (permission & ~mask) === 0
				}
			},
			isAllowed() {
				return (permission, permissions) => {
					return (permission & permissions) > 0
				}
			},
			getState() {
				return (permission, permissions, mask) => {
					const inheritance = this.isInherited(permission, permissions, mask) << 1
					const permitted = this.isAllowed(permission, permissions)
					return inheritance | permitted;
				}
			}
		},
		methods: {
			searchMappings(query) {
				axios.get(OC.generateUrl(`apps/groupfolders/folders/${this.groupFolderId}/search`) + '?format=json&search=' + query).then((result) => {
					let groups = result.data.ocs.data.groups.map((group) => {
						return {
							unique: 'group:' + group.gid,
							type: 'group',
							id: group.gid,
							displayname: group.displayname
						}
					})
					let users = Object.values(result.data.ocs.data.users).map((user) => {
						return {
							unique: 'user:' + user.uid,
							type: 'user',
							id: user.uid,
							displayname: user.displayname
						}
					})
					this.options = [...groups, ...users].filter((entry) => {
						// filter out existing acl rules
						return !this.list.find((existingAcl) => entry.unique === existingAcl.getUniqueMappingIdentifier());
					});
				})
			},
			toggleAclCreate() {
				this.showAclCreate = !this.showAclCreate;
				Vue.nextTick(() => {
					this.$refs.select.$el.focus()
				});
			},
			createAcl(option) {
				let rule = new Rule();
				rule.fromValues(option.type, option.id, 0b00000, 0b11111);
				this.list.push(rule);
				client.propPatch(this.model, this.list).then(() => {
					this.showAclCreate = false;
				});
			},
			removeAcl(rule) {
				const index = this.list.indexOf(rule);
				let list = JSON.parse(JSON.stringify(this.list))
				if (index > -1) {
					list.splice(index, 1);
				}
				client.propPatch(this.model, this.list).then(() => {
					this.list.splice(index, 1);
				});

			},
			changePermission(item, permission, $event) {
				let index = this.list.indexOf(item);
				const inherit = ($event < 2);
				const allow = ($event & (0b01)) === 1;
				const bit = BinaryTools.firstHigh(permission);
				if (inherit) {
					item.mask = BinaryTools.clear(item.mask, bit)
					// TODO check if: we can ignore permissions, since they are inherited
				} else {
					item.mask = BinaryTools.set(item.mask, bit)
					if (allow) {
						item.permissions = BinaryTools.set(item.permissions, bit)
					} else {
						item.permissions = BinaryTools.clear(item.permissions, bit)
					}
				}
				Vue.set(this.list, index, item)
				client.propPatch(this.model, this.list).then(() => {
					// TODO block UI during save
				});
			}
		}
	}
</script>

<style scoped>
	.groupfolder-entry {
		height: 44px;
		white-space: normal;
		display: inline-flex;
		align-items: center;
		position: relative;
	}
	.avatar.icon-group-white {
		display: inline-block;
		background-color: var(--color-primary, #0082c9);
		padding: 16px;
	}
	.groupfolder-entry .username {
		padding: 0 8px;
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}
	table {
		width: 100%;
		margin-top: -44px;
		margin-bottom: 5px;
	}
	thead th {
		text-align: center;
		height: 44px;
	}
	tbody tr td:first-child {
		width: 24px;
		padding: 0;
		padding-left: 4px;
	}
	table .avatardiv {
		margin-top: 6px;
	}
	table thead th:nth-child(2),
	table .username {
		width: 50%;
	}
	table button {
		height: 26px;
		width: 24px !important;
		display: block;
		border-radius: 50%;
		margin: auto;
	}
	a.icon-close {
		display: inline-block;
		height: 24px;
		vertical-align: middle;
		background-size: 12px;
		opacity: .7;
		float: right;
	}
	a.icon-close:hover {
		opacity: 1;
	}

	.multiselect {
		margin-left: 44px;
		width: calc(100% - 44px);
	}
</style>
