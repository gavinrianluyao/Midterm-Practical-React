1. Smart Task Counter 
    > Debugging case

        a. I reproduced the bug and found that both `setCount(count + 1)` calls read the same `count` value captured by the event handler. React batches them, so both request the same next value and the count rises by only one. I fixed this with functional updates, which React applies in sequence to the latest queued state: `setCount(current => current + 1)` called twice, or one update such as `setCount(current => Math.min(current + 2, target))`. I kept the clamp so the count cannot exceed the target.
    
2. Live Search and Timer 
    > Debugging cases

        a. I fixed the repeated effect by making the query its trigger and not depending on the state the effect updates. I debounce with `useEffect(() => { const id = setTimeout(() => setDebouncedQuery(query), 350); return () => clearTimeout(id) }, [query])`, so the effect reruns when the query changes rather than when the debounced result changes.

        b. I fixed the stale timer by changing the callback to `setSeconds(current => current + 1)`. The functional update reads the latest state on every tick. Because the callback has no other changing dependencies, I create the interval once with an empty dependency list.

        c. I return `clearTimeout(id)` from the search effect so a new keystroke cancels the prior delay and unmount cancels any pending search. I return `clearInterval(id)` from the timer effect so navigation stops the repeating timer and remounting cannot leave a duplicate interval running.

3. Accessible Form Toolkit 
    > Debugging case

        a. I found that `clicksRef.current += 1` changed the ref but did not schedule a React render, which left the displayed count unchanged. I replaced the displayed value with state: `const [clicks, setClicks] = useState(0)`, updated using `setClicks(current => current + 1)`. I kept refs for DOM access and values that do not need to appear in the rendered output.


4. Application Preferences 
    > Debugging cases

        a. I fixed the default-only consumer by wrapping the dashboard's common ancestor with `<PreferencesProvider>`. Consumers now read the value from the nearest matching provider instead of falling back to the context default.

        b. I moved context creation into one shared module and imported that exact object in both provider and consumer code. I found that separate `createContext()` calls create different identities, so a provider using one object cannot supply a consumer using the other.

        c. I moved the shared theme and font-size state into context so nested consumers can read it directly. This removed the need for intermediate components to accept and forward preference props, reducing prop plumbing while allowing context updates to notify the consumers.

5. Product Explorer Performance Repair
    > Requirement

        I verified that `products.js` generates 500 local product records with an id, name, category, and price, meeting the starter-data requirement without a network request.

        I memoized filtering and sorting with `useMemo` because searching the full product array is more work than a trivial render calculation. I included the source products, query, category, and sort order in its dependencies so each filter change produces current results. I wrapped `ProductList` in `React.memo` and pass it the memoized result, selected id, and a `useCallback` selection handler. I verified that changing the unrelated counter does not commit another product-list render. I log render evidence from `useEffect`, not from the memo calculation.

        I would not memoize a cheap calculation such as formatting one selected product's price, or a value used only once in the same render. In those cases, dependency comparisons and cache retention add complexity without avoiding meaningful work. I use memoization when profiling or a clearly expensive calculation/stable-child boundary justifies it.

        I kept the `useMemo` calculation pure because React may rerun or discard it. I moved analytics and other side effects to `useEffect` (or an event handler), rather than running them inside the memo callback.

    > Debugging cases

        a. I repaired the stale search by adding every value read by the calculation to the `useMemo` dependency list: `[products, query, category, sortOrder]`. Omitting `query` kept the old memoized result when the search text changed.

        b. I fixed the stale selection callback by ensuring changing project or selected-item values it captures are dependencies. In this case, I used a functional update, `setSelectedId(current => current === id ? null : id)`, so the callback no longer captures the selected value and can safely use an empty dependency list.

        c. I moved the analytics call out of `useMemo` and into `useEffect`, keyed to the results or filters being reported. React may run a memo calculation more than once or discard it, so I keep side effects in an effect.

6. Issue Tracker State Engine 
    > Debugging cases

        a. I removed direct state and issue-object mutation. For toggling, I map to a new array and return a copied target issue: `issues.map(issue => issue.id === id ? { ...issue, done: !issue.done } : issue)`. I also create new arrays and a new state object for additions and deletions.

        b. I made the action shape consistent between dispatch and the reducer. Since the reducer reads `action.payload.title`, I dispatch `{ type: 'ADD_ISSUE', payload: { title } }`. For toggle/delete, I dispatch an object payload such as `{ type: 'TOGGLE_ISSUE', payload: { id } }` and read `action.payload.id`.
        
        c. I repaired the reducer's default branch to return the current state: `default: return state`. Returning `undefined` breaks the reducer contract; returning the same object correctly represents an unknown or no-op action.
