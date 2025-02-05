<template>
  <div class="p-24px spec-container">
    <Alert
      v-if="isAlertOpen"
      v-model="isAlertOpen"
      status="error"
      :title="t('specPage.noSpecErrorTitle')"
      class="mb-16px"
      :icon="WarningIcon"
      dismissible
    >
      <p class="mb-24px">
        {{ t('specPage.noSpecErrorIntro') }} <InlineCodeFragment variant="error">
          {{ route.params.unrunnable }}
        </InlineCodeFragment>
      </p>
      <p>{{ t('specPage.noSpecErrorExplainer') }}</p>
    </Alert>
    <SpecsListHeader
      v-model="search"
      :specs-list-input-ref-fn="specsListInputRefFn"
      class="pb-32px"
      :result-count="specs.length"
      :spec-count="cachedSpecs.length"
      @show-create-spec-modal="emit('showCreateSpecModal')"
      @show-spec-pattern-modal="showSpecPatternModal = true"
    />
    <SpecPatternModal
      v-if="props.gql.currentProject"
      :show="showSpecPatternModal"
      :gql="props.gql.currentProject"
      @close="showSpecPatternModal = false"
    />
    <div
      v-if="specs.length"
      class="mb-4 grid grid-cols-2 children:font-medium children:text-gray-800 "
    >
      <div
        class="flex items-center justify-between"
        data-cy="specs-testing-type-header"
      >
        {{ props.gql.currentProject?.currentTestingType === 'component' ?
          t('specPage.componentSpecsHeader') : t('specPage.e2eSpecsHeader') }}
      </div>
      <div class="flex items-center justify-between">
        <div>{{ t('specPage.lastUpdatedHeader') }}</div>
      </div>
    </div>
    <!--
      The markup around the virtualized list is pretty delicate. We might be tempted to
      combine the `v-if="specs.length"` above and the `:class="specs.length ? 'grid': 'hidden'"` below
      into a single v-if on a `<template>` that would wrap both, but we are deliberately using
      `hidden` here to ensure that the `.spec-list-container` element stays in the DOM when
      the empty state is shown, fixing a bug that meant recovering from the empty state with the
      "Clear Search" button didn't work as expected.
    -->
    <div
      class="pb-32px spec-list-container"
      :class="specs.length ? 'grid': 'hidden'"
      v-bind="containerProps"
    >
      <div
        v-bind="wrapperProps"
        class="divide-y-1 children:h-40px"
      >
        <SpecsListRowItem
          v-for="row in list"
          :id="getIdIfDirectory(row)"
          :key="row.index"
          :data-cy="row.data.isLeaf ? 'spec-list-file' : 'spec-list-directory'"
          :data-cy-row="row.data.data?.baseName"
        >
          <template #file>
            <RouterLink
              v-if="row.data.isLeaf && row.data"
              :key="row.data.data?.absolute"
              class="focus:outline-transparent"
              :to="{ path: '/specs/runner', query: { file: row.data.data?.relative?.replace(/\\/g, '/') } }"
              data-cy="spec-item-link"
              @click.meta.prevent="handleCtrlClick"
              @click.ctrl.prevent="handleCtrlClick"
            >
              <SpecItem
                :file-name="row.data.data?.fileName || row.data.name"
                :extension="row.data.data?.specFileExtension || ''"
                :indexes="row.data.data?.fileIndexes"
                :style="{ paddingLeft: `${((row.data.depth - 2) * 10) + 22}px` }"
              />
            </RouterLink>

            <RowDirectory
              v-else
              :name="row.data.name"
              :expanded="treeSpecList[row.index].expanded.value"
              :depth="row.data.depth - 2"
              :style="{ paddingLeft: `${(row.data.depth - 2) * 10}px` }"
              :indexes="getDirIndexes(row.data)"
              :aria-controls="getIdIfDirectory(row)"
              @click="row.data.toggle"
            />
          </template>

          <template #git-info>
            <SpecListGitInfo
              v-if="row.data.isLeaf && row.data.data?.gitInfo"
              :gql="row.data.data?.gitInfo"
            />
          </template>
        </SpecsListRowItem>
      </div>
    </div>
    <NoResults
      v-show="!specs.length"
      :search="search"
      :message="t('specPage.noResultsMessage')"
      class="mt-56px"
      @clear="handleClear"
    />
  </div>
</template>

