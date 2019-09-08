<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-09-05 14:28:26
 * @LastEditTime: 2019-09-08 16:22:52
 * @LastEditors: Please set LastEditors
 -->
## completeUnitOfWork

每个FiberNode都维护着一个effectList链表，一个fiber的effect list只包括他children的更新，不包括他本身，保存着reconciliation阶段的结果，每个effectList包括nextEffect、firstEffect、lastEffect三个指针，分别指向下一个待处理的effect fiber，第一个和最后一个待处理的effect fiber。react调用completeUnitOfWork沿workInProgress进行effect list的收集

1. 首先会在当前节点完成工作
2. 如果有sibling,移动到next sibling进行工作
3. 没有sibling,返回到parent fiber



```
function completeUnitOfWork(workInProgress) {
  while (true) {
    // 记录当前workInProgress的状态
    var current$$1 = workInProgress.alternate;
    {
      setCurrentFiber(workInProgress);
    }
    // 父级和兄弟fiber
    var returnFiber = workInProgress.return;
    var siblingFiber = workInProgress.sibling;

    if ((workInProgress.effectTag & Incomplete) === NoEffect) {
      // 如果当前fiber work已经完成
      // 记录我们将要完成的任务，以便当后面操作失败时找到该边界
      nextUnitOfWork = workInProgress;
      
      
      // completeWork 根据workInProgress.tag创建、更新组件的属性等
      // nextUnitOfWork 用来判断是否有下一个工作
      nextUnitOfWork = completeWork(current$$1, workInProgress, nextRenderExpirationTime);
     
      // 暂停work
      stopWorkTimer(workInProgress);
      //重置childExpirationTime
      resetChildExpirationTime(workInProgress, nextRenderExpirationTime);
      {
        resetCurrentFiber();
      }

      if (nextUnitOfWork !== null) {
        // 完成当前fiber时，产生了新当work（completeWork返回值）
        return nextUnitOfWork;
      }
        //如果有节点在complete的时候失败了就不进行effect收集
      if (returnFiber !== null && (returnFiber.effectTag & Incomplete) === NoEffect) {
        // 将子树当所有effects和该fiber加入父节点的effect list，子节点的完成顺序决定了effect的顺序
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = workInProgress.firstEffect;
        }
        if (workInProgress.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
          }
          returnFiber.lastEffect = workInProgress.lastEffect;
        }

        // 如果该节点有effects，则将其加在子节点effects之后，
        var effectTag = workInProgress.effectTag;
        // 创建effect list时跳过 NoWork and PerformedWork节点.
        
        if (effectTag > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress;
          } else {
            returnFiber.firstEffect = workInProgress;
          }
          returnFiber.lastEffect = workInProgress;
        }
      }
      

      if (siblingFiber !== null) {
        // 如果其兄弟节点不为空，则继续遍历siblingFiber
        return siblingFiber;
      } else if (returnFiber !== null) {
        // 如果没有其他任务要做.
        workInProgress = returnFiber;
        continue;
      } else {
        // 否则认为我们遍历到了root节点
        return null;
      }
    } else {
      // 当前fiber work未完成
      // 见 throwException 小节
    }
  }

  return null;
}
```

### completeWork

- 初始化时会创建dom
- 更新阶段diffProperties
  

