<template>
  <div ref="wrapper" :class=wrapperClass>
    <template v-if="state.isReady">
      <HiddenFields />
      <Control ref="control">
        <template v-if="$slots['value-label']" #value-label="{ node }">
          <slot name="value-label"
                :node="node"/>
        </template>
      </Control>
      <MenuPortal v-if="appendToBody" ref="portal" />
      <Menu v-else ref="rmenu">
        <template v-if="$slots['before-list']" #before-list>
          <slot name="before-list" />
        </template>
        <template v-if="$slots['after-list']" #after-list>
          <slot name="after-list" />
        </template>
        <template v-if="$slots['option-label']" #option-label="{ node, shouldShowCount, count }">
          <slot name="option-label"
                :node="node"
                :shouldShowCount="shouldShowCount"
                :count="count"/>
        </template>
      </Menu>
    </template>
  </div>
</template>

<script>
export default {
  name: 'vue-treeselect',
}
</script>

<script setup>
import { computed, defineProps, reactive, watch, defineEmits, getCurrentInstance, ref, nextTick, onMounted, provide, useSlots, toRaw } from 'vue';
import fuzzysearch from 'fuzzysearch'
import {
  warning,
  onLeftClick, scrollIntoView,
  isNaN, isPromise, once,
  identity, constant, createMap,
  quickDiff, last as getLast, includes, find, removeFromArray,
  getId,
} from '../utils'
import {
  NO_PARENT_NODE,
  UNCHECKED, INDETERMINATE, CHECKED,
  LOAD_ROOT_OPTIONS, LOAD_CHILDREN_OPTIONS, ASYNC_SEARCH,
  ALL, BRANCH_PRIORITY, LEAF_PRIORITY, ALL_WITH_INDETERMINATE,
  ALL_CHILDREN, ALL_DESCENDANTS, LEAF_CHILDREN, LEAF_DESCENDANTS,
  ORDER_SELECTED, LEVEL, INDEX,
} from '../constants'
import HiddenFields from '@/components/HiddenFields.vue'
import Control from '@/components/Control.vue'
import Menu from '@/components/Menu.vue'
import MenuPortal from '@/components/MenuPortal.vue'

const control = ref(null);
const wrapper = ref(null);
const portal = ref(null);
const rmenu = ref(null);
const instanceId = getId();

const sortValueByIndex = (a, b) => {
  let i = 0
  do {
    if (a.level < i) return -1
    if (b.level < i) return 1
    if (a.index[i] !== b.index[i]) return a.index[i] - b.index[i]
    i++
  } while (true)
}
const sortValueByLevel = (a, b) => {
  return a.level === b.level
      ? sortValueByIndex(a, b)
      : a.level - b.level
}
const createAsyncOptionsStates = () => {
  return {
    isLoaded: false,
    isLoading: false,
    loadingError: '',
  }
}
const stringifyOptionPropValue = (value) => {
  if (typeof value === 'string') return value
  if (typeof value === 'number' && !isNaN(value)) return value + ''
  // istanbul ignore next
  return ''
}
const match = (enableFuzzyMatch, needle, haystack) => {
  return enableFuzzyMatch
      ? fuzzysearch(needle, haystack)
      : includes(haystack, needle)
}
const getErrorMessage = (err) => {
  return err.message || /* istanbul ignore next */String(err)
}
const extractCheckedNodeIdsFromValue = () => {
  if (props.modelValue == null) return []

  if (props.valueFormat === 'id') {
    return props.multiple
        ? props.modelValue.slice()
        : [ props.modelValue ]
  }

  return (props.multiple ? props.modelValue : [ props.modelValue ])
      .map(node => enhancedNormalizer(node))
      .map(node => node.id)
};

const emit = defineEmits(['update:modelValue', 'search-change', 'close', 'open', 'select', 'deselect'])

