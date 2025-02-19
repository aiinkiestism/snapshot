<script setup>
import { watchEffect, computed } from 'vue';
import { shorten } from '@/helpers/utils';
import { useUsername } from '@/composables/useUsername';
import removeMd from 'remove-markdown';
import { useIntl } from '@/composables/useIntl';

const { formatRelativeTime, formatCompactNumber } = useIntl();

const props = defineProps({
  proposal: Object,
  profiles: Object
});

const body = computed(() => removeMd(props.proposal.body));

const winningChoice = computed(() =>
  props.proposal.scores.indexOf(Math.max(...props.proposal.scores))
);

const period = computed(() => {
  if (props.proposal.state === 'closed') return 'endedAgo';
  if (props.proposal.state === 'active') return 'endIn';
  return 'startIn';
});

const { address, profile, username } = useUsername();

watchEffect(() => {
  address.value = props.proposal.author;
  profile.value = props.profiles[props.proposal.author];
});
</script>

<template>
  <Block slim class="timeline-proposal transition-colors">
    <router-link
      class="p-4 block text-color"
      :to="{
        name: 'spaceProposal',
        params: { key: proposal.space.id, id: proposal.id }
      }"
    >
      <div>
        <div class="mb-2 flex justify-between items-center">
          <div class="flex items-center space-x-1">
            <Token :space="proposal.space" size="28" />
            <span class="!ml-2" v-text="proposal.space.name" />
            <span v-text="$tc('proposalBy', [username])" />
            <Badges
              :address="proposal.author"
              :members="proposal.space.members"
            />
          </div>
          <UiState :state="proposal.state" />
        </div>
        <h3 v-text="proposal.title" class="my-1" />
        <p v-text="shorten(body, 140)" class="break-words mb-2 text-md" />
        <div>
          <span
            v-if="proposal.scores_state !== 'final'"
            v-text="
              $tc(period, [
                formatRelativeTime(proposal.start),
                formatRelativeTime(proposal.end)
              ])
            "
          />
          <span
            v-if="proposal.scores_state === 'final'"
            class="mt-2 flex space-x-1 items-center"
          >
            <Icon size="20" name="check1" class="text-green" />
            <span
              >{{ shorten(proposal.choices[winningChoice], 64) }} -
              {{ formatCompactNumber(proposal.scores[winningChoice]) }}
              {{ proposal.space.symbol }}</span
            >
          </span>
        </div>
      </div>
    </router-link>
  </Block>
</template>

<style scoped lang="scss">
.timeline-proposal {
  &:hover {
    border-color: var(--link-color) !important;
  }
}
</style>
