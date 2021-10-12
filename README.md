# firestore-livehooks

This package solves the common design pattern where a React component wants to show a collection from Firestore that live-updates (i.e. that subscribes to Firestore changes and updates the UI).

# Installation
`npm install firestore-livehooks` or `yarn add firestore-livehooks`. The package assumes you're using React or React Native, and also that you're using some flavor of Firestore (whether the Web SDK or perhaps react-native-firebase).

# Usage
Most people will just use the `useLiveQueryResult` hook. Some might use the `reconcileSnapshotChanges` utility function directly.

## useLiveQueryResult
This hook returns an up-to-date array reflecting the Firestore query you specify. The result will reflect ongoing changes you make that might impact the results of the underlying query. It does this by subscribing to changes on your query and reconciling those changes with an in-memory cache of past data.

```js
function MyReactComponent() {
  const latestTime = 8675309;
  const todoList = useLiveQueryResult(
    useCallback( // more on this below
      () => firebase
        .firestore()
        .collection("todos")
        .where("timestamp", ">=", latestTime),
      [latestTime]
    ),
    getTodoID, // Function that extracts a key from the type of item in your collection
  );
  
  return (
    <View>
      { todoList.map(t => renderTodo(t)) }
    </View>
  );
}

function getTodoID(todo: Todo) {
  return todo.id;
}
```

The code above will keep `todoList` up to date with the latest changes, whether you add/remove/update the todos from elsewhere in the code. Note that you should pass callbacks for both parameters of `useLiveQueryResult` that _do not change upon re-renders_ -- otherwise, the hook will keep thinking you've changed its inputs and keep resubscribing to the collection and causing a rendering loop.

So: do not just pass `firestore.firebase().collection("todos")` to `useLiveQueryResult` — if you do, every time MyReactComponent re-renders, `useLiveQueryResult` will cause yet another re-render because it will think you've changed its inputs. Instead, `useCallback` or `useMemo` to keep those changes to a minimum. In the above example, `useLiveQueryResult` will only change its query subscription when `latestTime` changes, which is exactly what you want.

## reconcileSnapshotChanges
This is a utility function that you might find useful to directly use at times, though most people will just need `useLiveQueryResult`. `reconcileSnapshotChanges` takes a snapshot and integrates all its changes into an in-memory `Map` of existing objects. This saves you the trouble of having to deal with `added`, `removed`, and `modified` change types yourself.

```js
  const myTodos = useRef(new Map());
  
  useEffect(() => {
    function onNewBootlegs(snap: QuerySnapshot) {
      // You want myTodos as a ref, not just as a `new Map()` directly, because
      // you don't want the closure here to capture a version of the map that's
      // different from subsequent renders.
      reconcileSnapshotChanges(snap, myTodos.current, getTodoID);
    }
    
    return firebase.firestore().collections("bootlegs").onSnapshot(onNewBootlegs);
  }, []);
  
  return (
    <View>{/* use myTodos.current somehow */}</View>
  );
```
