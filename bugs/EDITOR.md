When we interact with the test editor, we get this error:
Failed to execute 'comparePoint' on 'Range': The node provided and the Range are not in the same tree.

Uncaught WrongDocumentError: Failed to execute 'comparePoint' on 'Range': The node provided and the Range are not in the same tree.
    at DomTextProjection.positionFromDomPoint ([root-of-the-server]__16e8aod._.js:sourcemap:10952:27)
    at push.constructor.onMouseUp ([root-of-the-server]__16e8aod._.js:sourcemap:11323:38)

---

## Error Type
Runtime WrongDocumentError

## Error Message
Failed to execute 'comparePoint' on 'Range': The node provided and the Range are not in the same tree.


    at DomTextProjection.positionFromDomPoint (../../packages/facet/src/content/lib/highlight/dom-text-projection.ts:88:7)
    at HighlightPointerRouter.onMouseUp (../../packages/facet/src/content/lib/highlight/highlight-interaction-controller.ts:217:19)

## Code Frame
  86 |         sources.push(pendingSpace ?? { node: undefined, offset: 0 });
  87 |       }
> 88 |       pendingSpace = undefined;
     |       ^
  89 |       pendingIsSeparator = false;
  90 |     };
  91 |

Next.js version: 16.3.3 (Turbopack)

---
## Error Type
Console TypeError

## Error Message
Cannot destructure property 'tile' of 'parents.pop(...)' as it is undefined.

Next.js version: 16.3.3 (Turbopack)
