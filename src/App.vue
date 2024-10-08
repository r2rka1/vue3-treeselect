<template>
  <div>
    <div class="heading">Single Value</div>

    <Treeselect v-model="test" :options="options" ref="myTreeSelect"
                @open="e => handleEvent(e, 'open')"
                @close="e => handleEvent(e, 'close')"
                :clear-on-select="false"
                @update:model-value="e => handleEvent(e, 'update')"
                @select="e => handleEvent(e, 'select')"
                @deselect="e => handleEvent(e, 'deselect')"
                @search-change="e => handleEvent(e, 'search-change')"
                :close-on-select="false"

    />

    <div class="heading">Multiple Values</div>

    <Treeselect v-model="test2" :options="options" ref="myTreeSelect2"
                :multiple="true"
                :close-on-select="false"
                @open="e => handleEvent(e, 'open')"
                @close="e => handleEvent(e, 'close')"
                @update:model-value="e => handleEvent(e, 'update')"
                @select="e => handleEvent(e, 'select')"
                @deselect="e => handleEvent(e, 'deselect')"
                @search-change="e => handleEvent(e, 'search-change')"
    />

    <div class="heading">Slots</div>

    <Treeselect v-model="test3" :options="options" ref="myTreeSelect3"
                @open="e => handleEvent(e, 'open')"
                @close="e => handleEvent(e, 'close')"
                :multiple="true"
                :clear-on-select="false"
                @update:model-value="e => handleEvent(e, 'update')"
                @select="e => handleEvent(e, 'select')"
                @deselect="e => handleEvent(e, 'deselect')"
                @search-change="e => handleEvent(e, 'search-change')"
                :close-on-select="false"
    >
      <template #before-list>
        <div v-html="'&#9757;'" />
      </template>
      <template #after-list>
        <div v-html="'&#9996;'" />
      </template>
      <template #option-label="{ node, shouldShowCount, count }">
        <div>{{ node.label}} <span v-html="node.raw.symbol"/></div>
      </template>
      <template #value-label="{ node }">
        <div style="display: flex; flex-flow: row">{{ node.label }} <span v-html="node.raw.symbol"/></div>
      </template>
    </Treeselect>

    <div class="heading">Async</div>

    <Treeselect v-model="test4" ref="myTreeSelect4"
                :async="true"
                :default-options="true"
                :load-options="loadOptions"
    />
  </div>
  <button @click="clear">Limpia</button>
</template>

<script setup>
import { ref, toRaw, watch } from 'vue';
import Treeselect from "@/components/Treeselect.vue";

// Variables reactivas
const test = ref('b');
const test2 = ref(['b', 'c']);
const test3 = ref(['b', 'c']);
const test4 = ref(null);

const options = ref([
  {
    id: 'a',
    label: 'Ohm',
    symbol: "&#8486;",
    children: [
      {
        id: 'aa',
        label: 'Beta',
        symbol: "&#8492;"
      },
      {
        id: 'ab',
        label: 'Charlie',
        symbol: "&#8493;"
      }
    ],
  },
  {
    id: 'b',
    label: 'Snowman',
    symbol: '&#9731;',
  },
  {
    id: 'c',
    label: 'Phone',
    symbol: '&#9743;'
  }
]);

const highLevelOptions = ref([
  {
    id: 'a1',
    label: 'Ohm-lvl1',
    symbol: "&#8486;",
    children: null,
  }
]);

// Referencia al componente
const myTreeSelect = ref(null);
const myTreeSelect2 = ref(null);
const myTreeSelect3 = ref(null);
const myTreeSelect4 = ref(null);

// Métodos
const handleEvent = (event, type) => {
  console.log(event, type);
};

const fakeLoad = () => {
  return new Promise((res) => {
    setTimeout(() => res(toRaw(highLevelOptions.value)), 2000);
  });
};

const fakeLoadChildren = () => {
  return new Promise((res) => {
    setTimeout(() => res(toRaw(options.value)), 2000);
  });
};

const loadOptions = async ({ action, searchQuery, callback }) => {
  if (action === 'ASYNC_SEARCH') {
    const options = await fakeLoad();
    callback(null, options);
  }
  if (action === 'LOAD_CHILDREN_OPTIONS') {
    const options = await fakeLoadChildren();
    callback(null, options);
  }
};

const clear = () => {
  myTreeSelect.value?.clear();
  myTreeSelect2.value?.clear();
  myTreeSelect3.value?.clear();
  myTreeSelect4.value?.clear();
};

// Watchers
watch(test, (newValue, oldValue) => {
  console.log(newValue, oldValue);
});

watch(test2, (newValue, oldValue) => {
  console.log(newValue, oldValue);
});
</script>

<style lang="scss">
html {
  min-height: 100%;
  position: relative;
}

#app {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

.heading {
  width: 100%;
  margin: 20px 0;
  text-align: left;
}
</style>
