<script setup>
import { ref, computed, watch, onMounted, watchEffect, inject } from 'vue';
import { useRoute } from 'vue-router';
import { useI18n } from 'vue-i18n';
import { useProfiles } from '@/composables/useProfiles';
import { isAddress } from '@ethersproject/address';
import { formatBytes32String } from '@ethersproject/strings';
import { getInstance } from '@snapshot-labs/lock/plugins/vue3';
import {
  sendTransaction,
  getScores
} from '@snapshot-labs/snapshot.js/src/utils';
import getProvider from '@snapshot-labs/snapshot.js/src/utils/provider';
import {
  getDelegates,
  getDelegators,
  getDelegatesBySpace,
  contractAddress
} from '@/helpers/delegation';
import { sleep } from '@snapshot-labs/snapshot.js/src/utils';
import { useWeb3 } from '@/composables/useWeb3';
import { useTxStatus } from '@/composables/useTxStatus';
import { shorten, setPageTitle } from '@/helpers/utils';
import { useIntl } from '@/composables/useIntl';
import { debouncedWatch } from '@vueuse/core';
import { SPACE_DELEGATE_QUERY } from '@/helpers/queries';
import { useApolloQuery } from '@/composables/useApolloQuery';
import { useModal } from '@/composables/useModal';
import { useEns } from '@/composables/useEns';
import Icon from '@/components/Icon.vue';

const abi = ['function setDelegate(bytes32 id, address delegate)'];

const route = useRoute();
const { t } = useI18n();
const auth = getInstance();
const notify = inject('notify');
const { web3, web3Account } = useWeb3();
const { pendingCount } = useTxStatus();
const { formatCompactNumber } = useIntl();
const { modalAccountOpen } = useModal();

const modalOpen = ref(false);
const currentId = ref('');
const currentDelegate = ref('');
const loaded = ref(false);
const loading = ref(false);
const delegatesLoading = ref(false);
const delegates = ref([]);
const delegatesWithScore = ref([]);
const delegators = ref([]);
const specifySpaceChecked = ref(false);
const space = ref({});
const form = ref({
  address: route.params.to || '',
  id: route.params.key || ''
});

const { profiles, loadProfiles } = useProfiles();

const networkKey = computed(() => web3.value.network.key);

const { loadOwnedEnsDomains, ownedEnsDomains, validEnsTlds, isValidEnsDomain } =
  useEns();

const isEnsOwnedByWeb3Account = computed(() =>
  ownedEnsDomains.value.map(d => d.name).includes(form.value.address)
);

const validateSpaceInput = computed(() => {
  if (space.value === null) return t('delegate.noValidSpaceId');
  return false;
});

const validateToInput = computed(() => {
  if (form.value.address === '') return false;
  const address = form.value.address;
  if (!isValidEnsDomain(address) && !isAddress(address)) {
    if (address.includes('.'))
      return `${t('delegate.noValidEns')} ${t(
        'setup.supportedEnsTLDs'
      )}: ${validEnsTlds.join(', ')}`;
    else return t('delegate.noValidAddress');
  }
  if (address.toLowerCase() === web3Account.value.toLowerCase())
    return t('delegate.delegateToSelf');
  if (isEnsOwnedByWeb3Account.value) return t('delegate.delegateToSelfAddress');
  return false;
});

watch(
  web3Account,
  () => {
    loadOwnedEnsDomains();
  },
  { immediate: true }
);

const isValidForm = computed(() => {
  const address = form.value.address;
  return (
    (isValidEnsDomain(address) || isAddress(address)) &&
    address.toLowerCase() !== web3Account.value.toLowerCase() &&
    (!specifySpaceChecked.value || space.value?.id === form.value.id) &&
    !isEnsOwnedByWeb3Account.value
  );
});

function revokeDelegate(id, delegate) {
  currentId.value = id;
  currentDelegate.value = delegate;
  modalOpen.value = true;
}

watch([networkKey, web3Account], ([valN, valA], [prevN, prevA]) => {
  if (valN !== prevN || valA?.toLowerCase() !== prevA)
    getDelegationsAndDelegates();
});

const getDelegationsAndDelegatesLoading = ref(false);

