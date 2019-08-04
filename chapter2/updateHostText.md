## updateHostText
文字节点就非常简单

tryToClaimNextHydratableInstance是进行hydrate的操作

因为文字节点是遍历过程中的终点，不需要reconcileChildren，接下来会进入complete步骤


```
function updateHostText(current, workInProgress) {
  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  }
  // Nothing to do here. This is terminal. We'll do the completion step
  // immediately after.
  return null;
}
```