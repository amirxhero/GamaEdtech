<template>
  <div>
    <!-- Mobile View Transaction History -->
    <div class="transaction-history-section mt-8 d-block d-sm-none">
      <div class="d-flex justify-space-between align-center mb-2">
        <div class="text-h5 primary-gray-700 font-weight-bold pb-2">
          Transaction History
        </div>
        <v-btn
          icon
          @click="toggleChart"
          class="chart-toggle-icon"
          :color="isChartVisible ? 'primary' : ''"
        >
          <v-icon>mdi-chart-line-variant</v-icon>
        </v-btn>
      </div>

      <div class="transaction-tabs-wrapper mb-4">
        <div class="d-flex align-center justify-space-between">
          <v-tabs
            v-model="activeTab"
            grow
            background-color="transparent"
            slider-color="amber"
            @change="handleTabChange"
          >
            <v-tab class="text-subtitle-2">All</v-tab>
            <v-tab class="text-subtitle-2">Earned</v-tab>
            <v-tab class="text-subtitle-2">Spent</v-tab>
          </v-tabs>
          <v-icon class="ml-1">mdi-tune-vertical</v-icon>
        </div>
      </div>

      <!-- Loading State -->
      <div v-if="loading && !hasMoreItems" class="py-4">
        <v-skeleton-loader
          v-for="i in 3"
          :key="i"
          type="list-item-two-line"
          class="mb-3"
        ></v-skeleton-loader>
      </div>

      <!-- Transactions List with Infinite Scroll -->
      <v-list v-else class="transaction-list pa-0" flat>
        <div
          ref="infiniteContainer"
          class="infinite-container"
          @scroll="handleScroll"
        >
          <v-list-item
            v-for="(transaction, i) in filteredMobileTransactions"
            :key="i"
            class="transaction-item py-3"
          >
            <v-list-item-content>
              <div class="d-flex flex-column">
                <div class="transaction-name mb-1">
                  {{ transaction.description }}
                </div>
                <div class="d-flex align-center">
                  <div class="" v-if="transaction.isDebit">
                    <div class="state-icon-wrapper spent">
                      <span class="state-icon spent align-center">
                        <v-icon color="red" size="18">mdi-tray-arrow-up</v-icon>
                      </span>
                      <span class="state-text spent ml-2">Spent</span>
                    </div>
                  </div>
                  <div class="" v-else>
                    <div class="state-icon-wrapper earned">
                      <span class="state-icon earned align-center">
                        <v-icon color="green" size="18"
                          >mdi-tray-arrow-down</v-icon
                        >
                      </span>
                      <span class="state-text earned ml-2">Earned</span>
                    </div>
                  </div>
                </div>
              </div>
            </v-list-item-content>

            <v-list-item-action class="transaction-details">
              <div class="d-flex flex-column align-end">
                <div class="transaction-amount mb-1">
                  <span class="font-weight-medium">{{
                    transaction.points
                  }}</span>
                  <span class="ml-1 caption grey--text">$GET</span>
                </div>
                <div class="d-flex align-center">
                  <v-icon x-small class="mr-1 grey--text text--lighten-1"
                    >mdi-clock-outline</v-icon
                  >
                  <span class="caption grey--text text--darken-1">{{
                    transaction.creationDate
                  }}</span>
                </div>
              </div>
            </v-list-item-action>
          </v-list-item>

          <!-- Loading More Indicator -->
          <div v-if="loading && hasMoreItems" class="pa-4 text-center">
            <v-progress-circular
              indeterminate
              color="primary"
              size="24"
            ></v-progress-circular>
          </div>

          <!-- End of List Message -->
          <div
            v-if="!hasMoreItems && filteredMobileTransactions.length > 0"
            class="pa-4 text-center"
          >
            <span class="caption grey--text">No more transactions</span>
          </div>

          <!-- Empty States for Different Tabs -->
          <div
            v-if="filteredMobileTransactions.length === 0 && !loading"
            class="text-center py-8"
          >
            <!-- All Transactions Empty State -->
            <template v-if="activeTab === 0">
              <v-icon size="64" color="grey lighten-2"
                >mdi-wallet-outline</v-icon
              >
              <p class="mt-4 grey--text">No transactions found</p>
            </template>

            <!-- Earned Transactions Empty State -->
            <template v-if="activeTab === 1">
              <v-icon size="64" color="green lighten-4">mdi-cash-plus</v-icon>
              <p class="mt-4 grey--text">No earned transactions yet</p>
              <p class="caption grey--text text--darken-1">
                Complete tasks to earn $GET
              </p>
            </template>

            <!-- Spent Transactions Empty State -->
            <template v-if="activeTab === 2">
              <v-icon size="64" color="red lighten-4">mdi-cash-minus</v-icon>
              <p class="mt-4 grey--text">No spent transactions yet</p>
              <p class="caption grey--text text--darken-1">
                Your spending history will appear here
              </p>
            </template>
          </div>
        </div>
      </v-list>
    </div>

    <!-- Desktop View Transaction History -->
    <div class="mt-8 d-none d-sm-block" elevation="0">
      <div class="text-h5 primary-gray-700 font-weight-bold pb-2">
        Transaction History
      </div>

      <div class="transaction-tabs mb-4">
        <v-tabs
          v-model="activeTab"
          background-color="transparent"
          color="primary"
          @change="handleTabChange"
        >
          <v-tab class="font-weight-regular">All</v-tab>
          <v-tab class="font-weight-regular">Earned</v-tab>
          <v-tab class="font-weight-regular">Spent</v-tab>
        </v-tabs>
      </div>

      <v-data-table
        :headers="headers"
        :items="filteredTransactions"
        :items-per-page="pageSize"
        :page.sync="currentPage"
        :server-items-length="totalRecords"
        :loading="loading"
        :footer-props="{
          'items-per-page-options': [10, 25, 50],
          'items-per-page-text': 'Rows per page:',
          showFirstLastPage: true,
          class: 'custom-table-footer',
        }"
        @update:page="handlePageChange"
        @update:items-per-page="handlePageSizeChange"
        loading-text="Loading transactions..."
        no-data-text="No transactions found"
        class="elevation-0 transaction-table"
      >
        <template v-slot:item.description="{ item }">
          {{ item.description }}
        </template>

        <template v-slot:item.points="{ item }">
          <div class="d-flex align-center">
            <span class="font-weight-bold">{{ item.points }}</span>
            <span class="ml-1 caption grey--text">$GET</span>
          </div>
        </template>

        <template v-slot:item.isDebit="{ item }">
          <div class="d-flex align-center">
            <div v-if="item.isDebit">
              <div class="state-icon-wrapper spent">
                <span class="state-icon spent align-center">
                  <v-icon color="red" size="18">mdi-tray-arrow-up</v-icon>
                </span>
                <span class="state-text spent ml-2">Spent</span>
              </div>
            </div>
            <div v-else>
              <div class="state-icon-wrapper earned">
                <span class="state-icon earned align-center">
                  <v-icon color="green" size="18">mdi-tray-arrow-down</v-icon>
                </span>
                <span class="state-text earned ml-2">Earned</span>
              </div>
            </div>
          </div>
        </template>

        <template v-slot:item.creationDate="{ item }">
          <div class="d-flex align-center">
            <v-icon small class="mr-1 primary-gray-300"
              >mdi-clock-outline</v-icon
            >
            <span class="primary-gray-500">{{ item.creationDate }}</span>
          </div>
        </template>
      </v-data-table>
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "TransactionHistory",
  props: {
    isChartVisible: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      activeTab: 0,
      loading: false,
      transactions: [],
      mobileTransactions: [], // Separate array for mobile view
      token: "",
      currentPage: 1,
      pageSize: 10,
      totalRecords: 0,
      hasMoreItems: true,
      loadingMore: false,
      headers: [
        {
          text: "Description",
          value: "description",
          sortable: true,
          align: "start",
          class: "font-weight-medium",
        },
        {
          text: "Amount",
          value: "points",
          sortable: true,
          align: "start",
          class: "font-weight-medium",
        },
        {
          text: "State",
          value: "isDebit",
          sortable: true,
          align: "start",
          class: "font-weight-medium",
        },
        {
          text: "Date",
          value: "creationDate",
          sortable: true,
          align: "start",
          class: "font-weight-medium",
        },
      ],
    };
  },
  created() {
    this.getToken();
    this.fetchTransactions();
  },
  computed: {
    filteredTransactions() {
      if (this.activeTab === 0) {
        // All transactions
        return this.transactions;
      } else if (this.activeTab === 1) {
        // Earned transactions (isDebit = false)
        return this.transactions.filter((transaction) => !transaction.isDebit);
      } else if (this.activeTab === 2) {
        // Spent transactions (isDebit = true)
        return this.transactions.filter((transaction) => transaction.isDebit);
      }
      return this.transactions;
    },
    filteredMobileTransactions() {
      if (this.activeTab === 0) {
        // All transactions
        return this.mobileTransactions;
      } else if (this.activeTab === 1) {
        // Earned transactions (isDebit = false)
        return this.mobileTransactions.filter(
          (transaction) => !transaction.isDebit
        );
      } else if (this.activeTab === 2) {
        // Spent transactions (isDebit = true)
        return this.mobileTransactions.filter(
          (transaction) => transaction.isDebit
        );
      }
      return this.mobileTransactions;
    },
    startIndex() {
      return (this.currentPage - 1) * this.pageSize + 1;
    },
    endIndex() {
      const end = this.currentPage * this.pageSize;
      return end > this.totalRecords ? this.totalRecords : end;
    },
  },
  methods: {
    getToken() {
      if (process.client) {
        this.token = localStorage.getItem("v2_token") || "";
      }
    },
    async fetchTransactions(loadMore = false) {
      if (this.loading || (loadMore && !this.hasMoreItems)) return;

      this.loading = true;
      const skip = loadMore
        ? this.mobileTransactions.length
        : (this.currentPage - 1) * this.pageSize;

      try {
        const response = await this.$axios.$get("/api/v2/transactions", {
          params: {
            "PagingDto.PageFilter.Size": this.pageSize,
            "PagingDto.PageFilter.Skip": skip,
            "PagingDto.PageFilter.ReturnTotalRecordsCount": true,
          },
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        });

        if (response.succeeded && response.data) {
          if (loadMore) {
            this.mobileTransactions = [
              ...this.mobileTransactions,
              ...response.data.list,
            ];
          } else {
            this.mobileTransactions = response.data.list;
            this.transactions = response.data.list; // Keep desktop view updated
          }
          this.totalRecords = response.data.totalRecordsCount;
          this.hasMoreItems =
            this.mobileTransactions.length < this.totalRecords;
        }
      } catch (err) {
        if (err.response && err.response.status == 403) {
          this.$auth.logout();
        }
        console.error("Error fetching transactions:", err);
      } finally {
        this.loading = false;
      }
    },
    handleScroll(event) {
      const container = event.target;
      const scrollPosition = container.scrollTop + container.clientHeight;
      const scrollThreshold = container.scrollHeight - 100; // Load more when within 100px of bottom

      if (
        scrollPosition >= scrollThreshold &&
        !this.loading &&
        this.hasMoreItems
      ) {
        this.fetchTransactions(true);
      }
    },
    handleTabChange() {
      // No need to reset or fetch, just let the computed property handle filtering
    },
    getStateColor(state) {
      switch (state) {
        case "Pending":
          return "amber";
        case "Spent":
          return "red";
        case "Earned":
          return "green";
        default:
          return "grey";
      }
    },
    getStateColorClass(state) {
      switch (state) {
        case "Pending":
          return "pending-bg";
        case "Spent":
          return "spent-bg";
        case "Earned":
          return "earned-bg";
        default:
          return "grey-bg";
      }
    },
    getStateTextColorClass(state) {
      switch (state) {
        case "Pending":
          return "amber--text";
        case "Spent":
          return "red--text";
        case "Earned":
          return "green--text";
        default:
          return "grey--text";
      }
    },
    getStateIcon(state) {
      switch (state) {
        case "Pending":
          return "mdi-dots-horizontal";
        case "Spent":
          return "mdi-arrow-down";
        case "Earned":
          return "mdi-arrow-up";
        default:
          return "mdi-help-circle-outline";
      }
    },
    toggleChart() {
      this.$emit("toggle-chart");
    },
    handlePageChange(page) {
      this.currentPage = page;
      this.fetchTransactions();
    },
    handlePageSizeChange(size) {
      this.pageSize = size;
      this.currentPage = 1; // Reset to first page when changing page size
      this.fetchTransactions();
    },
  },
});
</script>

