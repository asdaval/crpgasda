<script setup lang="ts">
import { useClipboard } from '@vueuse/core';
import { itemSellCostPenalty } from '@root/data/constants.json';
import { ItemCompareMode, type CompareItemsResult, type ItemFlat } from '@/models/item';
import { type UserItem } from '@/models/user';
import {
  getAggregationsConfig,
  getVisibleAggregationsConfig,
} from '@/services/item-search-service';
import {
  getItemImage,
  computeSalePrice,
  computeBrokenItemRepairCost,
  canUpgrade,
} from '@/services/item-service';
import { notify } from '@/services/notification-service';
import { t } from '@/services/translate-service';
import { parseTimestamp } from '@/utils/date';
import { omitPredicate } from '@/utils/object';

const {
  item,
  userItem,
  compareResult,
  equipped = false,
} = defineProps<{
  item: ItemFlat;
  userItem: UserItem;
  compareResult: CompareItemsResult | undefined;
  equipped?: boolean;
}>();

const userItemToReplaceSalePrice = computed(() => {
  const { price, graceTimeEnd } = computeSalePrice(userItem);

  return {
    price,
    graceTimeEnd:
      graceTimeEnd === null ? null : parseTimestamp(graceTimeEnd.valueOf() - new Date().valueOf()),
  };
});

const repairCost = computed(() => computeBrokenItemRepairCost(item.price));

const emit = defineEmits<{
  sell: [];
  repair: [];
  upgrade: [];
  reforge: [];
}>();

const omitEmptyParam = (field: keyof ItemFlat) => {
  if (Array.isArray(item[field]) && (item[field] as string[]).length === 0) {
    return false;
  }

  if (item[field] === 0) {
    return false;
  }

  return true;
};

const aggregationsConfig = computed(() =>
  omitPredicate(
    getVisibleAggregationsConfig(getAggregationsConfig(item.type, item.weaponClass)),
    (key: keyof ItemFlat) => omitEmptyParam(key)
  )
);

const isUpgradable = computed(() => canUpgrade(item.type));

const { copy } = useClipboard();

const onNameCopy = () => {
  copy(item.name);
  notify(t('action.copied'));
};
</script>

