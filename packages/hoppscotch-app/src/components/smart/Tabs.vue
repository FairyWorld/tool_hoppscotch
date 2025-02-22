<template>
  <div
    class="flex flex-1 h-full flex-nowrap"
    :class="{ 'flex-col h-auto': !vertical }"
  >
    <div
      class="relative tabs border-dividerLight"
      :class="[vertical ? 'border-r' : 'border-b', styles]"
    >
      <div class="flex flex-1">
        <div
          class="flex justify-between flex-1"
          :class="{ 'flex-col': vertical }"
        >
          <div class="flex" :class="{ 'flex-col space-y-2 p-2': vertical }">
            <button
              v-for="([tabID, tabMeta], index) in tabEntries"
              :key="`tab-${index}`"
              v-tippy="{
                theme: 'tooltip',
                placement: 'left',
                content: vertical ? tabMeta.label : null,
              }"
              class="tab"
              :class="[
                { active: modelValue === tabID },
                { vertical: vertical },
              ]"
              :aria-label="tabMeta.label || ''"
              role="button"
              @keyup.enter="selectTab(tabID)"
              @click="selectTab(tabID)"
            >
              <component
                :is="tabMeta.icon"
                v-if="tabMeta.icon"
                class="svg-icons"
              />
              <span v-else-if="tabMeta.label">{{ tabMeta.label }}</span>
              <span
                v-if="tabMeta.info && tabMeta.info !== 'null'"
                class="tab-info"
              >
                {{ tabMeta.info }}
              </span>
              <span
                v-if="tabMeta.indicator"
                class="w-1 h-1 ml-2 rounded-full bg-accentLight"
              ></span>
            </button>
          </div>
          <div class="flex items-center justify-center">
            <slot name="actions"></slot>
          </div>
        </div>
      </div>
    </div>
    <div
      class="w-full h-full contents"
      :class="{
        '!flex flex-col flex-1 overflow-y-auto ': vertical,
      }"
    >
      <slot></slot>
    </div>
  </div>
</template>

<script setup lang="ts">
import { pipe } from "fp-ts/function"
import { not } from "fp-ts/Predicate"
import * as A from "fp-ts/Array"
import * as O from "fp-ts/Option"
import type { Component } from "vue"
import { ref, ComputedRef, computed, provide } from "vue"
import { throwError } from "~/helpers/functional/error"

export type TabMeta = {
  label: string | null
  icon: string | Component | null
  indicator: boolean
  info: string | null
}

export type TabProvider = {
  // Whether inactive tabs should remain rendered
  renderInactive: ComputedRef<boolean>
  activeTabID: ComputedRef<string>
  addTabEntry: (tabID: string, meta: TabMeta) => void
  updateTabEntry: (tabID: string, newMeta: TabMeta) => void
  removeTabEntry: (tabID: string) => void
}

const props = defineProps({
  styles: {
    type: String,
    default: "",
  },
  renderInactiveTabs: {
    type: Boolean,
    default: false,
  },
  vertical: {
    type: Boolean,
    default: false,
  },
  modelValue: {
    type: String,
    required: true,
  },
})

const emit = defineEmits<{
  (e: "update:modelValue", newTabID: string): void
}>()

const tabEntries = ref<Array<[string, TabMeta]>>([])

const addTabEntry = (tabID: string, meta: TabMeta) => {
  tabEntries.value = pipe(
    tabEntries.value,
    O.fromPredicate(not(A.exists(([id]) => id === tabID))),
    O.map(A.append([tabID, meta] as [string, TabMeta])),
    O.getOrElseW(() => throwError(`Tab with duplicate ID created: '${tabID}'`))
  )
}

const updateTabEntry = (tabID: string, newMeta: TabMeta) => {
  tabEntries.value = pipe(
    tabEntries.value,
    A.findIndex(([id]) => id === tabID),
    O.chain((index) =>
      pipe(
        tabEntries.value,
        A.updateAt(index, [tabID, newMeta] as [string, TabMeta])
      )
    ),
    O.getOrElseW(() => throwError(`Failed to update tab entry: ${tabID}`))
  )
}

const removeTabEntry = (tabID: string) => {
  tabEntries.value = pipe(
    tabEntries.value,
    A.findIndex(([id]) => id === tabID),
    O.chain((index) => pipe(tabEntries.value, A.deleteAt(index))),
    O.getOrElseW(() => throwError(`Failed to remove tab entry: ${tabID}`))
  )

  // If we tried to remove the active tabEntries, switch to first tab entry
  if (props.modelValue === tabID)
    if (tabEntries.value.length > 0) selectTab(tabEntries.value[0][0])
}

provide<TabProvider>("tabs-system", {
  renderInactive: computed(() => props.renderInactiveTabs),
  activeTabID: computed(() => props.modelValue),
  addTabEntry,
  updateTabEntry,
  removeTabEntry,
})

const selectTab = (id: string) => {
  emit("update:modelValue", id)
}
</script>

<style lang="scss" scoped>
.tabs {
  @apply flex;
  @apply whitespace-nowrap;
  @apply overflow-auto;
  @apply flex-shrink-0;

  // &::after {
  //   @apply absolute;
  //   @apply inset-x-0;
  //   @apply bottom-0;
  //   @apply bg-dividerLight;
  //   @apply z-1;
  //   @apply h-0.5;
  //   content: "";
  // }

  .tab {
    @apply relative;
    @apply flex;
    @apply flex-shrink-0;
    @apply items-center;
    @apply justify-center;
    @apply py-2 px-4;
    @apply text-secondary;
    @apply font-semibold;
    @apply cursor-pointer;
    @apply hover: text-secondaryDark;
    @apply focus: outline-none;
    @apply focus-visible: text-secondaryDark;

    .tab-info {
      @apply inline-flex;
      @apply items-center;
      @apply justify-center;
      @apply w-5;
      @apply h-4;
      @apply ml-2;
      @apply text-8px;
      @apply border border-divider;
      @apply rounded;
      @apply text-secondaryLight;
    }

    &::after {
      @apply absolute;
      @apply left-4;
      @apply right-4;
      @apply bottom-0;
      @apply bg-transparent;
      @apply z-2;
      @apply h-0.5;
      content: "";
    }

    &:focus::after {
      @apply bg-divider;
    }

    &.active {
      @apply text-secondaryDark;

      .tab-info {
        @apply text-secondary;
        @apply border-dividerDark;
      }

      &::after {
        @apply bg-accent;
      }
    }

    &.vertical {
      @apply p-2;
      @apply rounded;

      &:focus::after {
        @apply hidden;
      }

      &.active {
        @apply text-accent;

        .tab-info {
          @apply text-secondary;
          @apply border-dividerDark;
        }

        &::after {
          @apply hidden;
        }
      }
    }
  }
}
</style>
