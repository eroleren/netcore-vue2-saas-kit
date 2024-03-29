<template>
  <div>
    <div class="bg-white shadow-sm border-b border-gray-300 w-full py-2">
      <div
        class="mx-auto max-w-5xl xl:max-w-7xl flex items-center justify-between px-4 sm:px-6 lg:px-8 space-x-2"
      >
        <h1 class="flex-1 font-bold flex items-center truncate">
          {{ $t("models.user.plural") }}
          <span
            v-if="!loading"
            class="ml-2 inline-flex items-center px-3 py-0.5 rounded-full text-xs font-medium bg-gray-50 text-gray-800 border border-gray-300"
          >{{ filteredItems.length }}</span>
        </h1>
        <div class="flex items-center space-x-2 h-9">
          <ButtonSecondary @click="reload" type="button">{{ $t("shared.reload") }}</ButtonSecondary>
        </div>
      </div>
    </div>
    <div class="pt-2 space-y-2 mx-auto px-4 sm:px-6 lg:px-8 max-w-5xl xl:max-w-7xl">
      <Loading v-if="loading" />
      <div v-else class="space-y-2">
        <div class="flex justify-between">
          <div class="flex items-center justify-start w-full">
            <div class="relative flex items-center w-full">
              <div
                class="focus-within:z-10 absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  class="h-5 w-5"
                  viewBox="0 0 20 20"
                  fill="currentColor"
                >
                  <path
                    fill-rule="evenodd"
                    d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z"
                    clip-rule="evenodd"
                  />
                </svg>
              </div>
              <input
                ref="inputSearch"
                type="text"
                name="buscador"
                id="buscador"
                class="w-full focus:ring-theme-500 focus:border-theme-500 block rounded-md pl-10 sm:text-sm border-gray-300"
                :placeholder="$t('shared.searchDot')"
                v-model="searchInput"
              />
            </div>
          </div>
        </div>
        <div v-if="filteredItems.length === 0">
          <EmptyState
            class="bg-white"
            :captions="{
              thereAreNo: $t('app.workspaces.errors.noUsers'),
            }"
          />
        </div>
        <div v-else>
          <div>
            <div class="flex flex-col">
              <div class="overflow-x-auto">
                <div class="py-2 align-middle inline-block min-w-full">
                  <div class="shadow overflow-hidden border border-gray-200 sm:rounded-lg">
                    <table class="min-w-full divide-y divide-gray-200">
                      <thead class="bg-gray-50">
                        <tr>
                          <th
                            v-for="(header, idx) in headers"
                            :key="idx"
                            scope="col"
                            class="text-xs px-3 py-2 text-left font-medium text-gray-500 tracking-wider select-none truncate"
                          >
                            <div class="flex items-center space-x-1 text-gray-500">
                              <div>{{ header.title }}</div>
                            </div>
                          </th>
                        </tr>
                      </thead>
                      <tbody class="bg-white divide-y divide-gray-200">
                        <tr v-for="(item, idx) in filteredItems" :key="idx">
                          <td class="px-3 py-2 whitespace-nowrap text-sm text-gray-600">
                            <div class="flex items-center space-x-4">
                              <div>
                                <div
                                  class="text-sm font-medium text-gray-900"
                                >{{ item.firstName }} {{ item.lastName }}</div>
                                <div class="text-sm text-gray-500">{{ item.email }}</div>
                              </div>
                            </div>
                          </td>
                          <td class="px-3 py-2 whitespace-nowrap text-sm text-gray-600">
                            <span
                              v-if="item.tenants && item.tenants.length > 0"
                            >{{ getUserTenants(item) }}</span>
                            <span v-else>?</span>
                          </td>
                          <td class="px-3 py-2 whitespace-nowrap text-sm text-gray-600">
                            <div class="flex items-center space-x-2">
                              <ButtonTertiary
                                :disabled="item.type === 0"
                                @click="impersonate(item)"
                              >{{ $t("models.user.impersonate") }}</ButtonTertiary>
                              <ButtonTertiary
                                :disabled="item.type === 0"
                                @click="changePassword(item)"
                              >{{ $t("settings.profile.changePassword") }}</ButtonTertiary>
                              <ButtonTertiary
                                :disabled="item.type === 0"
                                @click="deleteUser(item)"
                                destructive
                              >{{ $t("shared.delete") }}</ButtonTertiary>
                            </div>
                          </td>
                        </tr>
                      </tbody>
                    </table>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <SuccessModal ref="successModal" />
    <ErrorModal ref="errorModal" />
    <ConfirmModal ref="confirmDelete" @yes="confirmDeleteUser" />
  </div>
