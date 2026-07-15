---
type: code-snippet
language: javascript
topics: [houzz, jukwaa, react, refs, hooks]
created: "2026-07-14"
---

# React Ref Composition

## Purpose

讓 [[Jukwaa]] 等 React codebase 中的 component 同時保留內部 ref，並接收 parent 經 `forwardRef` 傳入的 ref，且不違反 Hooks 不可條件式呼叫的規則。

## Code

```javascript
const SimpleProductCard = React.forwardRef((props, forwardedRef) => {
  const localRef = React.useRef(null);
  const composedRef = useForkRef(forwardedRef, localRef);

  React.useEffect(() => {
    // 使用 localRef.current 執行 component 內部邏輯
  }, []);

  return <div ref={composedRef}>Card</div>;
});
```

## Parameters

- `forwardedRef`：parent 需要存取 DOM node 時傳入。
- `localRef`：component 自己的穩定 ref。
- `useForkRef`：把同一個 node 同步寫入兩個 refs 的 helper；不是 React built-in Hook。

## Caveats

不要寫成 `forwardedRef ? forwardedRef : useRef()`；這會讓 `useRef` 變成條件式 Hook，破壞 Hook call order。

## Related

- [[TanStack Query Server State]]
- [[Intersection Observer]]
- [[Jukwaa]]

## References

- `Row notes/2022-11-11 - Fork Ref.md`
