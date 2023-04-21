---
layout: post
title: "How To Create Toasts In Remix Project"
date:   2023-04-21
categories: ["Remix", "React", "Redux"]
---

# Problem Statement

Using Toasts in a frontend, developed in [Remix](https://remix.run/),
[Tailwind CSS](https://tailwindcss.com/) and [DaisyUI](https://daisyui.com/)
should be easy and controlled by the fronted.

We want to:
- create toast notifications easily from any React Component,
- backend be unaware of toast/notification (no session, cookies, API, etc.).

We  don't want:
- to have a dependency on third-party toast libraries.


# Solution

To achieve this, we’ll use [Redux Toolkit](https://redux-toolkit.js.org/) to manage the toasts on our frontend.

Most of the existing and popular toast libraries for React use Redux pattern or implementation internally, so I’ll just use Redux Toolkit and don’t try to come up with some more clever solution.


> Detail documentation on how to create a Remix project with Tailwind, DaisyUI, and Redux Toolkit can be found at:
>
> - [https://remix.run/docs/en/1.15.0](https://remix.run/docs/en/1.15.0);
> - [https://remix.run/docs/en/1.15.0/guides/styling#tailwind-css](https://remix.run/docs/en/1.15.0/guides/styling#tailwind-css);
> - [https://tailwindcss.com/docs/guides/remix](https://tailwindcss.com/docs/guides/remix);
> - [https://redux-toolkit.js.org/tutorials/quick-start](https://redux-toolkit.js.org/tutorials/quick-start);

Let's create our app:

```bash
$ npx create-remix@latest retoasts
> Just the basic
> Remix App Server
> TypeScript
> Y
$ cd retoasts
$ npm install -D tailwindcss @fontsource/roboto @heroicons/react daisyui
$ npx tailwindcss init
$ npm install @reduxjs/toolkit react-redux
$ npm install uuid @types/uuid
```

Open `remix.config.js` and add:

```js
module.exports = {
  //...
  future: {
    //...
    unstable_tailwind: true,
  },
  //...
};
```

Create `app/tailwind.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Add paths and daisyUI config to `tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./app/**/*.{js,jsx,ts,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [require("daisyui")],
};
```

Create redux reducer/slice for managing the theme at `app/features/toasts/toastsSlice.ts`:

```ts
import type { PayloadAction } from "@reduxjs/toolkit";
import { createSlice } from "@reduxjs/toolkit";
import { v4 as uuidv4 } from 'uuid';
import type { AppDispatch } from "../store";

const DEFAULT_TOAST_TIMEOUT = 2000

export const ToastLevels = ["info", "success", "warning", "error"]

export type ToastLevel = typeof ToastLevels[0] | typeof ToastLevels[1] | typeof ToastLevels[2] | typeof ToastLevels[3]

export interface Toast {
    id: string,
    message: string,
    level: ToastLevel
}
const initialState: Toast[] = []

export const toastsSlice = createSlice({
    name: "toasts",
    initialState,
    reducers: {
        /**
         * Addes new toast
         */
        pushToast: (state, action: PayloadAction<Toast>) => {
            state.push(action.payload)
        },
        /**
         * Removes existing toast
         */
        removeToast: (state, action: PayloadAction<string>) => {
            const index = state.findIndex(toast => toast.id === action.payload)
            if (index !== -1) state.splice(index, 1)
        },
    }
})

export const { pushToast, removeToast } = toastsSlice.actions
export default toastsSlice.reducer

export const addToast = (
    { message, level }: { message: string, level?: ToastLevel }
) => (
    dispatch: AppDispatch
) => {
        level = level === undefined ? "info" : level

        const newToast = createToast({ message, level })

        dispatch(pushToast(newToast))
        if (level !== "error") {
            setTimeout(() => {
                dispatch(removeToast(newToast.id))
            }, DEFAULT_TOAST_TIMEOUT)
        }
    }

export function createToast(
    { message, level}: { message: string, level: ToastLevel }
): Toast {
    const id = uuidv4()
    return {
        id,
        message,
        level,
    } as Toast
}
```

Create a redux store and add themeSlice to it at `app/features/store.ts`:

```js
import { configureStore } from "@reduxjs/toolkit";
import toastsReducer from "./toasts/toastsSlice";

export const store = configureStore({
    reducer: {
        toasts: toastsReducer
    },
    devTools: process.env.NODE_ENV !== 'production',
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

Create redux hooks in `app/hooks/redux.ts`:

```ts
import type { TypedUseSelectorHook} from "react-redux";
import { useDispatch, useSelector } from "react-redux";
import type { AppDispatch, RootState } from "../features/store"

export const useAppDispatch: () => AppDispatch = useDispatch
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector
```



Create `Toast` component for displaying single toast in `app/features/toasts/Toast.tsx`:

```tsx
import {
    XCircleIcon,
    ExclamationTriangleIcon,
    InformationCircleIcon,
    CheckCircleIcon
} from "@heroicons/react/24/outline"
import type { ToastLevel } from "./toastsSlice";
import { removeToast } from "./toastsSlice"
import React from "react"
import { useAppDispatch } from "~/hooks/redux"

export interface ToastProps {
    id: string,
    message: string,
    level: ToastLevel
}

/**
 * Displays single Toast
 */
export function Toast({ id, message, level }: ToastProps) {
    const dispatch = useAppDispatch()

    const [Icon, alertClass] = getToastIconAndClass(level)

    function handleClose(event: React.MouseEvent) {
        event.preventDefault()
        dispatch(removeToast(id))
    }

    return (
        <div className={`z-[1000] w-auto pointer-events-auto shadow-lg alert ${alertClass}`}>
            <div>
                <Icon className="flex-shrink-0 w-6 h-6 stroke-current" />
                <span>{message}</span>
            </div>
            {(level === "error") && <div className="flex-none">
                <button className="link" onClick={handleClose}>
                    close
                </button>
            </div>
            }
        </div>
    )
}

function getToastIconAndClass(level: ToastLevel) {
    if (level === "info") return [InformationCircleIcon, ""]
    if (level === "success") return [CheckCircleIcon, "alert-success"]
    if (level === "warning") return [ExclamationTriangleIcon, "alert-warning"]
    if (level === "error") return [XCircleIcon, "alert-error"]
    return [InformationCircleIcon, ""]
}
```
Create `Toasts` component for displaying list of Toast in `app/features/toasts/Toasts.tsx`:


```tsx
import { useAppSelector } from "~/hooks/redux"
import { Toast } from "./Toast"

/** Displays taosts stacks at the bottom right corner of the page */
export function Toasts() {
    const toasts = useAppSelector((state) => state.toasts)

    return (
        <div className="absolute inset-0 flex justify-end pointer-events-none">
            <div className="box-border flex flex-col justify-end h-full gap-4 p-8 sm:max-w-1/3">
                <div className="contents">
                    {toasts.map(
                        toast =>
                            <Toast
                                key={toast.id}
                                id={toast.id}
                                level={toast.level}
                                message={toast.message}
                            />
                    )}
                </div>
            </div>
        </div>
    )
}
```

Change `app/root.tsx` to:
- use redux provider by extracting `App` component into `AppHtml` component and surrounding it with the redux provider
- add a loader to load theme settings from cookies on the first render (SSR).

```tsx
import type { LinksFunction } from "@remix-run/node";
import {
    Links,
    LiveReload,
    Meta,
    Outlet,
    Scripts,
    ScrollRestoration
} from "@remix-run/react";

import StyleSheet from "./tailwind.css";
import FontStyles from "@fontsource/roboto/index.css"
import { Provider } from "react-redux";
import { store } from "./features/store";
import { Toasts } from "./features/toasts/Toasts";


export default function App() {
    return (
        <Provider store={store}>
            <AppHtml />
        </Provider>
    );
}

export const links: LinksFunction = () => [
    { rel: "stylesheet", href: StyleSheet },
    { rel: "stylesheet", href: FontStyles },
];

/**
 * Separeted HTML component to be able to access redux store and obtain theme
 */
function AppHtml() {
    return (
        <html lang="en">
            <head>
                <meta charSet="utf-8" />
                <meta name="viewport" content="width=device-width,initial-scale=1" />
                <Meta />
                <Links />
            </head>
            <body className="box-border m-0 text-base, bg-base-200 [font-kerning:normal] min-h-screen">
                <Outlet />
                <Toasts />
                <ScrollRestoration />
                <Scripts />
                <LiveReload />
            </body>
        </html>
    );
}
```

Lets now test toasts by replacing `app/routes/_index.tsx`
where we will add a navigation bar and buttons to display toasts:

```tsx
import type { V2_MetaFunction } from "@remix-run/react";
import { Outlet } from "@remix-run/react";
import { addToast } from "~/features/toasts/toastsSlice";
import { useAppDispatch } from "~/hooks/redux";

export const meta: V2_MetaFunction = () => {
    return [{ title: "Remix Toasts" }];
};

export default function Index() {
    const dispatch = useAppDispatch()
    return (
        <>
            <div className="sticky top-0 z-30 w-full transition-all duration-300 shadow-sm bg-opacity-60 bg-base-100 text-base-content backdrop-blur">
                <nav className="navbar min-h-12 bg-base-100">
                    <div className="flex items-center flex-1 gap-1 lg:gap-2">
                        <div className="inline-flex text-lg font-semibold transition-all duration-200 md:text-2xl">
                            <span className="uppercase text-primary">
                                Application Title
                            </span>
                        </div>
                    </div>
                    <div className="flex-0">
                        {/* Right side of navbar */}
                    </div>
                </nav>
            </div>
            <div className="m-4">
                <button
                    className="btn btn-sm"
                    onClick={() => dispatch(addToast({ message: "Info toast" }))}
                >
                    Add Info Toast
                </button>
                <button
                    className="btn btn-sm btn-success"
                    onClick={() => dispatch(
                        addToast({ message: "Everyting is gooing realy well", level: "success" })
                    )}
                >
                    Add Success Toast
                </button>
                <button
                    className="btn btn-sm btn-warning"
                    onClick={() => dispatch(
                        addToast({ message: "Be carefull what are you doing", level: "warning" })
                    )}
                >
                    Add Warning Toast
                </button>
                <button
                    className="btn btn-sm btn-error"
                    onClick={() => dispatch(
                        addToast({ message: "Ka Buuuummm, you are dead!", level: "error" })
                    )}
                >
                    Add Error Toast
                </button>
            </div>
            <Outlet />
        </>
    )
}
```

To run the application execute the following:

```bash
$ npm run dev
```

### Demo

![Changing theme demo](/assets/retoasts_demo.gif)


The demo code is available at [https://github.com/PredragPeranovic/retoasts](https://github.com/PredragPeranovic/retoasts).

If you have any suggestion or question you can create an Issue on GitHub or contact me on Twitter at [@peranp](https://twitter.com/peranp).
