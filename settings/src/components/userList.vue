<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div id="app-content" class="user-list-grid" v-on:scroll.passive="onScroll">
		<div class="row" id="grid-header" :class="{'sticky': scrolled && !showConfig.showNewUserForm}">
			<div id="headerAvatar" class="avatar"></div>
			<div id="headerName" class="name">{{ t('settings', 'Username') }}</div>
			<div id="headerDisplayName" class="displayName">{{ t('settings',  'Display name') }}</div>
			<div id="headerPassword" class="password">{{ t('settings',  'Password') }}</div>
			<div id="headerAddress" class="mailAddress">{{ t('settings',  'Email') }}</div>
			<div id="headerGroups" class="groups">{{ t('settings',  'Groups') }}</div>
			<div id="headerSubAdmins" class="subadmins"
				 v-if="subAdminsGroups.length>0 && settings.isAdmin">{{ t('settings', 'Group admin for') }}</div>
			<div id="headerQuota" class="quota">{{ t('settings', 'Quota') }}</div>
			<div id="headerLanguages" class="languages"
				 v-if="showConfig.showLanguages">{{ t('settings', 'Language') }}</div>
			<div class="headerStorageLocation storageLocation"
				 v-if="showConfig.showStoragePath">{{ t('settings', 'Storage location') }}</div>
			<div class="headerUserBackend userBackend"
				 v-if="showConfig.showUserBackend">{{ t('settings', 'User backend') }}</div>
			<div class="headerLastLogin lastLogin"
				 v-if="showConfig.showLastLogin">{{ t('settings', 'Last login') }}</div>
			<div class="userActions"></div>
		</div>

		<form class="row" id="new-user" v-show="showConfig.showNewUserForm"
			  v-on:submit.prevent="createUser" :disabled="loading.all"
			  :class="{'sticky': scrolled && showConfig.showNewUserForm}">
			<div :class="loading.all?'icon-loading-small':'icon-add'"></div>
			<div class="name">
				<input id="newusername" type="text" required v-model="newUser.id"
					   :placeholder="t('settings', 'Username')" name="username"
					   autocomplete="off" autocapitalize="none" autocorrect="off"
					   ref="newusername" pattern="[a-zA-Z0-9 _\.@\-']+">
			</div>
			<div class="displayName">
				<input id="newdisplayname" type="text" v-model="newUser.displayName"
					   :placeholder="t('settings', 'Display name')" name="displayname"
					   autocomplete="off" autocapitalize="none" autocorrect="off">
			</div>
			<div class="password">
				<input id="newuserpassword" type="password" v-model="newUser.password"
					   :required="newUser.mailAddress===''" ref="newuserpassword"
					   :placeholder="t('settings', 'Password')" name="password"
					   autocomplete="new-password" autocapitalize="none" autocorrect="off"
					   :minlength="minPasswordLength">
			</div>
			<div class="mailAddress">
				<input id="newemail" type="email" v-model="newUser.mailAddress"
					   :required="newUser.password===''"
					   :placeholder="t('settings', 'Email')" name="email"
					   autocomplete="off" autocapitalize="none" autocorrect="off">
			</div>
			<div class="groups">
				<!-- hidden input trick for vanilla html5 form validation -->
				<input type="text" :value="newUser.groups" v-if="!settings.isAdmin"
					   tabindex="-1" id="newgroups" :required="!settings.isAdmin"
					   :class="{'icon-loading-small': loading.groups}"/>
				<multiselect v-model="newUser.groups" :options="canAddGroups" :disabled="loading.groups||loading.all"
						 tag-placeholder="create" :placeholder="t('settings', 'Add user in group')"
						 label="name" track-by="id" class="multiselect-vue"
						 :multiple="true" :taggable="true" :close-on-select="false"
						 :tag-width="60" @tag="createGroup">
							 <!-- If user is not admin, he is a subadmin.
							 	  Subadmins can't create users outside their groups
								  Therefore, empty select is forbidden -->
					<span slot="noResult">{{t('settings', 'No results')}}</span>
				</multiselect>
			</div>
			<div class="subadmins" v-if="subAdminsGroups.length>0 && settings.isAdmin">
				<multiselect :options="subAdminsGroups" v-model="newUser.subAdminsGroups"
							 :placeholder="t('settings', 'Set user as admin for')"
							 label="name" track-by="id" class="multiselect-vue"
							 :multiple="true" :close-on-select="false" :tag-width="60">
					<span slot="noResult">{{t('settings', 'No results')}}</span>
			</multiselect>
			</div>
			<div class="quota">
				<multiselect :options="quotaOptions" v-model="newUser.quota"
							 :placeholder="t('settings', 'Select user quota')"
							 label="label" track-by="id" class="multiselect-vue"
							 :allowEmpty="false" :taggable="true"
						 	 @tag="validateQuota" >
				</multiselect>
			</div>
			<div class="languages" v-if="showConfig.showLanguages">
				<multiselect :options="languages" v-model="newUser.language"
							 :placeholder="t('settings', 'Default language')"
							 label="name" track-by="code" class="multiselect-vue"
							 :allowEmpty="false" group-values="languages" group-label="label">
				</multiselect>
			</div>
			<div class="storageLocation" v-if="showConfig.showStoragePath"></div>
			<div class="userBackend" v-if="showConfig.showUserBackend"></div>
			<div class="lastLogin" v-if="showConfig.showLastLogin"></div>
			<div class="userActions">
				<input type="submit" id="newsubmit" class="button primary icon-checkmark-white has-tooltip"
					   value="" :title="t('settings', 'Add a new user')">
			</div>
		</form>

		<user-row v-for="(user, key) in filteredUsers" :user="user" :key="key" :settings="settings" :showConfig="showConfig"
				  :groups="groups" :subAdminsGroups="subAdminsGroups" :quotaOptions="quotaOptions" :languages="languages"
				  :externalActions="externalActions" />
		<infinite-loading @infinite="infiniteHandler" ref="infiniteLoading">
			<div slot="spinner"><div class="users-icon-loading icon-loading"></div></div>
			<div slot="no-more"><div class="users-list-end"></div></div>
			<div slot="no-results">
				<div id="emptycontent">
					<div class="icon-contacts-dark"></div>
					<h2>{{t('settings', 'No users in here')}}</h2>
				</div>
			</div>
		</infinite-loading>
	</div>