<style scoped>
.transaction-history-section {
}

.transaction-tabs-wrapper {
  border-bottom: 1px solid #f0f0f0;
  width: 100%;
}
.transaction-tabs-wrapper .v-tabs {
  width: 80%;
}

.transaction-list {
  margin-top: 8px;
}

.transaction-item {
  border-bottom: 1px solid #f5f5f5;
  padding: 12px 0;
}

.transaction-name {
  font-size: 14px;
  font-weight: 500;
  color: #333;
}

.transaction-amount {
  font-size: 14px;
}

.state-icon-bg {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.state-icon-bg-desktop {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Desktop Transaction History Styles */
.transaction-tabs >>> .v-tab {
  min-width: 0;
  text-transform: none;
  font-weight: 400;
  padding: 0 16px;
  font-size: 14px;
}

.transaction-tabs >>> .v-tabs-slider {
  height: 2px;
}

.transaction-table >>> .v-data-table__wrapper {
  overflow-x: auto;
}

.transaction-table >>> thead {
  background-color: #f5f7fa;
  border-radius: 8px;
}

.transaction-table >>> thead th {
  font-size: 14px !important;
  color: #4b5563 !important;
  font-weight: 500 !important;
  padding: 12px 16px !important;
  text-transform: none !important;
  letter-spacing: 0 !important;
}

.transaction-table >>> thead th.sortable .v-data-table-header__icon {
  color: #9ca3af;
  margin-left: 4px;
}

.transaction-table >>> tbody td {
  padding: 16px !important;
  height: auto !important;
}

.transaction-chart {
  margin-bottom: 16px;
  position: relative;
  width: 100%;
}

/* State Colors */
.pending-bg {
  background-color: #ffc107;
}

.spent-bg {
  background-color: #ff5252;
}

.earned-bg {
  background-color: #4caf50;
}

.pending-bg-desktop {
  background-color: rgba(255, 193, 7, 0.15);
}

.pending-bg-desktop i {
  color: #ffc107 !important;
}

.spent-bg-desktop {
  background-color: rgba(255, 82, 82, 0.15);
}

.spent-bg-desktop i {
  color: #ff5252 !important;
}

.earned-bg-desktop {
  background-color: rgba(76, 175, 80, 0.15);
}

.earned-bg-desktop i {
  color: #4caf50 !important;
}

.state-icon-wrapper {
  display: flex;
  align-items: center;
}

.state-icon-wrapper .state-icon {
  width: 24px;
  height: 24px;
  border-radius: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.state-icon-wrapper .state-icon.pending {
  background-color: rgba(255, 193, 7, 0.1);
}

.state-icon-wrapper .state-icon.spent {
  background-color: rgba(255, 82, 82, 0.1);
}

.state-icon-wrapper .state-icon.earned {
  background-color: rgba(76, 175, 80, 0.1);
}

.state-icon-wrapper .state-text.pending {
  color: #ffc107;
  font-size: 14px;
}

.state-icon-wrapper .state-text.spent {
  color: #ff5252;
  font-size: 14px;
}

.state-icon-wrapper .state-text.earned {
  color: #4caf50;
  font-size: 14px;
}

/* Add missing skeleton loading styles at the end of the style section */
::v-deep .v-skeleton-loader__list-item-two-line {
  background: rgba(0, 0, 0, 0.05) !important;
  border-radius: 8px;
}

.chart-toggle-icon {
  cursor: pointer;
}

/* Pagination Styles */
::v-deep .custom-table-footer {
  font-size: 14px !important;
}

::v-deep .v-data-footer__select {
  font-size: 14px !important;
}

::v-deep .v-data-footer__pagination {
  font-size: 14px !important;
}

::v-deep .v-data-footer__select .v-select {
  margin: 0 10px !important;
}

::v-deep .v-data-footer__select .v-select__selection {
  font-size: 14px !important;
}

::v-deep .v-data-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 16px;
  margin-top: 15px;
}

::v-deep .v-data-footer__icons-before,
::v-deep .v-data-footer__icons-after {
  font-size: 18px !important;
}

.mobile-pagination {
  border-top: 1px solid #f0f0f0;
  margin-top: 16px;
  background: white;
}

.mobile-rows-select {
  min-width: 140px;
}

::v-deep .mobile-select {
  max-width: 65px;
}

::v-deep .mobile-select .v-input__slot {
  min-height: 24px !important;
  padding: 0 !important;
}

::v-deep .mobile-select .v-select__selection {
  font-size: 12px;
  color: rgba(0, 0, 0, 0.87);
}

::v-deep .mobile-select .v-input__append-inner {
  margin-top: 2px !important;
}

::v-deep .mobile-select .v-icon {
  font-size: 16px;
}

::v-deep .mobile-pagination .v-btn.v-btn--icon.v-size--x-small {
  width: 20px;
  height: 20px;
}

::v-deep .mobile-pagination .v-btn.v-btn--icon.v-size--x-small .v-icon {
  font-size: 16px;
}

.caption {
  font-size: 12px !important;
  color: rgba(0, 0, 0, 0.6);
}

.infinite-container {
  height: calc(100vh - 300px);
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}

.transaction-list {
  margin-top: 8px;
  height: 100%;
}

/* Add smooth scrolling behavior */
.infinite-container {
  scroll-behavior: smooth;
}

/* Hide scrollbar for Chrome, Safari and Opera */
.infinite-container::-webkit-scrollbar {
  display: none;
}

/* Hide scrollbar for IE, Edge and Firefox */
.infinite-container {
  -ms-overflow-style: none; /* IE and Edge */
  scrollbar-width: none; /* Firefox */
}

.empty-state-icon {
  opacity: 0.7;
  margin-bottom: 16px;
}

.empty-state-text {
  font-size: 16px;
  font-weight: 500;
  color: rgba(0, 0, 0, 0.6);
  margin-bottom: 8px;
}

.empty-state-subtext {
  font-size: 14px;
  color: rgba(0, 0, 0, 0.38);
}
</style>