```
function completeWork(current, workInProgress, renderExpirationTime) {
  var newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
      break;
    case LazyComponent:
      break;
    case SimpleMemoComponent:
    case FunctionComponent:
      break;
    case ClassComponent:
      {
        var Component = workInProgress.type;
        if (isContextProvider(Component)) {
          popContext(workInProgress);
        }
        break;
      }
    case HostRoot:
      {
        popHostContainer(workInProgress);
        popTopLevelContextObject(workInProgress);
        var fiberRoot = workInProgress.stateNode;
        if (fiberRoot.pendingContext) {
          fiberRoot.context = fiberRoot.pendingContext;
          fiberRoot.pendingContext = null;
        }
        if (current === null || current.child === null) {
          // If we hydrated, pop so that we can delete any remaining children
          // that weren't hydrated.
          popHydrationState(workInProgress);
          // This resets the hacky state to fix isMounted before committing.
          // TODO: Delete this when we delete isMounted and findDOMNode.
          workInProgress.effectTag &= ~Placement;
        }
        updateHostContainer(workInProgress);
        break;
      }
    case HostComponent:
      {
        popHostContext(workInProgress);
        var rootContainerInstance = getRootHostContainer();
        var type = workInProgress.type;
        if (current !== null && workInProgress.stateNode != null) {
          updateHostComponent$1(current, workInProgress, type, newProps, rootContainerInstance);

          if (current.ref !== workInProgress.ref) {
            markRef$1(workInProgress);
          }
        } else {
          if (!newProps) {
            (function () {
              if (!(workInProgress.stateNode !== null)) {
                {
                  throw ReactError('We must have new props for new mounts. This error is likely caused by a bug in React. Please file an issue.');
                }
              }
            })();
            // This can happen when we abort work.
            break;
          }

          var currentHostContext = getHostContext();
          // TODO: Move createInstance to beginWork and keep it on a context
          // "stack" as the parent. Then append children as we go in beginWork
          // or completeWork depending on we want to add then top->down or
          // bottom->up. Top->down is faster in IE11.
          var wasHydrated = popHydrationState(workInProgress);
          if (wasHydrated) {
            // TODO: Move this and createInstance step into the beginPhase
            // to consolidate.
            if (prepareToHydrateHostInstance(workInProgress, rootContainerInstance, currentHostContext)) {
              // If changes to the hydrated node needs to be applied at the
              // commit-phase we mark this as such.
              markUpdate(workInProgress);
            }
          } else {
            var instance = createInstance(type, newProps, rootContainerInstance, currentHostContext, workInProgress);

            appendAllChildren(instance, workInProgress, false, false);

            // Certain renderers require commit-time effects for initial mount.
            // (eg DOM renderer supports auto-focus for certain elements).
            // Make sure such renderers get scheduled for later work.
            if (finalizeInitialChildren(instance, type, newProps, rootContainerInstance, currentHostContext)) {
              markUpdate(workInProgress);
            }
            workInProgress.stateNode = instance;
          }

          if (workInProgress.ref !== null) {
            // If there is a ref on a host node we need to schedule a callback
            markRef$1(workInProgress);
          }
        }
        break;
      }
    case HostText:
      {
        var newText = newProps;
        if (current && workInProgress.stateNode != null) {
          var oldText = current.memoizedProps;
          // If we have an alternate, that means this is an update and we need
          // to schedule a side-effect to do the updates.
          updateHostText$1(current, workInProgress, oldText, newText);
        } else {
          if (typeof newText !== 'string') {
            (function () {
              if (!(workInProgress.stateNode !== null)) {
                {
                  throw ReactError('We must have new props for new mounts. This error is likely caused by a bug in React. Please file an issue.');
                }
              }
            })();
            // This can happen when we abort work.
          }
          var _rootContainerInstance = getRootHostContainer();
          var _currentHostContext = getHostContext();
          var _wasHydrated = popHydrationState(workInProgress);
          if (_wasHydrated) {
            if (prepareToHydrateHostTextInstance(workInProgress)) {
              markUpdate(workInProgress);
            }
          } else {
            workInProgress.stateNode = createTextInstance(newText, _rootContainerInstance, _currentHostContext, workInProgress);
          }
        }
        break;
      }
    case ForwardRef:
      break;
    case SuspenseComponent:
      {
        var nextState = workInProgress.memoizedState;
        if ((workInProgress.effectTag & DidCapture) !== NoEffect) {
          // Something suspended. Re-render with the fallback children.
          workInProgress.expirationTime = renderExpirationTime;
          // Do not reset the effect list.
          return workInProgress;
        }

        var nextDidTimeout = nextState !== null;
        var prevDidTimeout = current !== null && current.memoizedState !== null;

        if (current === null) {
          // In cases where we didn't find a suitable hydration boundary we never
          // downgraded this to a DehydratedSuspenseComponent, but we still need to
          // pop the hydration state since we might be inside the insertion tree.
          popHydrationState(workInProgress);
        } else if (!nextDidTimeout && prevDidTimeout) {
          // We just switched from the fallback to the normal children. Delete
          // the fallback.
          // TODO: Would it be better to store the fallback fragment on
          var currentFallbackChild = current.child.sibling;
          if (currentFallbackChild !== null) {
            // Deletions go at the beginning of the return fiber's effect list
            var first = workInProgress.firstEffect;
            if (first !== null) {
              workInProgress.firstEffect = currentFallbackChild;
              currentFallbackChild.nextEffect = first;
            } else {
              workInProgress.firstEffect = workInProgress.lastEffect = currentFallbackChild;
              currentFallbackChild.nextEffect = null;
            }
            currentFallbackChild.effectTag = Deletion;
          }
        }

        if (supportsPersistence) {
          if (nextDidTimeout) {
            // If this boundary just timed out, schedule an effect to attach a
            // retry listener to the proimse. This flag is also used to hide the
            // primary children.
            workInProgress.effectTag |= Update;
          }
        }
        if (supportsMutation) {
          if (nextDidTimeout || prevDidTimeout) {
            // If this boundary just timed out, schedule an effect to attach a
            // retry listener to the proimse. This flag is also used to hide the
            // primary children. In mutation mode, we also need the flag to
            // *unhide* children that were previously hidden, so check if the
            // is currently timed out, too.
            workInProgress.effectTag |= Update;
          }
        }
        break;
      }
    case Fragment:
      break;
    case Mode:
      break;
    case Profiler:
      break;
    case HostPortal:
      popHostContainer(workInProgress);
      updateHostContainer(workInProgress);
      break;
    case ContextProvider:
      // Pop provider fiber
      popProvider(workInProgress);
      break;
    case ContextConsumer:
      break;
    case MemoComponent:
      break;
    case IncompleteClassComponent:
      {
        // Same as class component case. I put it down here so that the tags are
        // sequential to ensure this switch is compiled to a jump table.
        var _Component = workInProgress.type;
        if (isContextProvider(_Component)) {
          popContext(workInProgress);
        }
        break;
      }
    case DehydratedSuspenseComponent:
      {
        if (enableSuspenseServerRenderer) {
          if (current === null) {
            var _wasHydrated2 = popHydrationState(workInProgress);
            (function () {
              if (!_wasHydrated2) {
                {
                  throw ReactError('A dehydrated suspense component was completed without a hydrated node. This is probably a bug in React.');
                }
              }
            })();
            skipPastDehydratedSuspenseInstance(workInProgress);
          } else if ((workInProgress.effectTag & DidCapture) === NoEffect) {
            // This boundary did not suspend so it's now hydrated.
            // To handle any future suspense cases, we're going to now upgrade it
            // to a Suspense component. We detach it from the existing current fiber.
            current.alternate = null;
            workInProgress.alternate = null;
            workInProgress.tag = SuspenseComponent;
            workInProgress.memoizedState = null;
            workInProgress.stateNode = null;
          }
        }
        break;
      }
    case EventComponent:
      {
        if (enableEventAPI) {
          popHostContext(workInProgress);
          var _rootContainerInstance2 = getRootHostContainer();
          var responder = workInProgress.type.responder;
          // Update the props on the event component state node
          workInProgress.stateNode.props = newProps;
          handleEventComponent(responder, _rootContainerInstance2, workInProgress);
        }
        break;
      }
    case EventTarget:
      {
        if (enableEventAPI) {
          popHostContext(workInProgress);
          var _type = workInProgress.type.type;
          var node = workInProgress.return;
          var parentHostInstance = null;
          // Traverse up the fiber tree till we find a host component fiber
          while (node !== null) {
            if (node.tag === HostComponent) {
              parentHostInstance = node.stateNode;
              break;
            }
            node = node.return;
          }
          
        }
        break;
      }
    default:
      (function () {
        {
          {
            throw ReactError('Unknown unit of work tag. This error is likely caused by a bug in React. Please file an issue.');
          }
        }
      })();
  }

  return null;
}
```