const slots = useSlots();
const props = defineProps({
  /**
   * Whether to allow resetting value even if there are disabled selected nodes.
   */
  allowClearingDisabled: {
    type: Boolean,
    default: false,
  },

  /**
   * When an ancestor node is selected/deselected, whether its disabled descendants should be selected/deselected.
   * You may want to use this in conjunction with `allowClearingDisabled` prop.
   */
  allowSelectingDisabledDescendants: {
    type: Boolean,
    default: false,
  },

  /**
   * Whether the menu should be always open.
   */
  alwaysOpen: {
    type: Boolean,
    default: false,
  },

  /**
   * Append the menu to <body />?
   */
  appendToBody: {
    type: Boolean,
    default: false,
  },

  /**
   * Whether to enable async search mode.
   */
  async: {
    type: Boolean,
    default: false,
  },

  /**
   * Automatically focus the component on mount?
   */
  autoFocus: {
    type: Boolean,
    default: false,
  },

  /**
   * Automatically load root options on mount. When set to `false`, root options will be loaded when the menu is opened.
   */
  autoLoadRootOptions: {
    type: Boolean,
    default: true,
  },

  /**
   * When user deselects a node, automatically deselect its ancestors. Applies to flat mode only.
   */
  autoDeselectAncestors: {
    type: Boolean,
    default: false,
  },

  /**
   * When user deselects a node, automatically deselect its descendants. Applies to flat mode only.
   */
  autoDeselectDescendants: {
    type: Boolean,
    default: false,
  },

  /**
   * When user selects a node, automatically select its ancestors. Applies to flat mode only.
   */
  autoSelectAncestors: {
    type: Boolean,
    default: false,
  },

  /**
   * When user selects a node, automatically select its descendants. Applies to flat mode only.
   */
  autoSelectDescendants: {
    type: Boolean,
    default: false,
  },

  /**
   * Whether pressing backspace key removes the last item if there is no text input.
   */
  backspaceRemoves: {
    type: Boolean,
    default: true,
  },

  /**
   * Function that processes before clearing all input fields.
   * Return `false` to prevent value from being cleared.
   * @type {function(): (boolean|Promise<boolean>)}
   */
  beforeClearAll: {
    type: Function,
    default: constant(true),
  },

  /**
   * Show branch nodes before leaf nodes?
   */
  branchNodesFirst: {
    type: Boolean,
    default: false,
  },

  /**
   * Should cache results of every search request?
   */
  cacheOptions: {
    type: Boolean,
    default: true,
  },

  /**
   * Show an "×" button that resets value?
   */
  clearable: {
    type: Boolean,
    default: true,
  },

  /**
   * Title for the "×" button when `multiple: true`.
   */
  clearAllText: {
    type: String,
    default: 'Clear all',
  },

  /**
   * Whether to clear the search input after selecting.
   * Use only when `multiple` is `true`.
   * For single-select mode, it **always** clears the input after selecting an option regardless of the prop value.
   */
  clearOnSelect: {
    type: Boolean,
    default: false,
  },

  /**
   * Title for the "×" button.
   */
  clearValueText: {
    type: String,
    default: 'Clear value',
  },

  /**
   * Whether to close the menu after selecting an option?
   * Use only when `multiple` is `true`.
   */
  closeOnSelect: {
    type: Boolean,
    default: true,
  },

  /**
   * How many levels of branch nodes should be automatically expanded when loaded.
   * Set `Infinity` to make all branch nodes expanded by default.
   */
  defaultExpandLevel: {
    type: Number,
    default: 0,
  },

  /**
   * The default set of options to show before the user starts searching. Used for async search mode.
   * When set to `true`, the results for search query as a empty string will be autoloaded.
   * @type {boolean|node[]}
   */
  defaultOptions: {
    default: false,
  },

  /**
   * Whether pressing delete key removes the last item if there is no text input.
   */
  deleteRemoves: {
    type: Boolean,
    default: true,
  },

  /**
   * Delimiter to use to join multiple values for the hidden field value.
   */
  delimiter: {
    type: String,
    default: ',',
  },

  /**
   * Only show the nodes that match the search value directly, excluding its ancestors.
   *
   * @type {Object}
   */
  flattenSearchResults: {
    type: Boolean,
    default: false,
  },

  /**
   * Prevent branch nodes from being selected?
   */
  disableBranchNodes: {
    type: Boolean,
    default: false,
  },

  /**
   * Disable the control?
   */
  disabled: {
    type: Boolean,
    default: false,
  },

  /**
   * Disable the fuzzy matching functionality?
   */
  disableFuzzyMatching: {
    type: Boolean,
    default: false,
  },

  /**
   * Whether to enable flat mode or not. Non-flat mode (default) means:
   *   - Whenever a branch node gets checked, all its children will be checked too
   *   - Whenever a branch node has all children checked, the branch node itself will be checked too
   * Set `true` to disable this mechanism
   */
  flat: {
    type: Boolean,
    default: false,
  },

  /**
   * Joins multiple values into a single form field with the `delimiter` (legacy mode).
   */
  joinValues: {
    type: Boolean,
    default: false,
  },

  /**
   * Limit the display of selected options.
   * The rest will be hidden within the limitText string.
   */
  limit: {
    type: Number,
    default: Infinity,
  },

  /**
   * Function that processes the message shown when selected elements pass the defined limit.
   * @type {function(number): string}
   */
  limitText: {
    type: Function,
    default: function limitTextDefault(count) { // eslint-disable-line func-name-matching
      return `and ${count} more`
    },
  },

  /**
   * Text displayed when loading options.
   */
  loadingText: {
    type: String,
    default: 'Loading...',
  },

  /**
   * Used for dynamically loading options.
   * @type {function({action: string, callback: (function((Error|string)=): void), parentNode: node=, instanceId}): void}
   */
  loadOptions: {
    type: Function,
  },

  /**
   * Which node properties to filter on.
   */
  matchKeys: {
    type: Array,
    default: constant([ 'label' ]),
  },

  /**
   * Sets `maxHeight` style value of the menu.
   */
  maxHeight: {
    type: Number,
    default: 300,
  },

  /**
   * Set `true` to allow selecting multiple options (a.k.a., multi-select mode).
   */
  multiple: {
    type: Boolean,
    default: false,
  },

  /**
   * Generates a hidden <input /> tag with this field name for html forms.
   */
  name: {
    type: String,
  },

  /**
   * Text displayed when a branch node has no children.
   */
  noChildrenText: {
    type: String,
    default: 'No sub-options.',
  },

  /**
   * Text displayed when there are no available options.
   */
  noOptionsText: {
    type: String,
    default: 'No options available.',
  },

  /**
   * Text displayed when there are no matching search results.
   */
  noResultsText: {
    type: String,
    default: 'No results found...',
  },

  /**
   * Used for normalizing source data.
   * @type {function(node, instanceId): node}
   */
  normalizer: {
    type: Function,
    default: identity,
  },

  /**
   * By default (`auto`), the menu will open below the control. If there is not
   * enough space, vue-treeselect will automatically flip the menu.
   * You can use one of other four options to force the menu to be always opened
   * to specified direction.
   * Acceptable values:
   *   - `"auto"`
   *   - `"below"`
   *   - `"bottom"`
   *   - `"above"`
   *   - `"top"`
   */
  openDirection: {
    type: String,
    default: 'auto',
    validator(value) {
      const acceptableValues = [ 'auto', 'top', 'bottom', 'above', 'below' ]
      return includes(acceptableValues, value)
    },
  },

  /**
   * Whether to automatically open the menu when the control is clicked.
   */
  openOnClick: {
    type: Boolean,
    default: true,
  },

  /**
   * Whether to automatically open the menu when the control is focused.
   */
  openOnFocus: {
    type: Boolean,
    default: false,
  },

  /**
   * Array of available options.
   * @type {node[]}
   */
  options: {
    type: Array,
  },

  /**
   * Field placeholder, displayed when there's no value.
   */
  placeholder: {
    type: String,
    default: 'Select...',
  },

  /**
   * Applies HTML5 required attribute when needed.
   */
  required: {
    type: Boolean,
    default: false,
  },

  /**
   * Text displayed asking user whether to retry loading children options.
   */
  retryText: {
    type: String,
    default: 'Retry?',
  },

  /**
   * Title for the retry button.
   */
  retryTitle: {
    type: String,
    default: 'Click to retry',
  },

  /**
   * Enable searching feature?
   */
  searchable: {
    type: Boolean,
    default: true,
  },

  /**
   * Search in ancestor nodes too.
   */
  searchNested: {
    type: Boolean,
    default: false,
  },

  /**
   * Text tip to prompt for async search.
   */
  searchPromptText: {
    type: String,
    default: 'Type to search...',
  },

  /**
   * Whether to show a children count next to the label of each branch node.
   */
  showCount: {
    type: Boolean,
    default: false,
  },

  /**
   * Used in conjunction with `showCount` to specify which type of count number should be displayed.
   * Acceptable values:
   *   - "ALL_CHILDREN"
   *   - "ALL_DESCENDANTS"
   *   - "LEAF_CHILDREN"
   *   - "LEAF_DESCENDANTS"
   */
  showCountOf: {
    type: String,
    default: ALL_CHILDREN,
    validator(value) {
      const acceptableValues = [ ALL_CHILDREN, ALL_DESCENDANTS, LEAF_CHILDREN, LEAF_DESCENDANTS ]
      return includes(acceptableValues, value)
    },
  },

  /**
   * Whether to show children count when searching.
   * Fallbacks to the value of `showCount` when not specified.
   * @type {boolean}
   */
  showCountOnSearch: null,

  /**
   * In which order the selected options should be displayed in trigger & sorted in `value` array.
   * Used for multi-select mode only.
   * Acceptable values:
   *   - "ORDER_SELECTED"
   *   - "LEVEL"
   *   - "INDEX"
   */
  sortValueBy: {
    type: String,
    default: ORDER_SELECTED,
    validator(value) {
      const acceptableValues = [ ORDER_SELECTED, LEVEL, INDEX ]
      return includes(acceptableValues, value)
    },
  },

  /**
   * Tab index of the control.
   */
  tabIndex: {
    type: Number,
    default: 0,
  },

  /**
   * The value of the control.
   * Should be `id` or `node` object for single-select mode, or an array of `id` or `node` object for multi-select mode.
   * Its format depends on the `valueFormat` prop.
   * For most cases, just use `v-model` instead.
   * @type {?Array}
   */
  modelValue: null,

  /**
   * Which kind of nodes should be included in the `value` array in multi-select mode.
   * Acceptable values:
   *   - "ALL" - Any node that is checked will be included in the `value` array
   *   - "BRANCH_PRIORITY" (default) - If a branch node is checked, all its descendants will be excluded in the `value` array
   *   - "LEAF_PRIORITY" - If a branch node is checked, this node itself and its branch descendants will be excluded from the `value` array but its leaf descendants will be included
   *   - "ALL_WITH_INDETERMINATE" - Any node that is checked will be included in the `value` array, plus indeterminate nodes
   */
  valueConsistsOf: {
    type: String,
    default: BRANCH_PRIORITY,
    validator(value) {
      const acceptableValues = [ ALL, BRANCH_PRIORITY, LEAF_PRIORITY, ALL_WITH_INDETERMINATE ]
      return includes(acceptableValues, value)
    },
  },

  /**
   * Format of `value` prop.
   * Note that, when set to `"object"`, only `id` & `label` properties are required in each `node` object in `value` prop.
   * Acceptable values:
   *   - "id"
   *   - "object"
   */
  valueFormat: {
    type: String,
    default: 'id',
  },

  /**
   * z-index of the menu.
   */
  zIndex: {
    type: [ Number, String ],
    default: 999,
  },
})

