+++
title = "API Design for functions that return Promises"
date = 2025-02-26T23:08:34+08:00
draft = false
description = "Improving the design of a TypeScript function that returns a Promise"
slug = "api-design-for-functions-that-return-promises"
tags = ["typescript", "promises", "api"]
+++

## Backstory

We use [Sentry](https://sentry.io/) to monitor our application for crashes and thrown exceptions. There was one error that stood out: `UnhandledRejection: Non-Error promise rejection captured with value: undefined`.

We would get dozens of the error per day, yet we didn't receive complaints about the application breaking in a similar frequency from our users at all. Regardless, it was still important to fix as the rate of this error appearing ate up our Sentry limits quickly, which might cause us to miss a more important error.

After investigating the issue, I found out that the errors were actually created by us, i.e. it was written directly in the code! As suggested by the title of the article and the error itself, we were rejecting promises in our code without catching the thrown error.

## How it works

The source of the unhandled rejections was our confirmation dialog. It makes use of promises to determine which action was taken by the user.

If the user presses the confirm button, the dialog will resolve the Promise. If any other action is taken, the dialog will reject the Promise.

With this, we can handle logic based on whether the user pressed confirm or otherwise by passing callback functions that get called when the Promise is resolved or rejected respectively.

## Previous implementation

For context, we use React. To show a confirmation dialog anywhere in the application, we call the `useConfirm` hook which returns a function called `confirm`. When `confirm` is called with the necessary arguments, a confirmation dialog is shown.

```js
export const useConfirm = () => {
  const { setDialogs } = useContext(ConfirmationDialogCtx);
  /**
   * @param {Omit<Dialog, 'id' | 'onConfirm'>} newDialog - dialog props
   * @returns {Promise<void>}
   */
  const confirm = (newDialog: Omit<Dialog, 'id' | 'onConfirm'>) =>
    new Promise<void>((resolve, reject) => {
      setDialogs((pre) => [
        ...(pre || []),
        {
          ...newDialog,
          id: uuidv4(),
          onConfirm: resolve,
          onCancel: reject,
        },
        ]);
    });
  };
  return confirm;
};
```

In the previous implementation, the `confirm` function simply returned the created Promise directly to the caller, i.e. the function definition was `() => Promise<void>`. That meant that the handling of the Promise was left to the caller.

The issue I discovered of leaving the handling of the Promise to the caller was that we almost always didn't handle the case where the Promise would be rejected! Since we reject the Promise whenever the user chooses to cancel, that led to many `unhandledrejection` events being sent to Sentry.

I think there are three reasons why we don't handle the Promise rejection at the call site.

1. We often only needed to provide logic for the happy path, i.e. when the user presses confirm. We seldom needed to think about handling the path where the user cancels, so the Promise rejection was not handled.
2. Unless we make it a point to use `try/catch` everywhere (an error will be shown if there's no `catch` block), there is no error shown if we do not handle the rejected Promise. Even then, it is difficult to enforce that we always use `try/catch` since other forms of Promise handling is also valid JS/TS.
3. Having to handle Promise rejections with empty catch blocks everywhere we call the confirm function just doesn't look like good code anyway.

```js
// With thenables, which we mostly use.
confirm().then(/* logic when user presses confirm */).catch(); // Empty catch to prevent "unhandledrejection" errors

// With async/await and try/catch blocks instead
try {
  await confirm();
} catch {} // Also raises "no-empty" error with recommended ESLint rules
```

## New implementation

The new function definition is just `() => void`. That means we no longer return any Promise to the caller; all the resolution and rejection logic is abstracted away from the caller and handled in the `confirm` function itself.

### Diff

```diff
 export const useConfirm = () => {
   const { setDialogs } = useContext(ConfirmationDialogCtx);

-  /**
-   * @param {Omit<Dialog, 'id' | 'onConfirm'>} newDialog - dialog props
-   * @returns {Promise<void>}
-   */
-  const confirm = (newDialog: Omit<Dialog, 'id' | 'onConfirm'>) =>
+  /**
+   * @param {Omit<Dialog, 'id' | 'onConfirm'>} newDialog - dialog props
+   * @param {() => void} onResolve - callback to call when user clicks confirm
+   * @param {() => void} onReject - callback to call when user clicks cancel
+   * @returns {void}
+   */
+  const confirm = (
+    newDialog: Omit<Dialog, 'id' | 'onConfirm'>,
+    onResolve: () => void,
+    onReject?: () => void,
+  ) => {
     new Promise<void>((resolve, reject) => {
       setDialogs((pre) => [
         ...(pre || []),
         {
           ...newDialog,
           id: uuidv4(),
           onConfirm: resolve,
           onCancel: reject,
         },
       ]);
-    });
+    })
+      .then(onResolve)
+      .catch(() => {
+        if (onReject) onReject();
+      });
+  };

   return confirm;
 };
```

In the new implementation, the caller has to pass the callback function(s) `onResolve` and `onReject` to be called when the user presses confirm or cancel respectively when calling the `confirm` function.

`onResolve` is a required argument, while `onReject` is optional. This matches the previous implementation where many callers didn't even handle the rejection, i.e. there was no code to run when the user pressed cancel, which makes sense because most of the time we only need something to happen when the user presses confirm.

This implementation ensures we never see an `unhandledRejection` error from this piece of code again, because the rejection of the Promise is always handled for the caller. The caller only has to concern itself with what needs to happen on confirm and cancel by passing the respective callback functions as necessary.

## Conclusion

From this experience, I'll always think again before returning a Promise from a function. Does the caller really need the Promise to be returned to them? Or can the function abstract away the fact that it's using Promises from the caller, i.e. receive callback functions from the caller to handle the various Promise states?

I'll likely favour the latter approach as the default, switching to the former only when there is a clear need for returning a Promise to the calling function.
