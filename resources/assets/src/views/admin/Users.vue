<template>
  <div class="container-fluid">
    <vue-good-table
      mode="remote"
      :rows="users"
      :total-rows="totalRecords || users.length"
      :columns="columns"
      :search-options="tableOptions.search"
      :pagination-options="tableOptions.pagination"
      style-class="vgt-table striped"
      @on-page-change="onPageChange"
      @on-sort-change="onSortChange"
      @on-search="onSearch"
      @on-per-page-change="onPerPageChange"
    >
      <template #table-row="props">
        <span v-if="props.column.field === 'email'">
          {{ props.formattedRow[props.column.field] }}
          <a :title="$t('admin.changeEmail')" data-test="email" @click="changeEmail(props.row)">
            <i class="fas fa-edit btn-edit" />
          </a>
        </span>
        <span v-else-if="props.column.field === 'nickname'">
          {{ props.formattedRow[props.column.field] }}
          <a :title="$t('admin.changeNickName')" data-test="nickname" @click="changeNickName(props.row)">
            <i class="fas fa-edit btn-edit" />
          </a>
        </span>
        <span v-else-if="props.column.field === 'score'">
          {{ props.formattedRow[props.column.field] }}
          <a :title="$t('admin.changeScore')" data-test="score" @click="changeScore(props.row)">
            <i class="fas fa-edit btn-edit" />
          </a>
        </span>
        <span v-else-if="props.column.field === 'players_count'">
          <a
            :href="props.row | playersLink"
            :title="$t('admin.inspectHisPlayers')"
            data-toggle="tooltip"
            data-placement="right"
          >{{ props.formattedRow[props.column.field] }}</a>
        </span>
        <span v-else-if="props.column.field === 'permission'">
          <span>{{ props.row | humanizePermission }}</span>
          <a
            v-if="props.row.permission < 1 || (props.row.operations === 2 && props.row.permission < 2)"
            :title="$t('admin.changePermission')"
            data-toggle="modal"
            data-target="#modal-permission"
            data-test="permission"
            @click="editingUser = props.row, permission = props.row.permission"
          >
            <i class="fas fa-edit btn-edit" />
          </a>
        </span>
        <span v-else-if="props.column.field === 'verified'">
          <span v-if="props.row.verified" v-t="'admin.verified'" />
          <span v-else v-t="'admin.unverified'" />
          <a
            :title="$t('admin.toggleVerification')"
            data-test="verification"
            @click="toggleVerification(props.row)"
          >
            <i
              class="fas btn-edit"
              :class="{ 'fa-toggle-on': props.row.verified, 'fa-toggle-off': !props.row.verified }"
            />
          </a>
        </span>
        <div v-else-if="props.column.field === 'operations'">
          <button class="btn btn-default" @click="changePassword(props.row)">
            {{ $t('admin.changePassword') }}
          </button>
          <button
            :disabled="props.row.permission >= 2 || (props.row.operations === 1 && props.row.permission >= 1)"
            class="btn btn-danger"
            data-test="deleteUser"
            @click="deleteUser(props.row)"
          >
            {{ $t('admin.deleteUser') }}
          </button>
        </div>
        <span v-else v-text="props.formattedRow[props.column.field]" />
      </template>
    </vue-good-table>
    <modal
      id="modal-permission"
      :title="$t(this.$t('admin.newPermission'))"
      center
      @confirm="changePermission"
    >
      <label class="mr-3">
        <input
          v-model="permission"
          type="radio"
          name="permission"
          :value="-1"
        >
        {{ $t('admin.banned') }}
      </label>
      <label class="mr-3">
        <input
          v-model="permission"
          type="radio"
          name="permission"
          :value="0"
        >
        {{ $t('admin.normal') }}
      </label>
      <label v-if="editingUser.operations === 2">
        <input
          v-model="permission"
          type="radio"
          name="permission"
          :value="1"
        >
        {{ $t('admin.admin') }}
      </label>
    </modal>
  </div>
</template>

<script>
import { VueGoodTable } from 'vue-good-table'
import 'vue-good-table/dist/vue-good-table.min.css'
import { trans } from '../../scripts/i18n'
import Modal from '../../components/Modal.vue'
import tableOptions from '../../components/mixins/tableOptions'
import serverTable from '../../components/mixins/serverTable'
import emitMounted from '../../components/mixins/emitMounted'
import { showModal, toast } from '../../scripts/notify'
import { truthy } from '../../scripts/validators'

