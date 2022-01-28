## How Vue Allows DevTools Injection

Inside the _packages/runtime-core/src/renderer.ts_ file, the function `baseCreateRenderer`:

```ts
const target = getGlobalThis()
target.__VUE__ = true
if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
  setDevtoolsHook(target.__VUE_DEVTOOLS_GLOBAL_HOOK__, target)
}
```

Knowing that `target == window`, Vue basically expects the `__VUE_DEVTOOLS_GLOBAL_HOOK__` variable to be set and contain the devtools hook.

On _packages/runtime-core/src/devtools.ts_ the `setDevtoolsHook` does the work of connecting the hook with the internals of the application.
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

## How Vue Devtools Injects Itself

On _packages/app-backend-core/src/hook.ts_, the function `installHook` does the trick.
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