async function getDelegationsAndDelegates() {
  if (web3Account.value) {
    getDelegationsAndDelegatesLoading.value = true;
    const [delegatesObj, delegatorsObj] = await Promise.all([
      getDelegates(networkKey.value, web3Account.value),
      getDelegators(networkKey.value, web3Account.value)
    ]);
    getDelegationsAndDelegatesLoading.value = false;
    delegates.value = delegatesObj.delegations;
    delegators.value = delegatorsObj.delegations;
  } else {
    // user logged out
    delegates.value = [];
    delegators.value = [];
  }
}

async function getDelegatesWithScore() {
  const delegationStrategy = space.value.strategies.filter(
    strategy => strategy.name === 'delegation'
  );
  if (delegationStrategy.length === 0) return;

  delegatesLoading.value = true;
  try {
    const delegationsRes = await Promise.all([
      getDelegatesBySpace(space.value.network, ''),
      getDelegatesBySpace(space.value.network, space.value.id)
    ]);

    const delegations = [
      ...delegationsRes[0].delegations,
      ...delegationsRes[1].delegations
    ];

    const uniqueDelegators = Array.from(
      new Set(delegations.map(d => d.delegate))
    ).map(delegate => {
      return delegations.find(a => a.delegate === delegate);
    });

    const delegatesAddresses = uniqueDelegators.map(d => d.delegate);

    const scores = await getScores(
      space.value.id,
      delegationStrategy,
      space.value.network,
      delegatesAddresses,
      'latest',
      import.meta.env.VITE_SCORES_URL + '/api/scores'
    );

    uniqueDelegators.forEach(delegate => {
      const delegationScore = scores[0];
      Object.entries(delegationScore).forEach(([address, score]) => {
        if (address === delegate.delegate) {
          delegate.score = score;
        }
      });
    });

    const sortedDelegates = uniqueDelegators
      .filter(delegate => delegate.score > 0)
      .sort((a, b) => b.score - a.score);

    delegatesWithScore.value = sortedDelegates;
    delegatesLoading.value = false;
  } catch (e) {
    delegatesLoading.value = false;
    console.log(e);
    return e;
  }
}

async function handleSubmit() {
  loading.value = true;
  try {
    let address = form.value.address;
    if (address.includes('.eth'))
      address = await getProvider('1').resolveName(address);
    let spaceId = form.value.id;
    if (!specifySpaceChecked.value) spaceId = '';
    const tx = await sendTransaction(
      auth.web3,
      contractAddress,
      abi,
      'setDelegate',
      [formatBytes32String(spaceId), address]
    );
    pendingCount.value++;
    loading.value = false;
    const receipt = await tx.wait();
    console.log('Receipt', receipt);
    await sleep(3e3);
    notify(t('notify.delegationSuccess'));
    pendingCount.value--;
    getDelegationsAndDelegates();
  } catch (e) {
    pendingCount.value--;
    console.log(e);
  }
  loading.value = false;
}

watchEffect(() => {
  loadProfiles(
    delegates.value
      .map(delegate => delegate.delegate)
      .concat(delegators.value.map(delegator => delegator.delegator))
      .concat(delegatesWithScore.value.map(delegate => delegate.delegate))
  );
});

const { apolloQuery, queryLoading: spaceLoading } = useApolloQuery();

debouncedWatch(
  () => form.value.id,
  async () => {
    if (!form.value.id) return;
    space.value = await apolloQuery(
      {
        query: SPACE_DELEGATE_QUERY,
        variables: {
          id: form.value.id
        }
      },
      'space'
    );
    if (space.value?.id === form.value?.id) {
      delegatesWithScore.value = [];
      getDelegatesWithScore();
    } else delegatesWithScore.value = [];
  },
  { immediate: true, debounce: 500 }
);

onMounted(async () => {
  if (route.params.key) specifySpaceChecked.value = true;
  setPageTitle('page.title.delegate');
  await getDelegationsAndDelegates();
  loaded.value = true;
});
</script>

