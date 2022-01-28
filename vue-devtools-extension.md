## Objective

The objective of today's learning is to create an [extension for Chrome](https://developer.chrome.com/docs/extensions/mv3/) that hooks into Vue's application to read from the Feature Toggle's store module.
The extension will allow swithwing fearture toggles dynamically, so that it's effects can be manually tested.

## Action Plan

- Follow along the [Hello, World](https://developer.chrome.com/docs/extensions/mv3/getstarted/) tutorial to have a basic functioning extension
- Fake a list of feature toggles and make a simple popup UI that shows them and allows to toggle their value
- Learn how to hook into the Vue's application instance (by reverse-engineering how the [official Devtools](https://github.com/vuejs/devtools) do it)
- Hook into the application's store
- Connect the changes on the feature toggle's values to the corresponding Vuex actions

## How Vue Allows DevTools Injection

The first thing that I want to understand is how Vue allows external devtools to hook into it.

After a quick search in Vue's source code I find the answer.
Inside the _packages/runtime-core/src/renderer.ts_ file, the function [`baseCreateRenderer`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts#L321):

```ts
const target = getGlobalThis()
target.__VUE__ = true
if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
  setDevtoolsHook(target.__VUE_DEVTOOLS_GLOBAL_HOOK__, target)
}
```

So, if we're either in development mode (`__DEV__`) or the `__FEATURE_PROD_DEVTOOLS__` flag is present, we register whaver is stored inside the `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable.
Knowing that `target == window`, Vue basically expects the `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable to be set and contain the devtools hook.

On _packages/runtime-core/src/devtools.ts_ the [`setDevtoolsHook`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/devtools.ts#L46) does the work of connecting the hook with the internals of the application.
This is how the hook type is defined:

```ts
interface DevtoolsHook {
  enabled?: boolean
  emit: (event: string, ...payload: any[]) => void
  on: (event: string, handler: Function) => void
  once: (event: string, handler: Function) => void
  off: (event: string, handler: Function) => void
  appRecords: AppRecord[]
}
```

We can analyze what this interface is used for later.

## How Vue Devtools Injects Itself

Having understood how Vue allows an external tool to hook into its internals for debugging purposes, I'll work on understanding how the official devtools actually set the `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable with an implementation of the `DevtoolsHook` interface. 

On _packages/app-backend-core/src/hook.ts_, the function [`installHook`](https://github.com/vuejs/devtools/blob/main/packages/app-backend-core/src/hook.ts#L11) does the trick.
But, there is a caveat:

```ts
/**
 * Install the hook on window, which is an event emitter.
 * Note because Chrome content scripts cannot directly modify the window object,
 * we are evaling this function by inserting a script tag. That's why we have
 * to inline the whole event emitter implementation here.
 *
 * @param {Window|global} target
 */
export function installHook (target, isIframe = false) { ... }
```

> Chrome content scripts cannot directly modify the window object

See [the details here](https://github.com/vuejs/devtools/blob/main/packages/app-backend-core/src/hook.ts).

This function adds a `<script>` inside an `iframe` to the document, where it installs the funtion:

```ts
if ((iframe as any).__vdevtools__injected) return
try {
  ;(iframe as any).__vdevtools__injected = true
  const inject = () => {
    try {
      ;(iframe.contentWindow as any).__VUE_DEVTOOLS_IFRAME__ = iframe
      const script = iframe.contentDocument.createElement('script')
      script.textContent = ';(' + installHook.toString() + ')(window, true)'
      iframe.contentDocument.documentElement.appendChild(script)
      script.parentNode.removeChild(script)
    } catch (e) {
      // Ignore
    }
  }
  inject()
  iframe.addEventListener('load', () => inject())
} catch (e) {
  // Ignore
}
```

Note this crazyness (a function serializing itself inside window ???):

```ts
script.textContent = ';(' + installHook.toString() + ')(window, true)'
```

Then, the hook is injected:

```ts
Object.defineProperty(target, '__VUE_DEVTOOLS_GLOBAL_HOOK__', {
  get() {
    return hook
  },
})
```

Since Chrome doesn't allow its extensions to access global variables in the `window` object (for security reasons), the devtools create an `<iframe>` with a `<script>` inside.
This script will include the entire `installHook` function which will grab the `iframe`'s parent `window` (the top `window`), and somehow (still not 100% clear how), sets the `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable there.

## Conclusion

So, the day is over and I haven't been able to hook my extension to Vue.
I haven't ben able to completely decipher the hack that sets the gobal `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable; copy-pasting was out of the question.

I also realized that there can only be one object hooked into `__VUE_DEVTOOLS_GLOBAL_HOOK__` anyway, so by hooking my extension into it, I would break Vue's devtools.
So my plan was to wrap that object inside a `Proxy` to forward the calls to Vue's devtools, but having access to Vue's app internals in the extension as well.