const state = reactive({
  trigger: {
    // Is the control focused?
    isFocused: false,
    // User entered search query - value of the input.
    searchQuery: '',
  },
  menu: {
    // Is the menu opened?
    isOpen: false,
    // Id of current highlighted option.
    current: null,
    // The scroll position before last menu closing.
    lastScrollPosition: 0,
    // Which direction to open the menu.
    placement: 'bottom',
  },
  forest: {
    // Normalized options.
    normalizedOptions: [],
    // <id, node> map for quick look-up.
    nodeMap: createMap(),
    // <id, checkedState> map, used for multi-select mode.
    checkedStateMap: createMap(),
    // Id list of all selected options.
    selectedNodeIds: extractCheckedNodeIdsFromValue(),
    // <id, true> map for fast checking:
    //   if (forest.selectedNodeIds.indexOf(id) !== -1) forest.selectedNodeMap[id] === true
    selectedNodeMap: createMap(),
  },
  rootOptionsStates: createAsyncOptionsStates(),
  localSearch: {
    // Has user entered any query to search local options?
    active: false,
    // Has any options matched the search query?
    noResults: false,
    // <id, countObject> map for counting matched children/descendants.
    countMap: createMap(),
  },
  // <searchQuery, remoteSearchEntry> map.
  remoteSearch: createMap(),
  isReady: false,
  _blurOnSelect: false,
})

/** Computed */
const instance = computed(() => ({
  name: props.name || 'vue-treeselect',
  hasValue: hasValue,
  trigger: state.trigger,
  forest: state.forest,
  localSearch: state.localSearch,
  remoteSearch: state.remoteSearch,
  isReady: state.isReady,
  rootOptionsStates: state.rootOptionsStates,
  internalValue: internalValue,
  selectedNodes: selectedNodes,
  $slots: slots,
  $refs: {
    control: control.value,
    wrapper: wrapper.value,
    portal: portal.value,
    menu: rmenu.value,
  },
  menu: state.menu,
  single: single,
  verifyProps,
  resetFlags,
  initialize,
  getInstanceId,
  getValue,
  getNode,
  createFallbackNode,
  extractCheckedNodeIdsFromValue,
  extractNodeFromValue,
  fixSelectedNodeIds,
  keepDataOfSelectedNodes,
  isSelected,
  traverseDescendantsBFS,
  traverseDescendantsDFS,
  traverseAllNodesDFS,
  traverseAllNodesByIndex,
  toggleClickOutsideEvent,
  getValueContainer,
  getInput,
  focusInput,
  blurInput,
  handleMouseDown,
  handleClickOutside,
  handleLocalSearch,
  getRemoteSearchEntry,
  shouldExpand,
  shouldOptionBeIncludedInSearchResult,
  shouldShowOptionInMenu,
  getControl,
  getMenu,
  setCurrentHighlightedOption,
  resetHighlightedOptionWhenNecessary,
  highlightFirstOption,
  highlightPrevOption,
  highlightNextOption,
  highlightLastOption,
  resetSearchQuery,
  closeMenu,
  openMenu,
  toggleMenu,
  toggleExpanded,
  buildForestState,
  enhancedNormalizer,
  normalize,
  loadRootOptions,
  loadChildrenOptions,
  callLoadOptionsProp,
  checkDuplication,
  verifyNodeShape,
  select,
  clear,
  _selectNode,
  _deselectNode,
  addValue,
  removeValue,
  removeLastValue,
  saveMenuScrollPosition,
  restoreMenuScrollPosition,
  handleRemoteSearch,
  allowClearingDisabled: props.allowClearingDisabled,
  allowSelectingDisabledDescendants: props.allowSelectingDisabledDescendants,
  alwaysOpen: props.alwaysOpen,
  appendToBody: props.appendToBody,
  async: props.async,
  autoFocus: props.autoFocus,
  autoLoadRootOptions: props.autoLoadRootOptions,
  autoDeselectAncestors: props.autoDeselectAncestors,
  autoDeselectDescendants: props.autoDeselectDescendants,
  autoSelectAncestors: props.autoSelectAncestors,
  autoSelectDescendants: props.autoSelectDescendants,
  backspaceRemoves: props.backspaceRemoves,
  beforeClearAll: props.beforeClearAll,
  branchNodesFirst: props.branchNodesFirst,
  cacheOptions: props.cacheOptions,
  clearable: props.clearable,
  clearAllText: props.clearAllText,
  clearOnSelect: props.clearOnSelect,
  clearValueText: props.clearValueText,
  closeOnSelect: props.closeOnSelect,
  defaultExpandLevel: props.defaultExpandLevel,
  defaultOptions: props.defaultOptions,
  deleteRemoves: props.deleteRemoves,
  delimiter: props.delimiter,
  flattenSearchResults: props.flattenSearchResults,
  disableBranchNodes: props.disableBranchNodes,
  disabled: props.disabled,
  disableFuzzyMatching: props.disableFuzzyMatching,
  flat: props.flat,
  instanceId: instanceId,
  joinValues: props.joinValues,
  limit: props.limit,
  limitText: props.limitText,
  loadingText: props.loadingText,
  loadOptions: props.loadOptions,
  matchKeys: props.matchKeys,
  maxHeight: props.maxHeight,
  multiple: props.multiple,
  noChildrenText: props.noChildrenText,
  noOptionsText: props.noOptionsText,
  noResultsText: props.noResultsText,
  normalizer: props.normalizer,
  openDirection: props.openDirection,
  openOnClick: props.openOnClick,
  openOnFocus: props.openOnFocus,
  options: props.options,
  placeholder: props.placeholder,
  required: props.required,
  retryText: props.retryText,
  retryTitle: props.retryTitle,
  searchable: props.searchable,
  searchNested: props.searchNested,
  searchPromptText: props.searchPromptText,
  showCount: props.showCount,
  showCountOf: props.showCountOf,
  showCountOnSearch: props.showCountOnSearch,
  sortValueBy: props.sortValueBy,
  tabIndex: props.tabIndex,
  modelValue: props.modelValue,
  valueConsistsOf: props.valueConsistsOf,
  valueFormat: props.valueFormat,
  zIndex: props.zIndex,
}))
const wrapperClass = computed(() => ({
  'vue-treeselect': true,
  'vue-treeselect--single': single.value,
  'vue-treeselect--multi': props.multiple,
  'vue-treeselect--searchable': props.searchable,
  'vue-treeselect--disabled': props.disabled,
  'vue-treeselect--focused': state.trigger.isFocused,
  'vue-treeselect--has-value': hasValue.value,
  'vue-treeselect--open': state.menu.isOpen,
  'vue-treeselect--open-above': state.menu.placement === 'top',
  'vue-treeselect--open-below': state.menu.placement === 'bottom',
  'vue-treeselect--branch-nodes-disabled': props.disableBranchNodes,
  'vue-treeselect--append-to-body': props.appendToBody,
}))
const selectedNodes = computed(() => {
  return state.forest.selectedNodeIds.map(getNode)
});
const internalValue = computed(() => {
  let internalValue

  // istanbul ignore else
  if (single.value || props.flat || props.disableBranchNodes || props.valueConsistsOf === ALL) {
    internalValue = state.forest.selectedNodeIds.slice()
  } else if (props.valueConsistsOf === BRANCH_PRIORITY) {
    internalValue = state.forest.selectedNodeIds.filter(id => {
      const node = getNode(id)
      if (node.isRootNode) return true
      return !isSelected(node.parentNode)
    })
  } else if (props.valueConsistsOf === LEAF_PRIORITY) {
    internalValue = state.forest.selectedNodeIds.filter(id => {
      const node = getNode(id)
      if (node.isLeaf) return true
      return node.children.length === 0
    })
  } else if (props.valueConsistsOf === ALL_WITH_INDETERMINATE) {
    const indeterminateNodeIds = []
    internalValue = state.forest.selectedNodeIds.slice()
    selectedNodes.value.forEach(selectedNode => {
      selectedNode.ancestors.forEach(ancestor => {
        if (includes(indeterminateNodeIds, ancestor.id)) return
        if (includes(internalValue, ancestor.id)) return
        indeterminateNodeIds.push(ancestor.id)
      })
    })
    internalValue.push(...indeterminateNodeIds)
  }

  if (props.sortValueBy === LEVEL) {
    internalValue.sort((a, b) => sortValueByLevel(getNode(a), getNode(b)))
  } else if (props.sortValueBy === INDEX) {
    internalValue.sort((a, b) => sortValueByIndex(getNode(a), getNode(b)))
  }

  return internalValue
});
const hasValue = computed(() => {
  return internalValue.value.length > 0
});
const single = computed(() => {
  return !props.multiple
});
const visibleOptionIds = computed(() => {
  const visibleOptionIds = []

  traverseAllNodesByIndex(node => {
    if (!state.localSearch.active || shouldOptionBeIncludedInSearchResult(node)) {
      visibleOptionIds.push(node.id)
    }
    // Skip the traversal of descendants of a branch node if it's not expanded.
    if (node.isBranch && !shouldExpand(node)) {
      return false
    }
  })

  return visibleOptionIds
});
const hasVisibleOptions = computed(() => {
  return visibleOptionIds.value.length !== 0
});
const showCountOnSearchComputed = computed(() => {
  // Vue doesn't allow setting default prop value based on another prop value.
  // So use computed property as a workaround.
  // https://github.com/vuejs/vue/issues/6358
  return typeof props.showCountOnSearch === 'boolean'
      ? props.showCountOnSearch
      : props.showCount
});
const hasBranchNodes = computed(() => {
  return state.forest.normalizedOptions.some(rootNode => rootNode.isBranch)
});
const shouldFlattenOptions = computed(() => {
  return state.localSearch.active && props.flattenSearchResults
});
const computedSearchQuery = computed(() => {
  return state.trigger.searchQuery;
})
const computedModelValue = computed(() => {
  return props.modelValue;
})
const CalwaysOpen = computed(() => props.alwaysOpen)
const CbranchNodesFirst = computed(() => props.branchNodesFirst)
const Cdisabled = computed(() => props.disabled)
const Cflat = computed(() => props.flat)
const CmatchKeys = computed(() => props.matchKeys)
const Cmultiple = computed(() => props.multiple)
const Coptions = computed(() => props.options)