<template>
  <Layout v-bind="$attrs">
    <template #content-left>
      <div class="px-4 md:px-0 mb-3">
        <router-link :to="{ path: '/' }" class="text-color">
          <Icon name="back" size="22" class="!align-middle" />
          {{ $t('backToHome') }}
        </router-link>
        <h1 v-if="loaded" v-text="$t('delegate.header')" />
      </div>
      <template v-if="loaded">
        <Block :title="$t('delegate.selectDelegate')">
          <UiInput
            v-model.trim="form.address"
            :placeholder="$t('delegate.addressPlaceholder')"
            :error="validateToInput"
            class="mt-2"
          >
            <template v-slot:label>{{ $t('delegate.to') }}</template>
          </UiInput>
          <div class="flex items-center space-x-2 px-2">
            <Checkbox v-model="specifySpaceChecked" />
            <span>{{ $t('setDelegationToSpace') }}</span>
          </div>
          <UiInput
            v-show="specifySpaceChecked"
            v-model.trim="form.id"
            placeholder="e.g. balancer.eth"
            :error="validateSpaceInput"
          >
            <template v-slot:label>{{ $t('space') }}</template>
          </UiInput>
        </Block>
        <Block
          v-if="
            delegates.length < 1 &&
            delegators.length < 1 &&
            !getDelegationsAndDelegatesLoading &&
            web3Account
          "
        >
          <Icon name="warning" class="mr-1" />
          {{ $t('delegate.noDelegationsAndDelegates') }}
        </Block>
        <Block
          v-if="delegates.length > 0"
          :slim="true"
          :title="$t('delegate.delegations')"
        >
          <div
            v-for="(delegate, i) in delegates"
            :key="i"
            :style="i === 0 && 'border: 0 !important;'"
            class="px-4 py-3 border-t flex"
          >
            <User
              :address="delegate.delegate"
              :space="{ network: networkKey }"
              :profile="profiles[delegate.delegate]"
            />
            <div
              v-text="shorten(delegate.space || $t('allSpaces'), 'choice')"
              class="flex-auto text-right link-color"
            />
            <a
              @click="revokeDelegate(delegate.space, delegate.delegate)"
              class="px-2 -mr-2 ml-2"
            >
              <Icon name="close" size="12" class="mb-1" />
            </a>
          </div>
        </Block>
        <Block
          v-if="delegators.length > 0"
          :slim="true"
          :title="$t('delegate.delegated')"
        >
          <div
            v-for="(delegator, i) in delegators"
            :key="i"
            :style="i === 0 && 'border: 0 !important;'"
            class="px-4 py-3 border-t flex"
          >
            <User
              :address="delegator.delegator"
              :space="{ network: networkKey }"
              :profile="profiles[delegator.delegator]"
            />
            <div
              v-text="shorten(delegator.space || '-', 'choice')"
              class="flex-auto text-right link-color"
            />
          </div>
        </Block>
        <Block
          v-if="space?.id && specifySpaceChecked"
          :title="$tc('delegate.topDelegates')"
          :loading="delegatesLoading"
          slim
        >
          <div
            v-for="(delegate, i) in delegatesWithScore"
            :key="i"
            :style="i === 0 && 'border: 0 !important;'"
            class="px-4 py-3 border-t flex"
          >
            <User
              :profile="profiles[delegate.delegate]"
              :address="delegate.delegate"
              :space="{ network: networkKey }"
              class="column"
            />
            <div
              class="flex-auto column text-right link-color"
              v-text="
                `${
                  delegate.score >= 0.005
                    ? formatCompactNumber(delegate.score)
                    : '< 0.01'
                } ${space.symbol}`
              "
            />
          </div>
          <div
            v-if="!delegatesLoading && delegatesWithScore.length < 1"
            class="mx-4 py-3 flex items-center"
          >
            {{ $tc('delegate.noDelegatesFoundFor', [space.id]) }}
          </div>
        </Block>
      </template>
      <PageLoading v-else />
    </template>
    <template #sidebar-right>
      <Block :title="$t('actions')">
        <UiButton
          @click="web3Account ? handleSubmit() : (modalAccountOpen = true)"
          :disabled="!isValidForm && !!web3Account"
          :loading="loading || spaceLoading"
          class="block w-full"
          primary
        >
          {{ $t('confirm') }}
        </UiButton>
      </Block>
    </template>
  </Layout>
  <teleport to="#modal">
    <ModalRevokeDelegate
      v-if="loaded"
      :open="modalOpen"
      @close="modalOpen = false"
      @reload="getDelegationsAndDelegates"
      :id="currentId"
      :delegate="currentDelegate"
      :profiles="profiles"
    />
  </teleport>
</template>