<script setup lang="ts">
import SpecsListHeader from './SpecsListHeader.vue'
import SpecListGitInfo from './SpecListGitInfo.vue'
import SpecsListRowItem from './SpecsListRowItem.vue'
import { gql, useSubscription } from '@urql/vue'
import { computed, ref, watch } from 'vue'
import { Specs_SpecsListFragment, SpecsList_GitInfoUpdatedDocument, SpecsListFragment } from '../generated/graphql'
import { useI18n } from '@cy/i18n'
import { buildSpecTree, fuzzySortSpecs, getDirIndexes, makeFuzzyFoundSpec, useCachedSpecs } from './spec-utils'
import type { FuzzyFoundSpec } from './spec-utils'
import { useCollapsibleTree } from '@packages/frontend-shared/src/composables/useCollapsibleTree'
import RowDirectory from './RowDirectory.vue'
import SpecItem from './SpecItem.vue'
import { useVirtualList } from '@packages/frontend-shared/src/composables/useVirtualList'
import NoResults from '@cy/components/NoResults.vue'
import SpecPatternModal from '../components/SpecPatternModal.vue'
import { useDebounce } from '@vueuse/core'
import Alert from '../../../frontend-shared/src/components/Alert.vue'
import InlineCodeFragment from '../../../frontend-shared/src/components/InlineCodeFragment.vue'
import WarningIcon from '~icons/cy/warning_x16.svg'
import { useRoute } from 'vue-router'

const route = useRoute()
const { t } = useI18n()

gql`
subscription SpecsList_GitInfoUpdated {
  gitInfoChange {
    id
    absolute
    gitInfo {
      ...SpecListRow
    }
  }
}
`

gql`
fragment SpecsList on Spec {
  id
  name
  specType
  absolute
  baseName
  fileName
  specFileExtension
  fileExtension
  relative
  gitInfo {
    ...SpecListRow
  }
}
`

gql`
fragment Specs_SpecsList on Query {
  currentProject {
    id
    projectRoot
    currentTestingType
    specs {
      id
      ...SpecsList
    }
    config
    ...SpecPatternModal
  }
}
`

useSubscription({ query: SpecsList_GitInfoUpdatedDocument })

const props = defineProps<{
  gql: Specs_SpecsListFragment
}>()

const emit = defineEmits<{
  (e: 'showCreateSpecModal'): void
}>()

const showSpecPatternModal = ref(false)

const isAlertOpen = ref(!!route.params?.unrunnable)

const cachedSpecs = useCachedSpecs(
  computed(() => props.gql.currentProject?.specs ?? []),
)

const search = ref('')
const specsListInputRef = ref<HTMLInputElement>()
const debouncedSearchString = useDebounce(search, 200)

const specsListInputRefFn = () => specsListInputRef

function handleClear () {
  search.value = ''
  specsListInputRef.value?.focus()
}

const specs = computed(() => {
  const specs = cachedSpecs.value.map((x) => makeFuzzyFoundSpec(x))

  if (!debouncedSearchString.value) {
    return specs
  }

  return fuzzySortSpecs(specs, debouncedSearchString.value)
})

const collapsible = computed(() => {
  return useCollapsibleTree(
    buildSpecTree<FuzzyFoundSpec<SpecsListFragment>>(specs.value), { dropRoot: true },
  )
})
const treeSpecList = computed(() => collapsible.value.tree.filter(((item) => !item.hidden.value)))

const { containerProps, list, wrapperProps, scrollTo } = useVirtualList(treeSpecList, { itemHeight: 40, overscan: 10 })

// If you are scrolled down the virtual list and list changes,
// reset scroll position to top of list
watch(() => treeSpecList.value, () => scrollTo(0))

function handleCtrlClick () {
  // noop intended to reduce the chances of opening tests multiple tabs
  // which is not a supported state in Cypress
}

function getIdIfDirectory (row) {
  if (row.data.isLeaf && row.data) {
    return undefined
  }

  return `speclist-${row.data.data.relative.replace(row.data.data.baseName, '')}`
}

</script>

<style scoped>
/** h-[calc] was getting dropped so moved to styles. Virtual list requires defined height */

/** Header is 64px */
.spec-container {
  height: calc(100vh - 64px);
}

/** Search bar is 72px + List header is 40px = 112px offset */
.spec-list-container {
  height: calc(100% - 112px)
}
</style>