/** Computed END */

/** Methods */

const verifyProps = () => {
  warning(
      () => props.async ? props.searchable : true,
      () => 'For async search mode, the value of "searchable" prop must be true.',
  )

  if (props.options == null && !props.loadOptions) {
    warning(
        () => false,
        () => 'Are you meant to dynamically load options? You need to use "loadOptions" prop.',
    )
  }

  if (props.flat) {
    warning(
        () => props.multiple,
        () => 'You are using flat mode. But you forgot to add "multiple=true"?',
    )
  }

  if (!props.flat) {
    const propNames = [
      'autoSelectAncestors',
      'autoSelectDescendants',
      'autoDeselectAncestors',
      'autoDeselectDescendants',
    ]

    propNames.forEach(propName => {
      warning(
          () => !props[propName],
          () => `"${propName}" only applies to flat mode.`,
      )
    })
  }
};
const resetFlags = () => {
  state._blurOnSelect = false;
};
const initialize = () => {
  const options = props.async
      ? getRemoteSearchEntry().options
      : toRaw(props.options)

  if (Array.isArray(options)) {
    // In case we are re-initializing options, keep the old state tree temporarily.
    const prevNodeMap = state.forest.nodeMap
    state.forest.nodeMap = createMap()
    keepDataOfSelectedNodes(prevNodeMap)
    state.forest.normalizedOptions = normalize(NO_PARENT_NODE, options, prevNodeMap)
    // Cases that need fixing `selectedNodeIds`:
    //   1) Children options of a checked node have been delayed loaded,
    //      we should also mark these children as checked. (multi-select mode)
    //   2) Root options have been delayed loaded, we need to initialize states
    //      of these nodes. (multi-select mode)
    //   3) Async search mode.
    fixSelectedNodeIds(internalValue.value)
    const { searchQuery } = state.trigger
    state.remoteSearch[searchQuery] = getRemoteSearchEntry();
  } else {
    state.forest.normalizedOptions = []
  }
};
const getInstanceId = () => {
  return instanceId
};
const getValue = () => {
  if (props.valueFormat === 'id') {
    return props.multiple
        ? internalValue.value.slice()
        : internalValue.value[0]
  }

  const rawNodes = internalValue.value.map(id => getNode(id).raw)
  return props.multiple ? rawNodes : rawNodes[0]
};
const getNode = (nodeId) => {
  warning(
      () => nodeId != null,
      () => `Invalid node id: ${nodeId}`,
  )

  if (nodeId == null) return null

  return nodeId in state.forest.nodeMap
      ? state.forest.nodeMap[nodeId]
      : createFallbackNode(nodeId)
};
const createFallbackNode = (id) => {
  // In case there is a default selected node that is not loaded into the tree yet,
  // we create a fallback node to keep the component working.
  // When the real data is loaded, we'll override this fake node.

  const raw = extractNodeFromValue(id)
  const label = enhancedNormalizer(raw).label || `${id} (unknown)`
  const fallbackNode = {
    id,
    label,
    ancestors: [],
    parentNode: NO_PARENT_NODE,
    isFallbackNode: true,
    isRootNode: true,
    isLeaf: true,
    isBranch: false,
    isDisabled: false,
    isNew: false,
    index: [ -1 ],
    level: 0,
    raw,
  }

  return state.forest.nodeMap[id] = fallbackNode
};