<template>
  <article>
    <div class="relative mb-3">
      <img
        :src="getItemImage(item.baseId)"
        :alt="item.name"
        :title="item.name"
        class="pointer-events-none w-full select-none object-contain"
      />

      <div class="absolute -left-0.5 -top-0.5 z-10">
        <OIcon
          v-if="userItem.isBroken"
          icon="error"
          size="2xl"
          class="cursor-default text-status-danger opacity-80 hover:opacity-100"
          v-tooltip="$t('character.inventory.item.broken.tooltip.title')"
        />

        <ItemRankIcon
          v-if="userItem.item.rank > 0"
          :rank="userItem.item.rank"
          class="cursor-default opacity-80 hover:opacity-100"
        />
      </div>

      <Tag
        v-if="equipped"
        class="absolute bottom-0 right-0 z-10 cursor-default text-primary opacity-75 hover:opacity-100"
        icon="check"
        variant="primary"
        rounded
        v-tooltip="$t('character.inventory.item.equipped')"
      />
    </div>

    <h3 class="z-10 mb-6 font-semibold text-content-100">
      {{ item.name }}
      <Tag icon="popup" variant="primary" rounded size="sm" @click="onNameCopy" />
    </h3>

    <div class="grid grid-cols-2 gap-4">
      <div class="space-y-1">
        <h6 class="text-2xs text-content-300">Type/Class</h6>
        <div class="flex flex-wrap gap-2">
          <ItemParam :item="item" field="type" />
          <ItemParam v-if="item.weaponClass !== null" :item="item" field="weaponClass" />
        </div>
      </div>

      <div v-for="(_agg, field) in aggregationsConfig" class="space-y-1">
        <VTooltip :delay="{ show: 600 }">
          <h6 class="text-2xs text-content-300">
            {{ $t(`item.aggregations.${field}.title`) }}
          </h6>

          <template #popper>
            <div class="prose prose-invert">
              <h5 class="text-content-100">
                {{ $t(`item.aggregations.${field}.title`) }}
              </h5>
              <p v-if="$t(`item.aggregations.${field}.description`)">
                {{ $t(`item.aggregations.${field}.description`) }}
              </p>
            </div>
          </template>
        </VTooltip>

        <ItemParam
          :item="item"
          :field="field"
          :isCompare="compareResult !== undefined"
          :compareMode="ItemCompareMode.Absolute"
          :bestValue="compareResult !== undefined ? compareResult[field] : undefined"
        >
          <template v-if="field === 'price'" #default="{ rawBuckets }">
            <Coin :value="(rawBuckets as number)" />
          </template>

          <template v-if="field === 'upkeep'" #default="{ rawBuckets }">
            <Coin>
              {{ $t('item.format.upkeep', { upkeep: $n(rawBuckets as number) }) }}
            </Coin>
          </template>
        </ItemParam>
      </div>
    </div>

    <div class="-mx-4 -mb-4 mt-6 flex items-center gap-2 bg-base-400 p-2">
      <ConfirmActionTooltip
        class="flex-1"
        :confirmLabel="$t('action.sell')"
        :title="$t('character.inventory.item.sell.confirm')"
        @confirm="emit('sell')"
      >
        <OButton variant="secondary" expanded rounded size="lg">
          <i18n-t
            scope="global"
            keypath="character.inventory.item.sell.title"
            tag="span"
            class="flex gap-2"
          >
            <template #price>
              <Coin :value="userItemToReplaceSalePrice.price" />
            </template>
          </i18n-t>

          <VTooltip>
            <Tag
              v-if="userItemToReplaceSalePrice.graceTimeEnd !== null"
              size="sm"
              variant="success"
              label="100%"
            />
            <Tag
              v-else
              size="sm"
              variant="danger"
              :label="$n(itemSellCostPenalty, 'percent', { minimumFractionDigits: 0 })"
            />

            <template #popper>
              <i18n-t
                v-if="userItemToReplaceSalePrice.graceTimeEnd !== null"
                scope="global"
                keypath="character.inventory.item.sell.freeRefund"
                tag="div"
              >
                <template #dateTime>
                  <span class="font-bold">
                    {{ $t('dateTimeFormat.mm', { ...userItemToReplaceSalePrice.graceTimeEnd }) }}
                  </span>
                </template>
              </i18n-t>

              <i18n-t
                v-else
                scope="global"
                keypath="character.inventory.item.sell.penaltyRefund"
                tag="div"
              >
                <template #penalty>
                  <span class="font-bold text-status-danger">
                    {{ $n(itemSellCostPenalty, 'percent', { minimumFractionDigits: 0 }) }}
                  </span>
                </template>
              </i18n-t>
            </template>
          </VTooltip>
        </OButton>
      </ConfirmActionTooltip>

      <ConfirmActionTooltip v-if="userItem.isBroken" @confirm="emit('repair')">
        <VTooltip>
          <OButton iconRight="repair" variant="secondary" size="lg" rounded />

          <template #popper>
            <i18n-t
              scope="global"
              keypath="character.inventory.item.repair.tooltip.title"
              tag="span"
              class="flex gap-2"
            >
              <template #price>
                <Coin :value="repairCost" />
              </template>
            </i18n-t>
          </template>
        </VTooltip>
      </ConfirmActionTooltip>

      <Modal v-if="isUpgradable" closable :autoHide="false">
        <OButton
          variant="secondary"
          rounded
          size="lg"
          iconLeft="blacksmith"
          v-tooltip="$t('character.inventory.item.upgrade.upgradesTitle')"
        />
        <template #popper>
          <div class="container pb-2 pt-12">
            <CharacterInventoryItemUpgrades
              :item="item"
              :cols="aggregationsConfig"
              @upgrade="emit('upgrade')"
              @reforge="emit('reforge')"
            />
          </div>
        </template>
      </Modal>
    </div>
  </article>
</template>
