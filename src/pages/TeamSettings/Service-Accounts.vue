<script>
import { mapActions, mapGetters } from 'vuex'

import ConfirmDialog from '@/components/ConfirmDialog'
import ManagementLayout from '@/layouts/ManagementLayout'
import ServiceAccountsTable from '@/pages/TeamSettings/Service-Accounts-Table'
import { ROLE_MAP, ROLE_COLOR_MAP } from '@/utils/roles.js'

export default {
  components: {
    ConfirmDialog,
    ManagementLayout,
    ServiceAccountsTable
  },
  data() {
    return {
      // Dialogs
      dialogAddServiceAccount: false,

      // Inputs
      searchInput: '',
      serviceAccountNameInput: null,

      // Forms
      serviceAccountFormValid: true,

      accountCreationError: null,

      // Input rules
      rules: {
        required: value => !!value || 'This field is is required.'
      },

      // Loading states
      isCreatingServiceUser: false,
      isUpdatingServiceUser: false,

      // Signal passed as prop to ServiceAccountsTable
      // MembersTable watches this data attribute & refetches members every time this value changes.
      serviceAccountsSignal: 0,
      roleInput: null,
      updateAccountRole: false,
      serviceAccountID: null,
      // Role maps
      roleMap: ROLE_MAP,
      roleColorMap: ROLE_COLOR_MAP
    }
  },
  computed: {
    ...mapGetters('tenant', ['tenant']),
    ...mapGetters('user', ['user']),
    ...mapGetters('license', ['license', 'hasPermission']),
    confirmText() {
      return this.updateAccountRole ? 'Update' : 'Add'
    },
    titleText() {
      return this.updateAccountRole
        ? 'Update service account role'
        : 'Add a new service account'
    },
    filteredRoles() {
      if (!this.roles) return []
      let rolesToRM = ['RUNNER']

      if (!this.hasPermission('license', 'admin')) {
        rolesToRM.push('ENTERPRISE_LICENSE_ADMIN')
      }

      if (!this.hasPermission('feature', 'basic-rbac')) {
        rolesToRM.push('READ_ONLY_USER', 'USER')
      }
      return this.roles.filter(role => !rolesToRM.includes(role.name))
    }
  },
  watch: {
    tenant() {
      this.serviceAccountsSignal++
    }
  },
  methods: {
    ...mapActions('alert', ['setAlert']),
    handleAlert(type, message) {
      this.setAlert({
        alertShow: true,
        alertMessage: message,
        alertType: type
      })
    },
    async addServiceAccount() {
      this.isCreatingServiceUser = true
      this.accountCreationError = null

      try {
        const res = await this.$apollo.mutate({
          mutation: require('@/graphql/User/create-service-account.gql'),
          variables: {
            input: {
              tenant_id: this.tenant.id,
              name: this.serviceAccountNameInput,
              role_id: this.roleInput
            }
          },
          errorPolicy: 'all'
        })

        if (res?.data?.create_service_account?.id) {
          this.serviceAccountsSignal++
          this.dialogAddServiceAccount = false
          this.serviceAccountNameInput = null
        } else if (res?.errors) {
          this.setAlert({
            alertShow: true,
            alertMessage: res?.errors[0]?.message,
            alertType: 'error'
          })
        }
      } catch (e) {
        this.accountCreationError = `There was an error creating your service account:
            ${e}`
      } finally {
        this.isCreatingServiceUser = false
        this.serviceAccountID = null
        this.serviceAccountNameInput = null
        this.dialogAddServiceAccount = false
        this.roleInput = null
        this.updateAccountRole = false
      }
    },
    resetServiceAccountDialog() {
      this.serviceAccountNameInput = null
      this.roleInput = null
      this.updateAccountRole = false
      this.accountCreationError = null
      this.$refs['service-user-form'].reset()
    },
    handleAddOrUpdate() {
      if (this.updateAccountRole) this.updateServiceAccount()
      else this.addServiceAccount()
    },
    async updateServiceAccount() {
      this.isUpdatingServiceUser = true
      try {
        const res = await this.$apollo.mutate({
          mutation: require('@/graphql/Tenant/set-membership-role.gql'),
          variables: {
            input: {
              role_id: this.roleInput,
              membership_id: this.serviceAccountID
            }
          },
          errorPolicy: 'all'
        })
        if (res?.data?.set_membership_role) {
          this.serviceAccountsSignal++
          this.setAlert({
            alertShow: true,
            alertMessage: 'Role updated',
            alertType: 'Success'
          })
        } else if (res?.errors) {
          this.setAlert({
            alertShow: true,
            alertMessage: res?.errors[0]?.message,
            alertType: 'error'
          })
        }
      } catch (e) {
        this.setAlert({
          alertShow: true,
          alertMessage: `${e}`,
          alertType: 'error'
        })
      } finally {
        this.isUpdatingServiceUser = false
        this.serviceAccountID = null
        this.serviceAccountNameInput = null
        this.dialogAddServiceAccount = false
        this.roleInput = null
        this.updateAccountRole = false
      }
    },
    updateRole(event) {
      this.updateAccountRole = true
      this.serviceAccountNameInput = event.firstName
      this.serviceAccountID = event.membershipID
      this.dialogAddServiceAccount = true
    }
  },
  apollo: {
    roles: {
      query: require('@/graphql/TeamSettings/roles.gql'),
      loadingKey: 'loading',
      variables() {
        return {}
      },
      pollInterval: 10000,
      update: data => data.auth_role
    }
  }
}
</script>

