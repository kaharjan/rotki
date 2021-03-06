<template>
  <div
    class="account-balances"
    :class="`${blockchain.toLocaleLowerCase()}-account-balances`"
  >
    <v-row>
      <v-col cols="11">
        <h3 class="text-center account-balances__title">{{ title }}</h3>
      </v-col>
      <v-col cols="1">
        <v-tooltip bottom>
          <template #activator="{ on }">
            <v-btn
              class="account-balances__refresh"
              color="primary"
              text
              icon
              :disabled="isLoading"
              v-on="on"
              @click="refresh()"
            >
              <v-icon>fa-refresh</v-icon>
            </v-btn>
          </template>
          <span>
            Refreshes the {{ blockchain }} balances ignoring any cached entries
          </span>
        </v-tooltip>
      </v-col>
    </v-row>
    <v-row>
      <v-col cols="6" offset="6">
        <tag-filter v-model="onlyTags"></tag-filter>
      </v-col>
    </v-row>
    <v-data-table
      :headers="headers"
      :items="extendedBalances"
      :loading="accountOperation || isLoading"
      loading-text="Please wait while Rotki queries the blockchain..."
      single-expand
      item-key="account"
      :expanded.sync="expanded"
      sort-by="usdValue"
      sort-desc
    >
      <template v-if="blockchain === 'ETH'" #header.usdValue>
        Total {{ currency.ticker_symbol }} value of account's assets
      </template>
      <template v-else #header.usdValue>
        {{ currency.ticker_symbol }} value
      </template>
      <template #item.identifier="{ item }">
        <v-row>
          <v-col cols="12" class="account-balances__account">
            <v-tooltip top>
              <template #activator="{ on }">
                <span class="account-balances__account__address" v-on="on">
                  <v-chip label outlined>
                    <span v-if="item.identifier !== item.account" class="pr-1">
                      {{ item.identifier }} |
                    </span>
                    <span :class="privacyMode ? 'blur-content' : ''">
                      {{ item.account | truncateAddress }}
                    </span>
                  </v-chip>
                </span>
              </template>
              <span> {{ item.account }} </span>
            </v-tooltip>
            <span v-if="item.tags">
              <tag-icon
                v-for="tag in item.tags"
                :key="tag"
                class="account-balances__tag"
                :tag="tags[tag]"
              ></tag-icon>
            </span>
          </v-col>
        </v-row>
      </template>
      <template #item.amount="{ item }">
        <amount-display :value="item.amount"></amount-display>
      </template>
      <template #item.usdValue="{ item }">
        <amount-display
          fiat-currency="USD"
          :value="item.usdValue"
        ></amount-display>
      </template>
      <template #item.actions="{ item }">
        <span class="account-balances__actions">
          <v-icon
            small
            class="mr-2"
            :disabled="accountOperation"
            @click="editAccount(item.account)"
          >
            fa-edit
          </v-icon>
          <v-icon
            small
            class="mr-2"
            :disabled="accountOperation"
            @click="toDeleteAccount = item.account"
          >
            fa-trash
          </v-icon>
        </span>
      </template>
      <template v-if="balances.length > 0" #body.append>
        <tr class="account-balances__total">
          <td>Total</td>
          <td class="text-end">
            <amount-display
              :value="extendedBalances.map(val => val.amount) | balanceSum"
            >
            </amount-display>
          </td>
          <td class="text-end">
            <amount-display
              fiat-currency="USD"
              :value="extendedBalances.map(val => val.usdValue) | balanceSum"
            >
            </amount-display>
          </td>
        </tr>
      </template>
      <template #expanded-item="{ headers, item }">
        <td :colspan="headers.length" class="account-balances__expanded">
          <account-asset-balances
            :account="item.account"
          ></account-asset-balances>
        </td>
      </template>
      <template #item.expand="{ item }">
        <span v-if="expandable && hasTokens(item.account)">
          <v-btn icon>
            <v-icon v-if="expanded.includes(item)" @click="expanded = []">
              fa fa-angle-up
            </v-icon>
            <v-icon v-else @click="expanded = [item]">fa fa-angle-down</v-icon>
          </v-btn>
        </span>
      </template>
    </v-data-table>
    <confirm-dialog
      :display="toDeleteAccount !== ''"
      title="Account delete"
      :message="`Are you sure you want to delete ${toDeleteAccount}`"
      @cancel="toDeleteAccount = ''"
      @confirm="deleteAccount()"
    ></confirm-dialog>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import { createNamespacedHelpers } from 'vuex';