</template>

<script>
import userRow from './userList/userRow';
import { Multiselect } from 'nextcloud-vue'
import InfiniteLoading from 'vue-infinite-loading';
import Vue from 'vue';

export default {
	name: 'userList',
	props: ['users', 'showConfig', 'selectedGroup', 'externalActions'],
	components: {
		userRow,
		Multiselect,
		InfiniteLoading
	},
	data() {
		let unlimitedQuota = {id:'none', label:t('settings', 'Unlimited')},
			defaultQuota = {id:'default', label:t('settings', 'Default quota')};
		return {
			unlimitedQuota: unlimitedQuota,
			defaultQuota: defaultQuota,
			loading: {
				all: false,
				groups: false
			},
			scrolled: false,
			searchQuery: '',
			newUser: {
				id:'',
				displayName:'',
				password:'',
				mailAddress:'',
				groups: [],
				subAdminsGroups: [],
				quota: defaultQuota,
				language: {code: 'en', name: t('settings', 'Default language')}
			}
		};
	},
	mounted() {
		if (!this.settings.canChangePassword) {
			OC.Notification.showTemporary(t('settings', 'Password change is disabled because the master key is disabled'));
		}

		/**
		 * Init default language from server data. The use of this.settings
		 * requires a computed variable, which break the v-model binding of the form,
		 * this is a much easier solution than getter and setter on a computed var
		 */
		Vue.set(this.newUser.language, 'code', this.settings.defaultLanguage);

		/**
		 * In case the user directly loaded the user list within a group
		 * the watch won't be triggered. We need to initialize it.
		 */
		this.setNewUserDefaultGroup(this.selectedGroup);

		/**
		 * Register search
		 */
		this.userSearch = new OCA.Search(this.search, this.resetSearch);
	},
	computed: {
		settings() {
			return this.$store.getters.getServerData;
		},
		filteredUsers() {
			if (this.selectedGroup === 'disabled') {
				let disabledUsers = this.users.filter(user => user.enabled === false);
				if (disabledUsers.length===0 && this.$refs.infiniteLoading && this.$refs.infiniteLoading.isComplete) {
					// disabled group is empty, redirection to all users
					this.$router.push({name: 'users'});
					this.$refs.infiniteLoading.stateChanger.reset()
				}
				return disabledUsers;
			}
			if (!this.settings.isAdmin) {
				// we don't want subadmins to edit themselves
				return this.users.filter(user => user.enabled !== false && user.id !== oc_current_user);
			}
			return this.users.filter(user => user.enabled !== false);
		},
		groups() {
			// data provided php side + remove the disabled group
			return this.$store.getters.getGroups
				.filter(group => group.id !== 'disabled')
				.sort((a, b) => a.name.localeCompare(b.name));
		},
		canAddGroups() {
			// disabled if no permission to add new users to group
			return this.groups.map(group => {
				// clone object because we don't want
				// to edit the original groups
				group = Object.assign({}, group);
				group.$isDisabled = group.canAdd === false;
				return group;
			});
		},
		subAdminsGroups() {
			// data provided php side
			return this.$store.getters.getSubadminGroups;
		},
		quotaOptions() {
			// convert the preset array into objects
			let quotaPreset = this.settings.quotaPreset.reduce((acc, cur) => acc.concat({id: cur, label: cur}), []);
			// add default presets
			//quotaPreset.unshift(this.unlimitedQuota);
			//quotaPreset.unshift(this.defaultQuota);
			return quotaPreset;
		},
		minPasswordLength() {
			return this.$store.getters.getPasswordPolicyMinLength;
		},
		usersOffset() {
			return this.$store.getters.getUsersOffset;
		},
		usersLimit() {
			return this.$store.getters.getUsersLimit;
		},
		usersCount() {
			return this.users.length
		},

		/* LANGUAGES */
		languages() {
			return Array(
				{
					label: t('settings', 'Common languages'),
					languages: this.settings.languages.commonlanguages
				},
				{
					label: t('settings', 'All languages'),
					languages: this.settings.languages.languages
				}
			);
		}
	},
	watch: {
		// watch url change and group select
		selectedGroup: function (val, old) {
			this.$store.commit('resetUsers');
			this.$refs.infiniteLoading.stateChanger.reset()
			this.setNewUserDefaultGroup(val);
		},

		// make sure the infiniteLoading state is changed if we manually
		// add/remove data from the store
		usersCount: function(val, old) {
			// deleting the last user, reset the list
			if (val === 0 && old === 1) {
				this.$refs.infiniteLoading.stateChanger.reset()
			// adding the first user, warn the infiniteLoader that
			// the list is not empty anymore (we don't fetch the newly
			// added user as we already have all the info we need)
			} else if (val === 1 && old === 0) {
				this.$refs.infiniteLoading.stateChanger.loaded()
			}
		}
	},
	methods: {
		onScroll(event) {
			this.scrolled = event.target.scrollTo > 0;
		},

		/**
		 * Validate quota string to make sure it's a valid human file size
		 *
		 * @param {string} quota Quota in readable format '5 GB'
		 * @returns {Object}
		 */
		validateQuota(quota) {
			// only used for new presets sent through @Tag
			let validQuota = OC.Util.computerFileSize(quota);
			if (validQuota !== null && validQuota >= 0) {
				// unify format output
				quota = OC.Util.humanFileSize(OC.Util.computerFileSize(quota));
				return this.newUser.quota = {id: quota, label: quota};
			}
			// Default is unlimited
			return this.newUser.quota = this.quotaOptions[0];
		},

		infiniteHandler($state) {
			this.$store.dispatch('getUsers', {
				offset: this.usersOffset,
				limit: this.usersLimit,
				group: this.selectedGroup !== 'disabled' ? this.selectedGroup : '',
				search: this.searchQuery
			})
			.then((response) => { response ? $state.loaded() : $state.complete() });
		},

		/* SEARCH */
		search(query) {
			this.searchQuery = query;
			this.$store.commit('resetUsers');
			this.$refs.infiniteLoading.stateChanger.reset()
		},
		resetSearch() {
			this.search('');
		},

		resetForm() {
			// revert form to original state
			Object.assign(this.newUser, this.$options.data.call(this).newUser);
			// reset group
			this.setNewUserDefaultGroup(this.selectedGroup);
			this.loading.all = false;
		},
		createUser() {
			this.loading.all = true;
			this.$store.dispatch('addUser', {
				userid: this.newUser.id,
				password: this.newUser.password,
				displayName: this.newUser.displayName,
				email: this.newUser.mailAddress,
				groups: this.newUser.groups.map(group => group.id),
				subadmin: this.newUser.subAdminsGroups.map(group => group.id),
				quota: this.newUser.quota.id,
				language: this.newUser.language.code,
			})
			.then(() => {
				this.resetForm()
				this.$refs.newusername.focus();
			})
			.catch((error) => {
				this.loading.all = false;
				if (error.response && error.response.data && error.response.data.ocs && error.response.data.ocs.meta) {
					const statuscode = error.response.data.ocs.meta.statuscode
					if (statuscode === 102) {
						// wrong username
						this.$refs.newusername.focus();
					} else if (statuscode === 107) {
						// wrong password
						this.$refs.newuserpassword.focus();
					}
				}
			});
		},
		setNewUserDefaultGroup(value) {
			if (value && value.length > 0) {
				// setting new user default group to the current selected one
				let currentGroup = this.groups.find(group => group.id === value);
				if (currentGroup) {
					this.newUser.groups = [currentGroup];
					return;
				}
			}
			// fallback, empty selected group
			this.newUser.groups = [];
		},

		/**
		 * Create a new group
		 *
		 * @param {string} groups Group id
		 * @returns {Promise}
		 */
		createGroup(gid) {
			this.loading.groups = true;
			this.$store.dispatch('addGroup', gid)
				.then((group) => {
					this.newUser.groups.push(this.groups.find(group => group.id === gid))
					this.loading.groups = false;
				})
				.catch(() => {
					this.loading.groups = false;
				});
			return this.$store.getters.getGroups[this.groups.length];
		}
	}
}
</script>