<template>
  <ManagementLayout>
    <template #title>Service Accounts</template>

    <template #subtitle>
      <span v-if="hasPermission('create', 'service-account')">
        Manage service accounts and their API keys
      </span>
      <span v-else>
        You don't have permission to manage Service Accounts
      </span>
    </template>

    <template v-if="hasPermission('create', 'service-account')" #cta>
      <v-btn
        color="primary"
        class="white--text"
        large
        data-cy="invite-service-account"
        @click="dialogAddServiceAccount = true"
      >
        <v-icon left>
          person_add
        </v-icon>
        Add Service Account
      </v-btn>
    </template>

    <v-card tile>
      <v-card-text class="pa-0">
        <v-text-field
          v-if="!$vuetify.breakpoint.mdAndUp"
          v-model="searchInput"
          class="rounded-0 elevation-1 mb-1"
          solo
          dense
          hide-details
          single-line
          placeholder="Search by name"
          prepend-inner-icon="search"
          autocomplete="new-password"
        ></v-text-field>

        <ServiceAccountsTable
          :role-color-map="roleColorMap"
          :role-map="roleMap"
          :search="searchInput"
          :tenant="tenant"
          :refetch-signal="serviceAccountsSignal"
          @successful-action="handleAlert('success', $event)"
          @failed-action="handleAlert('error', $event)"
          @update="updateRole"
        ></ServiceAccountsTable>
      </v-card-text>
    </v-card>

    <!-- SERVICE ACCOUNT ADD DIALOG -->
    <ConfirmDialog
      v-model="dialogAddServiceAccount"
      :title="titleText"
      :confirm-text="confirmText"
      :error="accountCreationError"
      :loading="isCreatingServiceUser || isUpdatingServiceUser"
      :disabled="
        !serviceAccountFormValid ||
          isCreatingServiceUser ||
          isUpdatingServiceUser
      "
      :dialog-props="{
        'max-width': '600'
      }"
      @cancel="resetServiceAccountDialog"
      @confirm="handleAddOrUpdate"
    >
      <v-form
        ref="service-user-form"
        v-model="serviceAccountFormValid"
        @submit.prevent
      >
        <v-text-field
          v-model="serviceAccountNameInput"
          class="mb-3"
          autofocus
          :disabled="updateAccountRole"
          label="Account Name"
          data-cy="service-account"
          prepend-icon="engineering"
          outlined
          :rules="[rules.required]"
          @keydown.enter="addServiceAccount"
        />
        <v-select
          v-model="roleInput"
          outlined
          :menu-props="{ offsetY: true }"
          label="Role"
          data-cy="invite-role"
          prepend-icon="supervised_user_circle"
          :items="filteredRoles"
          :rules="[rules.required]"
          item-text="name"
          item-value="id"
          item-disabled="disabled"
        >
          <template #item="{item}">
            {{ roleMap[item.name] ? roleMap[item.name] : item.name }}
          </template>
          <template #selection="{item}">
            {{ roleMap[item.name] ? roleMap[item.name] : item.name }}
          </template>
        </v-select>
      </v-form>
    </ConfirmDialog>
  </ManagementLayout>
</template>