import AccountForm from '@/components/accounts/AccountForm.vue';
import ConfirmDialog from '@/components/dialogs/ConfirmDialog.vue';
import AmountDisplay from '@/components/display/AmountDisplay.vue';
import TagFilter from '@/components/inputs/TagFilter.vue';
import AccountAssetBalances from '@/components/settings/AccountAssetBalances.vue';
import AssetBalances from '@/components/settings/AssetBalances.vue';
import TagIcon from '@/components/tags/TagIcon.vue';
import { AccountBalance } from '@/model/blockchain-balances';
import { Currency } from '@/model/currency';
import { TaskType } from '@/model/task-type';
import { BlockchainBalancePayload } from '@/store/balances/actions';
import { Account, Blockchain, Tags } from '@/typing/types';

const { mapGetters: mapTaskGetters } = createNamespacedHelpers('tasks');
const { mapGetters, mapState } = createNamespacedHelpers('session');
const { mapGetters: mapBalancesGetters } = createNamespacedHelpers('balances');

@Component({
  components: {
    AmountDisplay,
    AccountForm,
    TagFilter,
    TagIcon,
    AccountAssetBalances,
    AssetBalances,
    ConfirmDialog
  },
  computed: {
    ...mapTaskGetters(['isTaskRunning']),
    ...mapGetters(['floatingPrecision', 'currency']),
    ...mapState(['privacyMode', 'tags']),
    ...mapBalancesGetters([
      'exchangeRate',
      'hasTokens',
      'accountTags',
      'accountLabel'
    ])
  }
})
export default class AccountBalances extends Vue {
  @Prop({ required: true })
  balances!: AccountBalance[];
  @Prop({ required: true })
  blockchain!: Blockchain;
  @Prop({ required: true })
  title!: string;

  editedAccount = '';
  onlyTags: string[] = [];
  expanded = [];
  privacyMode!: boolean;

  isTaskRunning!: (type: TaskType) => boolean;
  accountTags!: (blockchain: Blockchain, address: string) => string[];
  accountLabel!: (blockchain: Blockchain, address: string) => string;
  tags!: Tags;

  get isLoading(): boolean {
    return this.isTaskRunning(TaskType.QUERY_BLOCKCHAIN_BALANCES);
  }

  get expandable(): boolean {
    return this.blockchain === 'ETH';
  }

  currency!: Currency;
  floatingPrecision!: number;
  exchangeRate!: (currency: string) => number;
  hasTokens!: (account: string) => boolean;

  toDeleteAccount: string = '';
  pending = false;

  headers = [
    { text: 'Account', value: 'identifier' },
    { text: this.blockchain, value: 'amount', align: 'end' },
    { text: 'USD Value', value: 'usdValue', align: 'end' },
    { text: 'Actions', value: 'actions', sortable: false, width: '50' },
    { text: '', value: 'expand', align: 'end', sortable: false }
  ];

  get visibleBalances(): AccountBalance[] {
    if (this.onlyTags.length === 0) {
      return this.balances;
    }

    const blockchain = this.blockchain;
    const accountTags = this.accountTags;
    const filteredTags = this.onlyTags;
    return this.balances.filter(({ account }) => {
      const tags = accountTags(blockchain, account);
      return filteredTags.every(tag => tags.includes(tag));
    });
  }

  get extendedBalances() {
    return this.visibleBalances.map(balance => {
      const accountLabel = this.accountLabel(this.blockchain, balance.account);

      const accountIdentifier = accountLabel ? accountLabel : balance.account;

      const accountTags = this.accountTags(this.blockchain, balance.account);

      return { identifier: accountIdentifier, tags: accountTags, ...balance };
    });
  }

  get accountOperation(): boolean {
    return (
      this.isTaskRunning(TaskType.ADD_ACCOUNT) ||
      this.isTaskRunning(TaskType.REMOVE_ACCOUNT) ||
      this.pending
    );
  }

  editAccount(account: Account) {
    const accountToEdit = { address: account, chain: this.blockchain };
    this.$emit('editAccount', accountToEdit);
    return accountToEdit;
  }

  async deleteAccount() {
    const address = this.toDeleteAccount;
    const blockchain = this.blockchain;
    this.toDeleteAccount = '';
    this.pending = true;

    await this.$store.dispatch('balances/removeAccount', {
      address,
      blockchain
    });
    this.pending = false;
  }

  refresh() {
    this.$store.dispatch('balances/fetchBlockchainBalances', {
      ignoreCache: true,
      blockchain: this.blockchain
    } as BlockchainBalancePayload);
  }
}
</script>

<style scoped lang="scss">
.account-balances {
  margin-top: 16px;
  margin-bottom: 16px;

  &__account {
    display: flex;
    flex-direction: column;

    &__address {
      font-weight: 500;
      padding-top: 6px;
      padding-bottom: 6px;
    }
  }

  &__actions {
    display: flex;
    flex-direction: row;
  }

  &__total {
    font-weight: 500;
  }

  &__expanded {
    padding: 0 !important;
  }

  &__tag {
    margin-right: 8px;
    margin-bottom: 2px;
  }
}

.blur-content {
  filter: blur(0.75em);
}
</style>