export default {
  name: 'UsersManagement',
  components: {
    Modal,
    VueGoodTable,
  },
  filters: {
    humanizePermission(user) {
      switch (user.permission) {
        case -1:
          return trans('admin.banned')
        case 0:
          return trans('admin.normal')
        case 1:
          return trans('admin.admin')
        case 2:
          return trans('admin.superAdmin')
        default:
          return ''
      }
    },
    playersLink(user) {
      return `${blessing.base_url}/admin/players?uid=${user.uid}`
    },
  },
  mixins: [
    emitMounted,
    tableOptions,
    serverTable,
  ],
  data() {
    return {
      users: [],
      columns: [
        {
          field: 'uid', label: 'UID', type: 'number',
        },
        { field: 'email', label: this.$t('general.user.email') },
        {
          field: 'nickname', label: this.$t('general.user.nickname'), width: '150px',
        },
        {
          field: 'score', label: this.$t('general.user.score'), type: 'number', width: '102px',
        },
        {
          field: 'players_count', label: this.$t('admin.playersCount'), type: 'number', sortable: false,
        },
        {
          field: 'permission', label: this.$t('admin.permission'), globalSearchDisabled: true,
        },
        {
          field: 'verified', label: this.$t('admin.verification'), type: 'boolean', globalSearchDisabled: true,
        },
        { field: 'register_at', label: this.$t('general.user.register-at') },
        {
          field: 'operations', label: this.$t('admin.operationsTitle'), sortable: false, globalSearchDisabled: true,
        },
      ],
      editingUser: {},
      permission: 0,
    }
  },
  beforeMount() {
    this.fetchData()
  },
  methods: {
    async fetchData() {
      const { data, totalRecords } = await this.$http.get(
        `/admin/users/list${location.search}`,
        !location.search && this.serverParams,
      )
      this.totalRecords = totalRecords
      this.users = data
    },
    async changeEmail(user) {
      let value
      try {
        ({ value } = await showModal({
          mode: 'prompt',
          text: this.$t('admin.newUserEmail'),
          input: user.email,
          validator: truthy(this.$t('auth.emptyEmail')),
        }))
      } catch {
        return
      }

      const { code, message } = await this.$http.post(
        '/admin/users?action=email',
        { uid: user.uid, email: value },
      )
      if (code === 0) {
        user.email = value
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
    async toggleVerification(user) {
      const { code, message } = await this.$http.post(
        '/admin/users?action=verification',
        { uid: user.uid },
      )
      if (code === 0) {
        user.verified = !user.verified
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
    async changeNickName(user) {
      let value
      try {
        ({ value } = await showModal({
          mode: 'prompt',
          text: this.$t('admin.newUserNickname'),
          input: user.nickname,
          validator: truthy(this.$t('auth.emptyNickname')),
        }))
      } catch {
        return
      }

      const { code, message } = await this.$http.post(
        '/admin/users?action=nickname',
        { uid: user.uid, nickname: value },
      )
      if (code === 0) {
        user.nickname = value
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
    async changePassword(user) {
      let value
      try {
        ({ value } = await showModal({
          mode: 'prompt',
          text: this.$t('admin.newUserPassword'),
          inputType: 'password',
        }))
      } catch {
        return
      }

      const { code, message } = await this.$http.post(
        '/admin/users?action=password',
        { uid: user.uid, password: value },
      )
      code === 0 ? toast.success(message) : toast.error(message)
    },
    async changeScore(user) {
      let value
      try {
        ({ value } = await showModal({
          mode: 'prompt',
          text: this.$t('admin.newScore'),
          input: user.score,
          inputType: 'number',
        }))
      } catch {
        return
      }
      const score = Number.parseInt(value)

      const { code, message } = await this.$http.post(
        '/admin/users?action=score',
        { uid: user.uid, score },
      )
      if (code === 0) {
        user.score = score
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
    async changePermission() {
      const permission = Number.parseInt(this.permission)
      const { code, message } = await this.$http.post('/admin/users?action=permission', {
        uid: this.editingUser.uid,
        permission,
      })
      if (code === 0) {
        this.editingUser.permission = permission
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
    async deleteUser({ uid, originalIndex }) {
      try {
        await showModal({
          text: this.$t('admin.deleteUserNotice'),
          okButtonType: 'danger',
        })
      } catch {
        return
      }

      const { code, message } = await this.$http.post(
        '/admin/users?action=delete',
        { uid },
      )
      if (code === 0) {
        this.$delete(this.users, originalIndex)
        toast.success(message)
      } else {
        toast.error(message)
      }
    },
  },
}
</script>

<style lang="stylus">
.operations-menu
  margin-left -35px

.fa-edit
  cursor pointer

.fa-toggle-on, .fa-toggle-off
  font-size 18px
  cursor pointer

.row-at-bottom
  margin-top -100px
</style>
