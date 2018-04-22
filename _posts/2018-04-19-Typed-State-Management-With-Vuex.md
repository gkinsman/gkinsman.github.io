---
layout: post
published: draft
title: >
    Typed State Management with Vuex
category: Programming
excerpt:
   In this post I'd like to demonstrate a pattern I've encountered for doing type-safe, low-overhead state management with Vuex.
---



In this post I'd like to demonstrate a pattern I've encountered for doing type-safe, low-overhead state management with Vuex. Vuex is the default, first class state management library for Vue.js. It allows developers to define a state tree that provides getters into the tree, synchronous mutations of the tree, and asynchronous actions that allow multiple mutations. Before looking at those however,

**Let's start with a few goals for state management in general:**

1. Everything should be typed using TypeScript, with `noImplicitAny: true`
2. There should be minimal amounts of boilerplate and 'hooking things up'
3. The store API should be as simple as possible.
4. The store should be modular and namespaced, so that properties on different modules don't conflict.

Vuex alone isn't very type-friendly, as it wasn't built from the ground up with types in mind. However, there has been a number of community efforts to add them. Several libraries now exist to achieve this goal, and most of them take the form of API's that build up a typed store that emit a regular Vuex store for inclusion in the root Vue instance. One such library that we'll take a closer look at is called `vuex-typex`.



*The full source code for this post can be found here: [https://github.com/gkinsman/vue-types-demo](https://github.com/gkinsman/vue-types-demo).*



For these code samples, I'm porting the Vuex shopping cart example over to TypeScript and vuex-typex. The original JS implementation can be found [here.](https://github.com/vuejs/vuex/tree/dev/examples/shopping-cart) I have made only minor model changes for ergonomics.

### Getting Started

Let's install the library:

`yarn add vuex-typex`

> While it isn't the focus of this article, I think it's also worth mentioning [av-ts](https://github.com/HerringtonDarkholme/av-ts)  as I've used it in my sample code - I've found it's  the best library for writing components with TypeScript and Vue. [Here's the author's take on why he wrote it](https://herringtondarkholme.github.io/2016/11/01/how-to-choose-vue-library/).

The next step is to define an interface that represents the root of the state tree. It combines all of the child states together. There will be a 1:1 mapping of child state to module, as each module will operate on a single child state.

*/store/index.ts*

{% highlight typescript %}
import Vue from 'vue';
import Vuex from 'vuex';
import { getStoreBuilder } from 'vuex-typex';
// -- more imports here --

Vue.use(Vuex);

export interface RootState {
    cart: CartState;
    products: ProductsState;
}

const store = getStoreBuilder<RootState>().vuexStore();
export default store;
{% endhighlight %}

Once we have the root state defined, we can define each module. Firstly there's the state interface and it's initial state:

*/store/ProductsModule.ts*

{% highlight typescript %}
export interface ProductsState {
    all: Product[];
}

const initialState: ProductsState = {
    all: [],
};

export const products {
    // our API will be exported here
}
{% endhighlight %}

We can now add the special sauce that is `vuex-typex`. This code uses the library helper `getStoreBuilder` to define a module with the namespace `products` whose root state is `RootState`, initialised with the `initialState` from above. We'll use this builder to define getters, mutations and actions. The role of this builder is to capture the context of the store so that call sites can call store methods without needing to pass in the store context. We'll see this in action soon.

{% highlight typescript %}
const builder = getStoreBuilder<RootState>().module('products', initialState);
{% endhighlight %}

### Getters

Using the builder, we can now define a getter that fetches a property from the state.

*/store/ProductsModule.ts*

{% highlight typescript %}

// This read function uses the name of the function passed in as the name of the store getter.
const allProductsGetter = builder.read(function allProducts(state: ProductsState) { 
    return state.all; 
});
...
export const products {
    get allProducts() { return allProductsGetter(); },
}
{% endhighlight %}

The biggest drawback of this approach is that since builder is capturing the context, it uses the name of the function parameter as the store's getter name: `allProducts`. This drawback only applies to getters as we'll see next.

### Mutations

Mutations are functions, and are captured using the builders `commit` function:

*/store/ProductsModule.ts*

{% highlight typescript %}
function decrementProductInventory(state: ProductsState, id: number) {
    const product = state.all.find(p => p.id === id);
    product!.inventory--;
}
...
export const products {
    ...
    decrementProductInventory: builder.commit(decrementProductInventory),
}
{% endhighlight %}



### Actions

Actions look very similar to functions, except that they may be asynchronous, and use the `dispatch` function of the builder instead of `commit`. When deciding between action or mutation, you should choose a mutation unless you want to commit other mutations or do async work.

*/store/ProductsModule.ts*

{% highlight typescript %}
/* BareActionContext here gives us access to other modules' state if we need it - also only possible with actions */
async function getAllProducts(context: BareActionContext<ProductsState, RootState>) {
    const shopProducts = await getProducts();
    products.setProducts(shopProducts);
}
...
export const products {
    ...
    getAllProducts: builder.dispatch(getAllProducts),
}
{% endhighlight %}



### Usage

Using this store API is straightforward: we can import the exported `products` object directly and access it:

{% highlight typescript %}
...
import { products } from '@/store/ProductsModule';
...

@Component
export default class ProductList extends Vue {
  get products() {
    return products.allProducts;
  }

  @Lifecycle
  async created() {
    await products.getAllProducts();
  }
}
{% endhighlight %}

It works great with the dev tools too, you can see that the cart and product modules are correctly namespaced, and time travel works for each commit to the store:

![](/images/Vuex-chrome.gif)



And that's it!