const extractNodeFromValue = (id) => {
  const defaultNode = { id }

  if (props.valueFormat === 'id') {
    return defaultNode
  }

  const valueArray = props.multiple
      ? Array.isArray(props.modelValue) ? props.modelValue : []
      : props.modelValue ? [ props.modelValue ] : []
  const matched = find(
      valueArray,
      node => node && enhancedNormalizer(node).id === id,
  )

  return matched || defaultNode
};
const fixSelectedNodeIds = (nodeIdListOfPrevValue) => {
  let nextSelectedNodeIds = []

  // istanbul ignore else
  if (single.value || props.flat || props.disableBranchNodes || props.valueConsistsOf === ALL) {
    nextSelectedNodeIds = nodeIdListOfPrevValue
  } else if (props.valueConsistsOf === BRANCH_PRIORITY) {
    nodeIdListOfPrevValue.forEach(nodeId => {
      nextSelectedNodeIds.push(nodeId)
      const node = getNode(nodeId)
      if (node.isBranch) traverseDescendantsBFS(node, descendant => {
        nextSelectedNodeIds.push(descendant.id)
      })
    })
  } else if (props.valueConsistsOf === LEAF_PRIORITY) {
    const map = createMap()
    const queue = nodeIdListOfPrevValue.slice()
    while (queue.length) {
      const nodeId = queue.shift()
      const node = getNode(nodeId)
      nextSelectedNodeIds.push(nodeId)
      if (node.isRootNode) continue
      if (!(node.parentNode.id in map)) map[node.parentNode.id] = node.parentNode.children.length
      if (--map[node.parentNode.id] === 0) queue.push(node.parentNode.id)
    }
  } else if (props.valueConsistsOf === ALL_WITH_INDETERMINATE) {
    const map = createMap()
    const queue = nodeIdListOfPrevValue.filter(nodeId => {
      const node = getNode(nodeId)
      return node.isLeaf || node.children.length === 0
    })
    while (queue.length) {
      const nodeId = queue.shift()
      const node = getNode(nodeId)
      nextSelectedNodeIds.push(nodeId)
      if (node.isRootNode) continue
      if (!(node.parentNode.id in map)) map[node.parentNode.id] = node.parentNode.children.length
      if (--map[node.parentNode.id] === 0) queue.push(node.parentNode.id)
    }
  }
};
const keepDataOfSelectedNodes = (prevNodeMap) => {
  // In case there is any selected node that is not present in the new `options` array.
  // It could be useful for async search mode.
  state.forest.selectedNodeIds.forEach(id => {
    if (!prevNodeMap[id]) return
    state.forest.nodeMap[id] = {
      ...prevNodeMap[id],
      isFallbackNode: true,
    }
  })
};
const isSelected = (node) => {
  // whether a node is selected (single-select mode) or fully-checked (multi-select mode)
  return state.forest.selectedNodeMap[node.id] === true
};
const traverseDescendantsBFS = (parentNode, callback) => {
  // istanbul ignore if
  if (!parentNode.isBranch) return
  const queue = parentNode.children.slice()
  while (queue.length) {
    const currNode = queue[0]
    if (currNode.isBranch) queue.push(...currNode.children)
    callback(currNode)
    queue.shift()
  }
};
const traverseDescendantsDFS = (parentNode, callback) => {
  if (!parentNode.isBranch) return
  parentNode.children.forEach(child => {
    // deep-level node first
    traverseDescendantsDFS(child, callback)
    callback(child)
  })
};
const traverseAllNodesDFS = (callback) => {
  state.forest.normalizedOptions.forEach(rootNode => {
    // deep-level node first
    traverseDescendantsDFS(rootNode, callback)
    callback(rootNode)
  })
};
const traverseAllNodesByIndex = (callback) => {
  const walk = parentNode => {
    parentNode.children.forEach(child => {
      if (callback(child) !== false && child.isBranch) {
        walk(child)
      }
    })
  }

  // To simplify the code logic of traversal,
  // we create a fake root node that holds all the root options.
  walk({ children: state.forest.normalizedOptions })
};
const toggleClickOutsideEvent = (enabled) => {
  if (enabled) {
    document.addEventListener('mousedown', handleClickOutside, false)
  } else {
    document.removeEventListener('mousedown', handleClickOutside, false)
  }
};
const getValueContainer = () => {
  return control.value.$refs['value-container']
};
const getInput = () => {
  return getValueContainer().$refs.input
};
const focusInput = () => {
  getInput().focus()
};
const blurInput = () => {
  getInput().blur()
};
const handleMouseDown = () => { onLeftClick((evt) => {
    evt.preventDefault()
    evt.stopPropagation()

    if (props.disabled) return

    const isClickedOnValueContainer = getValueContainer().$el.contains(evt.target)
    if (isClickedOnValueContainer && !state.menu.isOpen && (props.openOnClick || state.trigger.isFocused)) {
      openMenu()
    }

    if (state._blurOnSelect) {
      blurInput()
    } else {
      // Focus the input or prevent blurring.
      focusInput()
    }

    resetFlags()
  })};