</template>

<script lang="ts">
import Vue from "vue";
import Component from "vue-class-component";
import { UserDto } from "@/application/dtos/core/users/UserDto";
import i18n from "@/locale/i18n";
import EmptyState from "@/components/ui/emptyState/EmptyState.vue";
import { TenantUserRole } from "@/application/enums/core/tenants/TenantUserRole";
import ConfirmModal from "@/components/ui/modals/ConfirmModal.vue";
import ErrorModal from "@/components/ui/modals/ErrorModal.vue";
import services from "@/services";
import SuccessModal from "@/components/ui/modals/SuccessModal.vue";
import Loading from "@/components/ui/loaders/Loading.vue";
import ButtonSecondary from "@/components/ui/buttons/ButtonSecondary.vue";
import ButtonTertiary from "@/components/ui/buttons/ButtonTertiary.vue";

@Component({
  metaInfo: {
    title: i18n.t("models.user.plural").toString(),
  },
  components: {
    EmptyState,
    ConfirmModal,
    SuccessModal,
    ErrorModal,
    Loading,
    ButtonSecondary,
    ButtonTertiary
  },
})
export default class Users extends Vue {
  $refs!: {
    confirmDelete: ConfirmModal;
    errorModal: ErrorModal;
    successModal: SuccessModal;
  };
  items: UserDto[] = [];
  loading = false;
  searchInput = "";
  headers = [
    {
      title: i18n.t("models.user.object"),
    },
    {
      title: i18n.t("models.user.tenants"),
    },
    {
      title: i18n.t("shared.actions"),
    },
  ];
  mounted() {
    this.reload();
  }
  reload() {
    this.loading = true;
    services.users
      .adminGetAll()
      .then((response: UserDto[]) => {
        this.items = response;
      })
      .finally(() => {
        this.loading = false;
      });
  }
  impersonate(user: UserDto) {
    services.authentication.impersonate(user.id)
      .catch(error => {
        this.$refs.errorModal.show(this.$t("shared.error"), this.$t(error));
      });
  }
  changePassword(user: UserDto) {
    const password = prompt(this.$t("settings.profile.changePassword") + " - " + user.email);
    if (!password || password.length < 8) {
      alert("Set a password with 8 characters minimum");
    } else if (user.type === 0) {
      alert("You cannot change password for admin user");
    } else {
      if (confirm("[ADMINISTRATOR] Update password for user " + user.email + "?")) {
        services.users
          .adminUpdatePassword(user.id, password)
          .then(() => {
            alert("Updated");
          })
          .catch((error) => {
            this.$refs.errorModal.show(this.$t("shared.error"), this.$t(error))
          });
      }
    }
  }
  getUserTenants(user: UserDto) {
    return user.tenants.map((f) => `${f.tenant?.name} (${i18n.t("settings.profile.roles." + TenantUserRole[f.role])})`).join(", ");
  }
  deleteUser(item: UserDto) {
    this.$refs.confirmDelete.value = item;
    this.$refs.confirmDelete.show(this.$t("shared.delete"), this.$t("shared.delete"), this.$t("shared.cancel"), this.$t("admin.users.deleteWarning"));
  }
  confirmDeleteUser(item) {
    this.loading = true;
    services.users
      .adminDelete(item.id)
      .then(() => {
        this.reload();
        this.$refs.successModal.show(this.$t("shared.deleted"));
      })
      .catch((error) => {
        this.$refs.errorModal.show(this.$t("shared.error"), this.$t(error));
      })
      .finally(() => {
        this.loading = false;
      });
  }
  get filteredItems(): UserDto[] {
    if (!this.items) {
      return [];
    }
    const items = this.items.filter(
      (f) =>
        f.firstName?.toString()?.toUpperCase().includes(this.searchInput.toUpperCase())
    );

    return items.sort((x, y) => {
      if (x.createdAt && y.createdAt) {
        return (x.createdAt > y.createdAt ? -1 : 1) ?? -1;
      }
      return -1;
    });
  }
}
</script>
