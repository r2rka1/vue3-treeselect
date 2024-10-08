<template>
  <div>
    <div class="heading">Single Value</div>

    <Treeselect v-model="test" :options="options"
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

    <Treeselect v-model="test2" :options="options"
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

    <Treeselect v-model="test3" :options="options"
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

    <Treeselect v-model="test4"
                :async="true"
                :default-options="true"
                :load-options="loadOptions"
    />
  </div>
  <button @click="clear">Limpia</button>
  <button @click="imprime">Imprime</button>
</template>

<script>
import Treeselect from "@/components/Treeselect.vue";
import { toRaw } from "vue";

export default {
  components: { Treeselect },
  data() {
    return {
      test: 'b',
      test2: ['b', 'c'],
      test3: ['b', 'c'],
      test4: null,
      options: [ {
        id: 'a',
        label: 'Ohm',
        symbol: "&#8486;",
        children: [ {
          id: 'aa',
          label: 'Beta',
          symbol: "&#8492;"
        }, {
          id: 'ab',
          label: 'Charlie',
          symbol: "&#8493;"
        } ],
      }, {
        id: 'b',
        label: 'Snowman',
        symbol: '&#9731;',
      }, {
        id: 'c',
        label: 'Phone',
        symbol: '&#9743;'
      } ],
      highLevelOptions: [ {
        id: 'a1',
        label: 'Ohm-lvl1',
        symbol: "&#8486;",
        children: null,
      }],
    }
  },
  methods: {
    handleEvent(event, type) {
      console.log(event, type);
    },
    fakeLoad() {
      return new Promise((res) => {
        setTimeout(() => res(toRaw(this.highLevelOptions)),2000)
      })
    },
    fakeLoadChildren() {
      return new Promise((res) => {
        setTimeout(() => res(toRaw(this.options)),2000)
      })
    },
    async loadOptions({ action, searchQuery, callback }) {
      if (action === 'ASYNC_SEARCH') {
        const options = await this.fakeLoad();
        callback(null, options);
      }
      if (action === 'LOAD_CHILDREN_OPTIONS') {
        const options = await this.fakeLoadChildren();
        callback(null, options);
      }
    },
    imprime(){
      console.log(this.test);
    },
    clear() {
      this.test = null;
    }
  },
  watch: {
    test(val, oVal) {
      console.log(val, oVal);
    },
    test2(val, oVal) {
      console.log(val, oVal);
    }
  }
};
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