const handleClickOutside = (evt) => {
  // istanbul ignore else
  if (wrapper.value && !wrapper.value.contains(evt.target)) {
    blurInput()
    closeMenu()
  }
};
const handleLocalSearch = () => {
  const { searchQuery } = state.trigger
  const done = () => resetHighlightedOptionWhenNecessary(true)

  if (!searchQuery) {
    // Exit sync search mode.
    state.localSearch.active = false
    state.localSearch.noResults = false
    return done()
  }

  // Enter sync search mode.
  state.localSearch.active = true

  // Reset states.
  state.localSearch.noResults = true
  traverseAllNodesDFS(node => {
    if (node.isBranch) {
      node.isExpandedOnSearch = false
      node.showAllChildrenOnSearch = false
      node.isMatched = false
      node.hasMatchedDescendants = false

      state.localSearch.countMap[node.id] = {
        [ALL_CHILDREN]: 0,
        [ALL_DESCENDANTS]: 0,
        [LEAF_CHILDREN]: 0,
        [LEAF_DESCENDANTS]: 0,
      }
    }
  })

  const lowerCasedSearchQuery = searchQuery.trim().toLocaleLowerCase()
  const splitSearchQuery = lowerCasedSearchQuery.replace(/\s+/g, ' ').split(' ')
  traverseAllNodesDFS(node => {
    if (props.searchNested && splitSearchQuery.length > 1) {
      node.isMatched = splitSearchQuery.every(filterValue =>
          match(false, filterValue, node.nestedSearchLabel),
      )
    } else {
      node.isMatched = props.matchKeys.some(matchKey =>
          match(!props.disableFuzzyMatching, lowerCasedSearchQuery, node.lowerCased[matchKey]),
      )
    }

    if (node.isMatched) {
      state.localSearch.noResults = false
      node.ancestors.forEach(ancestor => state.localSearch.countMap[ancestor.id][ALL_DESCENDANTS]++)
      if (node.isLeaf) node.ancestors.forEach(ancestor => state.localSearch.countMap[ancestor.id][LEAF_DESCENDANTS]++)
      if (node.parentNode !== NO_PARENT_NODE) {
        state.localSearch.countMap[node.parentNode.id][ALL_CHILDREN] += 1
        // istanbul ignore else
        if (node.isLeaf) state.localSearch.countMap[node.parentNode.id][LEAF_CHILDREN] += 1
      }
    }

    if (
        (node.isMatched || (node.isBranch && node.isExpandedOnSearch)) &&
        node.parentNode !== NO_PARENT_NODE
    ) {
      node.parentNode.isExpandedOnSearch = true
      node.parentNode.hasMatchedDescendants = true
    }
  })

  done()
};
const getRemoteSearchEntry = () => {
  const { searchQuery } = state.trigger
  const entry = state.remoteSearch[searchQuery] || {
    ...createAsyncOptionsStates(),
    options: [],
  }

  if (searchQuery === '') {
    if (Array.isArray(props.defaultOptions)) {
      entry.options = props.defaultOptions
      entry.isLoaded = true
      return entry
    } else if (props.defaultOptions !== true) {
      entry.isLoaded = true
      return entry
    }
  }

  if (!state.remoteSearch[searchQuery]) {
    // $ set(remoteSearch, searchQuery, entry)
    state.remoteSearch[searchQuery] = entry;
  }

  return state.remoteSearch[searchQuery];
};
const shouldExpand = (node) => {
  return state.localSearch.active ? node.isExpandedOnSearch : node.isExpanded
};
const shouldOptionBeIncludedInSearchResult = (node) => {
  // 1) This option is matched.
  if (node.isMatched) return true
  // 2) This option is not matched, but has matched descendant(s).
  if (node.isBranch && node.hasMatchedDescendants && !props.flattenSearchResults) return true
  // 3) This option's parent has no matched descendants,
  //    but after being expanded, all its children should be shown.
  if (!node.isRootNode && node.parentNode.showAllChildrenOnSearch) return true
  // 4) The default case.
  return false
};
const shouldShowOptionInMenu = (node) => {
  if (state.localSearch.active && !shouldOptionBeIncludedInSearchResult(node)) {
    return false
  }
  return true
};
const getControl = () => {
  return control.value.$el
};
const getMenu = () => {
  return rmenu.value.$el
};
const setCurrentHighlightedOption = (node, scroll = true) => {
  const prev = state.menu.current
  if (prev != null && prev in state.forest.nodeMap) {
    state.forest.nodeMap[prev].isHighlighted = false
  }

  state.menu.current = node.id
  node.isHighlighted = true

  if (state.menu.isOpen && scroll) {
    const scrollToOption = () => {
      const $menu = getMenu()

      const $option = $menu.querySelector(`.vue-treeselect__option[data-id="${node.id}"]`)
      if ($option) scrollIntoView($menu, $option)
    }

    // In case `openMenu()` is just called and the menu is not rendered yet.
    if (getMenu()) {
      scrollToOption()
    } else {
      // istanbul ignore next
      nextTick(scrollToOption)
    }
  }
};
const resetHighlightedOptionWhenNecessary = (forceReset = false) => {
  const { current } = state.menu

  if (
      forceReset || current == null ||
      !(current in state.forest.nodeMap) ||
      !shouldShowOptionInMenu(getNode(current))
  ) {
    highlightFirstOption()
  }
};
const highlightFirstOption = () => {
  if (!hasVisibleOptions.value) return

  const first = visibleOptionIds.value[0]
  setCurrentHighlightedOption(getNode(first))
};
const highlightPrevOption = () => {
  if (!hasVisibleOptions.value) return

  const prev = visibleOptionIds.value.indexOf(state.menu.current) - 1
  if (prev === -1) return highlightLastOption()
  setCurrentHighlightedOption(getNode(visibleOptionIds.value[prev]))
};
const highlightNextOption = () => {
  if (!hasVisibleOptions.value) return

  const next = visibleOptionIds.value.indexOf(state.menu.current) + 1
  if (next === visibleOptionIds.value.length) return highlightFirstOption()
  setCurrentHighlightedOption(getNode(visibleOptionIds.value[next]))
};
const highlightLastOption = () => {
  if (!hasVisibleOptions.value) return

  const last = getLast(visibleOptionIds.value)
  setCurrentHighlightedOption(getNode(last))
};
const resetSearchQuery = () => {
  state.trigger.searchQuery = ''
};
const closeMenu = () => {
  if (!state.menu.isOpen || (!props.disabled && props.alwaysOpen)) return
  saveMenuScrollPosition()
  state.menu.isOpen = false
  toggleClickOutsideEvent(false)
  resetSearchQuery()
  emit('close', getValue(), getInstanceId())
};
const openMenu = () => {
  if (props.disabled || state.menu.isOpen) return
  state.menu.isOpen = true
  nextTick(resetHighlightedOptionWhenNecessary)
  nextTick(restoreMenuScrollPosition)
  if (!props.options && !props.async) loadRootOptions()
  toggleClickOutsideEvent(true)
  emit('open', getInstanceId())
};
const toggleMenu = () => {
  if (state.menu.isOpen) {
    closeMenu()
  } else {
    openMenu()
  }
};
const toggleExpanded = (node) => {
  let nextState

  if (state.localSearch.active) {
    nextState = node.isExpandedOnSearch = !node.isExpandedOnSearch
    if (nextState) node.showAllChildrenOnSearch = true
  } else {
    nextState = node.isExpanded = !node.isExpanded
  }

  if (nextState && !node.childrenStates.isLoaded) {
    loadChildrenOptions(node)
  }
};
const buildForestState = () => {
  const selectedNodeMap = createMap()
  state.forest.selectedNodeIds.forEach(selectedNodeId => {
    selectedNodeMap[selectedNodeId] = true
  })
  state.forest.selectedNodeMap = selectedNodeMap

  const checkedStateMap = createMap()
  if (props.multiple) {
    traverseAllNodesByIndex(node => {
      checkedStateMap[node.id] = UNCHECKED
    })

    selectedNodes.value.forEach(selectedNode => {
      checkedStateMap[selectedNode.id] = CHECKED

      if (!props.flat && !props.disableBranchNodes) {
        selectedNode.ancestors.forEach(ancestorNode => {
          if (!isSelected(ancestorNode)) {
            checkedStateMap[ancestorNode.id] = INDETERMINATE
          }
        })
      }
    })
  }
  state.forest.checkedStateMap = checkedStateMap
};
const enhancedNormalizer = (raw) => {
  return {
    ...raw,
    ...props.normalizer(raw, getInstanceId()),
  }
};
const normalize = (parentNode, nodes, prevNodeMap) => {
  let normalizedOptions = nodes
      .map(node => [ enhancedNormalizer(node), node ])
      .map(([ node, raw ], index) => {
        checkDuplication(node)
        verifyNodeShape(node)

        const { id, label, children, isDefaultExpanded } = node
        const isRootNode = parentNode === NO_PARENT_NODE
        const level = isRootNode ? 0 : parentNode.level + 1
        const isBranch = Array.isArray(children) || children === null
        const isLeaf = !isBranch
        const isDisabled = !!node.isDisabled || (!props.flat && !isRootNode && parentNode.isDisabled)
        const isNew = !!node.isNew
        const lowerCased = props.matchKeys.reduce((prev, key) => ({
          ...prev,
          [key]: stringifyOptionPropValue(node[key]).toLocaleLowerCase(),
        }), {})
        const nestedSearchLabel = isRootNode
            ? lowerCased.label
            : parentNode.nestedSearchLabel + ' ' + lowerCased.label

        // $ set(forest.nodeMap, id, createMap())
        state.forest.nodeMap[id] = createMap();

        const normalized = state.forest.nodeMap[id];

        normalized.id = id;
        normalized.label = label;
        normalized.level = level;
        normalized.ancestors = isRootNode ? [] : [ parentNode ].concat(parentNode.ancestors);
        normalized.index = (isRootNode ? [] : parentNode.index).concat(index);
        normalized.parentNode = parentNode;
        normalized.lowerCased = lowerCased;
        normalized.nestedSearchLabel = nestedSearchLabel;
        normalized.isDisabled = isDisabled;
        normalized.isNew = isNew;
        normalized.isMatched = false;
        normalized.isHighlighted = false;
        normalized.isBranch = isBranch;
        normalized.isLeaf = isLeaf;
        normalized.isRootNode = isRootNode;
        normalized.raw = raw;

        if (isBranch) {
          const isLoaded = Array.isArray(children)
          normalized.childrenStates = {...createAsyncOptionsStates(),isLoaded}
          normalized.isExpanded = typeof isDefaultExpanded === 'boolean' ? isDefaultExpanded : level < props.defaultExpandLevel;

          normalized.hasMatchedDescendants = false;
          normalized.hasDisabledDescendants = false;
          normalized.isExpandedOnSearch = false;
          normalized.showAllChildrenOnSearch = false;
          normalized.count = {
            [ALL_CHILDREN]: 0,
            [ALL_DESCENDANTS]: 0,
            [LEAF_CHILDREN]: 0,
            [LEAF_DESCENDANTS]: 0,
          }

          normalized.children = isLoaded ? normalize(normalized, children, prevNodeMap) :[];

          if (isDefaultExpanded === true) normalized.ancestors.forEach(ancestor => {
            ancestor.isExpanded = true
          })

          if (!isLoaded && typeof props.loadOptions !== 'function') {
            warning(
                () => false,
                () => 'Unloaded branch node detected. "loadOptions" prop is required to load its children.',
            )
          } else if (!isLoaded && normalized.isExpanded) {
            loadChildrenOptions(normalized)
          }
        }

        normalized.ancestors.forEach(ancestor => ancestor.count[ALL_DESCENDANTS]++)
        if (isLeaf) normalized.ancestors.forEach(ancestor => ancestor.count[LEAF_DESCENDANTS]++)
        if (!isRootNode) {
          parentNode.count[ALL_CHILDREN] += 1
          if (isLeaf) parentNode.count[LEAF_CHILDREN] += 1
          if (isDisabled) parentNode.hasDisabledDescendants = true
        }

        // Preserve previous states.
        if (prevNodeMap && prevNodeMap[id]) {
          const prev = prevNodeMap[id]

          normalized.isMatched = prev.isMatched
          normalized.showAllChildrenOnSearch = prev.showAllChildrenOnSearch
          normalized.isHighlighted = prev.isHighlighted

          if (prev.isBranch && normalized.isBranch) {
            normalized.isExpanded = prev.isExpanded
            normalized.isExpandedOnSearch = prev.isExpandedOnSearch
            if (prev.childrenStates.isLoaded && !normalized.childrenStates.isLoaded) {
              normalized.isExpanded = false
            } else {
              normalized.childrenStates = { ...prev.childrenStates }
            }
          }
        }

        return normalized
      })

  if (props.branchNodesFirst) {
    const branchNodes = normalizedOptions.filter(option => option.isBranch)
    const leafNodes = normalizedOptions.filter(option => option.isLeaf)
    normalizedOptions = branchNodes.concat(leafNodes)
  }

  return normalizedOptions
};
const loadRootOptions = () => {
  callLoadOptionsProp({
    action: LOAD_ROOT_OPTIONS,
    isPending: () => {
      return state.rootOptionsStates.isLoading
    },
    start: () => {
      state.rootOptionsStates.isLoading = true
      state.rootOptionsStates.loadingError = ''
    },
    succeed: () => {
      state.rootOptionsStates.isLoaded = true
      // Wait for `options` being re-initialized.
      nextTick(() => {
        resetHighlightedOptionWhenNecessary(true)
      })
    },
    fail: err => {
      state.rootOptionsStates.loadingError = getErrorMessage(err)
    },
    end: () => {
      state.rootOptionsStates.isLoading = false
    },
  })
};
const loadChildrenOptions = (parentNode) => {
  // The options may be re-initialized anytime during the loading process.
  // So `parentNode` can be stale and we use `getNode()` to avoid that.

  const { id, raw } = parentNode

  callLoadOptionsProp({
    action: LOAD_CHILDREN_OPTIONS,
    args: {
      // We always pass the raw node instead of the normalized node to any
      // callback provided by the user of this component.
      // Because the shape of the raw node is more likely to be closing to
      // what the back-end API service of the application needs.
      parentNode: raw,
    },
    isPending: () => {
      return getNode(id).childrenStates.isLoading
    },
    start: () => {
      getNode(id).childrenStates.isLoading = true
      getNode(id).childrenStates.loadingError = ''
    },
    succeed: (result) => {
      const node = getNode(id);
      node.children = normalize(parentNode, result);
      node.childrenStates.isLoaded = true
    },
    fail: err => {
      getNode(id).childrenStates.loadingError = getErrorMessage(err)
    },
    end: () => {
      getNode(id).childrenStates.isLoading = false
    },
  })
};
const callLoadOptionsProp = ({ action, args, isPending, start, succeed, fail, end }) => {
  if (!props.loadOptions || isPending()) {
    return
  }

  start()

  const callback = once((err, result) => {
    if (err) {
      fail(err)
    } else {
      succeed(result)
    }

    end()
  })
  const result = props.loadOptions({
    id: getInstanceId(),
    instanceId: getInstanceId(),
    action,
    ...args,
    callback,
  })

  if (isPromise(result)) {
    result.then(() => {
      callback()
    }, err => {
      callback(err)
    }).catch(err => {
      // istanbul ignore next
      console.error(err)
    })
  }
};
const checkDuplication = (node) => {
  warning(
      () => !((node.id in state.forest.nodeMap) && !state.forest.nodeMap[node.id].isFallbackNode),
      () => `Detected duplicate presence of node id ${JSON.stringify(node.id)}. ` +
          `Their labels are "${state.forest.nodeMap[node.id].label}" and "${node.label}" respectively.`,
  )
};
const verifyNodeShape = (node) => {
  warning(
      () => !(node.children === undefined && node.isBranch === true),
      () => 'Are you meant to declare an unloaded branch node? ' +
          '`isBranch: true` is no longer supported, please use `children: null` instead.',
  )
};
const select = (node) => {
  if (props.disabled || node.isDisabled) {
    return
  }

  if (single.value) {
    clear()
  }

  const nextState = props.multiple && !props.flat
      ? state.forest.checkedStateMap[node.id] === UNCHECKED
      : !isSelected(node)

  if (nextState) {
    _selectNode(node)
  } else {
    _deselectNode(node)
  }

  buildForestState()

  if (nextState) {
    emit('select', node.raw, getInstanceId())
  } else {
    emit('deselect', node.raw, getInstanceId())
  }

  if (state.localSearch.active && nextState && (single.value || props.clearOnSelect)) {
    resetSearchQuery()
  }

  if (single.value && props.closeOnSelect) {
    closeMenu()
    // istanbul ignore else
    if (props.searchable) {
      state._blurOnSelect = true
    }
  }

  emit('update:modelValue', props.multiple ? state.forest.selectedNodeIds : state.forest.selectedNodeIds[0] || null);

};
const clear = () => {
  if (hasValue.value) {
    if (single.value || props.allowClearingDisabled) {
      state.forest.selectedNodeIds = []
    } else /* if (multiple && !allowClearingDisabled) */ {
      state.forest.selectedNodeIds = state.forest.selectedNodeIds.filter(nodeId =>
          getNode(nodeId).isDisabled,
      )
    }
    buildForestState()
    emit('update:modelValue', null);
  }
};
const _selectNode = (node) => {
  if (single.value || props.disableBranchNodes) {
    return addValue(node)
  }

  if (props.flat) {
    addValue(node)

    if (props.autoSelectAncestors) {
      node.ancestors.forEach(ancestor => {
        if (!isSelected(ancestor) && !ancestor.isDisabled) addValue(ancestor)
      })
    } else if (props.autoSelectDescendants) {
      traverseDescendantsBFS(node, descendant => {
        if (!isSelected(descendant) && !descendant.isDisabled) addValue(descendant)
      })
    }

    return
  }

  const isFullyChecked = (
      node.isLeaf ||
      (/* node.isBranch && */!node.hasDisabledDescendants) ||
      (/* node.isBranch && */props.allowSelectingDisabledDescendants)
  )
  if (isFullyChecked) {
    addValue(node)
  }

  if (node.isBranch) {
    traverseDescendantsBFS(node, descendant => {
      if (!descendant.isDisabled || props.allowSelectingDisabledDescendants) {
        addValue(descendant)
      }
    })
  }

  if (isFullyChecked) {
    let curr = node
    while ((curr = curr.parentNode) !== NO_PARENT_NODE) {
      if (curr.children.every(isSelected)) addValue(curr)
      else break
    }
  }
};
const _deselectNode = (node) => {
  if (props.disableBranchNodes) {
    return removeValue(node)
  }

  if (props.flat) {
    removeValue(node)

    if (props.autoDeselectAncestors) {
      node.ancestors.forEach(ancestor => {
        if (isSelected(ancestor) && !ancestor.isDisabled) removeValue(ancestor)
      })
    } else if (props.autoDeselectDescendants) {
      traverseDescendantsBFS(node, descendant => {
        if (isSelected(descendant) && !descendant.isDisabled) removeValue(descendant)
      })
    }

    return
  }

  let hasUncheckedSomeDescendants = false
  if (node.isBranch) {
    traverseDescendantsDFS(node, descendant => {
      if (!descendant.isDisabled || props.allowSelectingDisabledDescendants) {
        removeValue(descendant)
        hasUncheckedSomeDescendants = true
      }
    })
  }

  if (
      node.isLeaf ||
      /* node.isBranch && */hasUncheckedSomeDescendants ||
      /* node.isBranch && */node.children.length === 0
  ) {
    removeValue(node)

    let curr = node
    while ((curr = curr.parentNode) !== NO_PARENT_NODE) {
      if (isSelected(curr)) removeValue(curr)
      else break
    }
  }
};
const addValue = (node) => {
  state.forest.selectedNodeIds.push(node.id)
  state.forest.selectedNodeMap[node.id] = true
};
const removeValue = (node) => {
  removeFromArray(state.forest.selectedNodeIds, node.id)
  delete state.forest.selectedNodeMap[node.id]
};
const removeLastValue = () => {
  if (!hasValue.value) return
  if (single.value) return clear()
  const lastValue = getLast(internalValue.value)
  const lastSelectedNode = getNode(lastValue)
  select(lastSelectedNode) // deselect
};
const saveMenuScrollPosition = () => {
  const $menu = getMenu();
  // istanbul ignore else
  if ($menu) rmenu.value.lastScrollPosition = $menu.scrollTop
};
const restoreMenuScrollPosition = () => {
  const $menu = getMenu()
  // istanbul ignore else
  if ($menu) $menu.scrollTop = rmenu.value.lastScrollPosition
};
const handleRemoteSearch = () => {
  const { searchQuery } = state.trigger
  const entry = getRemoteSearchEntry()
  const done = () => {
    initialize(entry)
    resetHighlightedOptionWhenNecessary(true)
  }

  if ((searchQuery === '' || props.cacheOptions) && entry.isLoaded) {
    return done()
  }

  callLoadOptionsProp({
    action: ASYNC_SEARCH,
    args: { searchQuery },
    isPending() {
      return entry.isLoading
    },
    start: () => {
      entry.isLoading = true
      entry.isLoaded = false
      entry.loadingError = ''
    },
    succeed: options => {
      entry.isLoading = false
      entry.isLoaded = true
      entry.options = options
      // When the request completes, the search query may have changed.
      // We only show these options if they are for the current search query.
      if (state.trigger.searchQuery === searchQuery) done()
    },
    fail: err => {
      entry.isLoading = false
      entry.loadingError = getErrorMessage(err)
    },
    end: () => {
      entry.isLoading = false
    },
  })
};

