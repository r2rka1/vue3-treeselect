# @r2rka/vue3-treeselect
> [!NOTE]
> Forked from public archive https://megafetis.github.io/vue3-treeselect-demo/
> In order to continue developing and keeping up with most recent updates

> [!CAUTION]
> This NPM package is still in progress. Not all the functionality is implemented or tested
> Please see the progress/roadmap below

[![vue](https://badgen.net/badge/vue/3.4.21/green)](https://badgen.net/badge/vue/3.0.5/green)
[![vue](https://badgen.net/badge/node/18+/green)](https://badgen.net/badge/vue/3.0.5/green)
[![vite](https://badgen.net/badge/vite/5.2.8/green)](https://badgen.net/badge/vite/5.2.8/green)
![License](https://badgen.net/github/license/megafetis/vue3-treeselect)

----
[![done](https://badgen.net/badge/Breaking/&#x203C;/red)](https://badgen.net/badge/vue/3.0.5/green)
* property `value` => `modelValue`
* event `input` => `updated:modelValue`
* Now use slots with `<template>`
----
[![done](https://badgen.net/badge/Done/&#x2714;/green)](https://badgen.net/badge/vue/3.0.5/green)
* Replace `render()` with `<template>` &check; 
* Use `Composition API` in main component &check; 
* Remove `mixin` usage &check;
* Rework `v-model` &check;
* Use `useSlots` composable &check;
* Use `defineEmits` &check;
* Rework `SingleValue` &check;
* Rework `MultipleValue` &check;
* Rework nested values &check;
* Migrate to `Vite` &check;
-----
[![done](https://badgen.net/badge/InProgress/&#x2692;/yellow)](https://badgen.net/badge/vue/3.0.5/green)
* _Use `CompositionAPI` across all components_  &#9874;
* _Rework/test `async` functionality_  &#9874;
* _Rework/test `before/after slots`_  &#9874;
* _Rework/test `scoped slots`_  &#9874;
* _Accept custom components for nodes/leafs/input/etc_  &#9874;

----

> A multi-select component with nested options support for Vue 3. Thank to [riophae](https://github.com/riophae/vue-treeselect) and his sources and library for vue 2 taken as basis.


![Vue-Treeselect Screenshot](https://raw.githubusercontent.com/riophae/vue-treeselect/master/screenshot.png)

### Features

- Single & multiple select with nested options support
- Fuzzy matching
- Async searching
- Delayed loading (load data of deep level options only when needed)
- Keyboard support (navigate using <kbd>Arrow Up</kbd> & <kbd>Arrow Down</kbd> keys, select option using <kbd>Enter</kbd> key, etc.)
- Rich options & highly customizable
- Supports a wide range of browsers (see [below](#browser-compatibility))
- RTL support

*Requires Vue 3.0+*

### Getting Started

```bash
npm install --save @r2rka/vue3-treeselect
```
```bash
yarn add @r2rka/vue3-treeselect
```

This example shows how to integrate vue3-treeselect with your [Vue SFCs](https://vuejs.org/v2/guide/single-file-components.html).

```vue
<template>
  <div id="app">
    <TreeSelect v-model="value" :multiple="true" :options="options" />
  </div>
</template>

<script setup>
  import { ref } from 'vue';
  import TreeSelect from '@r2rka/vue3-treeselect'
  import 'vue3-treeselect/dist/style.css'
  
  const value = ref(null);
  const options = [ 
      {
        id: 'a',
        label: 'a',
        children: [{
          id: 'aa',
          label: 'aa',
        }, {
          id: 'ab',
          label: 'ab'
        }]
      },
      {
        id: 'b',
        label: 'b',
      },
      {
        id: 'c',
        label: 'c',
      }];
</script>
```
### Documentation for vue 2 & Live Demo. Be careful with breaking changes above.

[Visit the website](https://vue-treeselect.js.org/)

Note: please use a desktop browser since the website hasn't been optimized for mobile devices.

### Browser Compatibility

- Chrome
- Edge
- Firefox
- Safari

It should function well on IE9, but the style can be slightly broken due to the lack of support of some relatively new CSS features, such as `transition` and `animation`. Nevertheless it should look 90% same as on modern browsers.

### Bugs

You can [open an issue](https://github.com/r2rka1/vue3-treeselect/issues).

### Contributing

1. Fork & clone the repo
2. Install dependencies by `yarn` or `npm install`
3. Check out a new branch
4. `npm run dev` & hack
5. Push your changes & file a pull request

### Credits

This project is inspired by [vue-treeselect](https://github.com/riophae/vue-treeselect).
Special thanks go to their respective authors!

Some icons used in this project:

  - "link" icon made by [Smashicons](https://www.flaticon.com/authors/smashicons) is licensed under [CC 3.0 BY](https://creativecommons.org/licenses/by/3.0/)
  - "spinner" icon from [SpinKit](https://github.com/tobiasahlin/SpinKit) is licensed under the [MIT License](https://github.com/tobiasahlin/SpinKit/blob/master/LICENSE)
  - "caret" icon made by [Dave Gandy](https://www.flaticon.com/authors/dave-gandy) is licensed under [CC 3.0 BY](https://creativecommons.org/licenses/by/3.0/)
  - "delete" icon made by [Freepik](https://www.flaticon.com/authors/freepik) is licensed under [CC 3.0 BY](https://creativecommons.org/licenses/by/3.0/)
  - "checkmark symbol" & "minus symbol" icons made by [Catalin Fertu](https://www.flaticon.com/authors/catalin-fertu) are licensed under [CC 3.0 BY](https://creativecommons.org/licenses/by/3.0/)

### License

Released under the [MIT License](https://github.com/megafetis/vue3-treeselect/blob/master/LICENSE).
