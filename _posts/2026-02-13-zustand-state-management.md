Starting a thread here to talk through plans and specific issues for migrating features from orcasound-next to orcasite, and adding Zustand for state management. 
___
1. **Feature flag**

In orcasound-next, I set up a `MasterDataLayout` that bundles global state and external API calls, but wraps only my `/beta` page routes downstream from `_app.tsx`. Should I keep this strategy for PRs to orcasite?
___
2. **Combining server data with external APIs**

Before getting into Zustand, the first thing to resolve is where to call the `useMasterData` hook in some top level context (I have it in `MasterDataLayout`). This runs a set of API calls to the server (human, machine, feeds) and external APIs (sightings, AIS) and combines them into a unified schema matched by time and location. The hook relies on React Query for all the underlying requests, so needs to live in React upstream of Zustand stores. 
___
3. **Zustand stores for global state context** 

There are three contexts that should be migrated to Zustand stores:

***DataContext*** 
- global filter state set by UI 
- resulting array of filtered data 
- array of new objects derived from filtered data 
- summary metrics about filtered data

***LayoutContext***
- UI states that persist across page routes

***NowPlayingContext***
- global audio player controls
- Web Audio API analyser node


Migrating these is actually fairly straightforward, it's mostly like this:
- Install -- `npm install zustand`
- Add stores in `/stores/exampleStore.tsx` 
- Set up store:
```
import { create } from "zustand";`

type ExampleStore = {
  myState: boolean;
  setMyState: (value: boolean) => void;
};

export const useExampleStore = create<ExampleStore>((set) => ({
  myState: true,
  setMyState: (value) => set({ myState: value }),
}));
```

But there are a couple of gotchas I am coming across, as below.
___
3. **Selector-based subscriptions** 

Context can trigger unnecessary global re-renders when a local component changes the state. Zustand behaves the same way unless you specifically target one slice of state.

For example if I have a LayoutContext provider, I typically access global state from any nested component within `<LayoutContext>{children}</LayoutContext>` like this:
```
const { alertOpen, setAlertOpen } = useLayout()
```
But every time I use `setAlertOpen`, I am at risk of triggering re-renders to everything inside the LayoutContext provider. 

With Zustand, the default approach does the same thing:
```
const { alertOpen, setAlertOpen } = useLayoutStore()
```
This subscribes to the entire store, so the component re-renders if there is a change to any unrelated state in that store (e.g. activeMobileTab, drawerContent). 

To avoid this, we need to subscribe only to specific slices from the store like this:
```
const alertOpen = useLayoutStore((state) => state.alertOpen);
const setAlertOpen = useLayoutStore((state) => state.setAlertOpen);
```
This is more verbose, but we can also export a custom hook for each slice like this, to make it more streamlined:
```
export const useAlertOpen = () => useLayoutStore((s) => s.alertOpen);
export const useSetAlertOpen = () => useLayoutStore((s) => s.setAlertOpen);
```
In components:
```
const alertOpen = useAlertOpen();
const setAlertOpen = useSetAlertOpen();
```
___
4. **Custom state setters** 

React has a built-in SetStateAction in the useState hook that can take either a value (e.g. 'true') or a function (e.g. (prev) => !prev). Zustand does not have this -- each state setter needs custom handling for different inputs. 

So if I have components that set global state like this:
```
setAlertOpen(true);
```
The Zustand store needs to define a state setter that looks like this:
```
setAlertOpen: (value) => set({ alertOpen: value })
```
If I want to pass a function, we normally need to define a different state setter.
In component:
```
toggleAlertOpen()
```
In store:
```
toggleAlertOpen: () => set((state) => ({ alertOpen: !state.alertOpen }))
```
Or, make the state setter responsive to either a value or function:
```
setAlertOpen: (input) => 
  set((state) => ({ 
    alertOpen: 
      typeof input === "function" ? input(state.alertOpen) : input,
    }))
```
___
Anyway, that's progress for now -- let me know any thoughts!