defineExpose({clear})

/** Methods END*/

/** Watch */

watch(CalwaysOpen, (newValue) => {
  if (newValue) openMenu()
  else closeMenu()
});
watch(CbranchNodesFirst, () => {
  initialize()
});
watch(Cdisabled, (newValue) => {
  // force close the menu after disabling the control
  if (newValue && state.menu.isOpen) {
    closeMenu()
  }
  else if (!newValue && !state.menu.isOpen && props.alwaysOpen) openMenu()
});
watch(Cflat, () => {
  initialize()
});

watch(CmatchKeys, () => {
  initialize()
});
watch(Cmultiple, (newValue) => {
  // We need to rebuild the state when switching from single-select mode
  // to multi-select mode.
  // istanbul ignore else
  if (newValue) buildForestState()
});
watch(Coptions, () => {
  if (props.async) return
  // Re-initialize options when the `options` prop has changed.
  initialize()
  state.rootOptionsStates.isLoaded = Array.isArray(props.options)
}, {
  deep: true,
  immediate: true,
});
watch(computedSearchQuery, () => {
  if (props.async) {
    handleRemoteSearch()
  } else {
    handleLocalSearch()
  }

  emit('search-change', state.trigger.searchQuery, getInstanceId())
});
watch(computedModelValue, () => {
  const nodeIdsFromValue = extractCheckedNodeIdsFromValue()
  const hasChanged = quickDiff(nodeIdsFromValue, internalValue.value)
  if (hasChanged) fixSelectedNodeIds(nodeIdsFromValue)
});

/** Watch END */

onMounted(() => {
  verifyProps()
  resetFlags()
  state.isReady = true;
  if (props.autoFocus) focusInput()
  if (!props.options && !props.async && props.autoLoadRootOptions) loadRootOptions()
  if (props.alwaysOpen) openMenu()
  if (props.async && props.defaultOptions) handleRemoteSearch()
  buildForestState();
})
provide('instance', instance.value);
</script>
