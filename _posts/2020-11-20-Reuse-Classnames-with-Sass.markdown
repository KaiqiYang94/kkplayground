---
layout: post-kayoung
title: "Reuse Classnames With Sass, Using Vue3 and PrimeVue's Grid System as An Example"
date: 2020-11-20
categories:
  - Vue3
  - PrimeVue
  - Sass
description:
image: https://picsum.photos/2000/1200
image-sm: https://picsum.photos/500/300
---

This blog post will talk about how to reuse classnames using Sass in general, and with classnames provided by third party libraries (PrimeVue as an example).

<br/>

Sometimes we just want to reuse classnames.

Here's an example from <a href="https://primefaces.org/primevue/showcase/#/grid">PrimeVue Grid System</a>.

> In example below, large screens display 4 columns, medium screens display 2 columns in 2 rows and finally on small devices, columns are stacked.

```js
<div class="p-grid">
  <div class="p-col-12 p-md-6 p-lg-3">A</div>
  <div class="p-col-12 p-md-6 p-lg-3">B</div>
  <div class="p-col-12 p-md-6 p-lg-3">C</div>
  <div class="p-col-12 p-md-6 p-lg-3">D</div>
</div>
```

<br/>

Developers hate this. Why do I need to write the exact same thing four times? I have to change four times if I need to adjust something blah blah blah. And then we spend wa----y more time to find out how to write it only once.

You can reuse the classnames `p-col-12 p-md-6 p-lg-3` by two methods.

<ul>
<li>
In the Script section of your .vue file
</li>
<li>
In your .scss file
</li>
</ul>
<br/>

<h3>In Script</h3>
In Vuejs, we could store the classname in a variable and bind it to `:class`.

```js
<template>
  <div class="p-grid">
    <div :class="myClassname">
    A
    </div>
    <div :class="myClassname">
    B
    </div>
    <div :class="myClassname">
    C
    </div>
    <div :class="myClassname">
    D
    </div>
  </div>
</template>
<script lang="ts">
import { Vue, Options } from 'vue-class-component';
...

@Options({
    components: {
        ...
    }
})
export default class MyView extends Vue {
  private myClassname = 'p-col-12 p-md-6 p-lg-3';
}
</script>
<style lang="scss" scoped>
...
</style>
```

<br/>
<h3>In .scss file</h3>
Many developers are against putting pure styling stuff in the actual program and I totally understand.

"Why not put 'p-col' in your scss file?"

To save your time, I will just put the answer here.

If you have seperated the `<template>` part from your .vue file, it's ok as well.

In your `.html` or `.vue` file `<template>` section, it looks something like this:

```html
<template>
  <div class="p-grid">
    <div class="my-classname">A</div>
    <div class="my-classname">B</div>
    <div class="my-classname">C</div>
    <div class="my-classname">D</div>
  </div>
</template>
```

In your `.scss` file, the code goes:

```scss
@import 'node_modules/primeflex/primeflex.scss';
...

.my-classname {
    @extend .p-col-12, .p-md-6, .p-lg-3;
}

...
```

The first import is to get all the `p-col`s into this `.scss` file. If you are using UI libraries, this step is extremely important. You can import any libraries that function in a similar mechanism.

Then, we can use the `@extend` method to "combine" the selected `p-col`s you want and it will have the same effect but you've only written it once. :)

Happy coding!
