---
layout: post
title: "How To Change Theme In Remix Project"
date:   2023-04-11
categories: ["Remix", "React", "Redux"]
---

# Problem Statement

Users should be able to switch between a light and dark theme in our [Remix](https://remix.run/) app, where we're using
[Tailwind CSS](https://tailwindcss.com/) with [DaisyUI](https://daisyui.com/) components for styling HTML.

We want the theme change to be:
- instant,
- without the need to communicate with backend, and
- we want the selected theme to be remembered after the user leaves the page or refreshes it.

We also want that:
- any React Component inside the app can access or change theme.

# Solution

To achieve this, we'll use [Redux Toolkit](https://redux-toolkit.js.org/) to manage the theme state of our frontend.
We'll also store the current theme in a [cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) so that the server can retrieve it and serve the appropriate theme when the user refreshes or navigates to the page for the first time (basically when page is rendered on the server).

> Detail documentation on how to create a Remix project with Tailwind, DaisyUI, and Redux Toolkit can be found at:
>
> - [https://remix.run/docs/en/1.15.0](https://remix.run/docs/en/1.15.0);
> - [https://remix.run/docs/en/1.15.0/guides/styling#tailwind-css](https://remix.run/docs/en/1.15.0/guides/styling#tailwind-css);
> - [https://tailwindcss.com/docs/guides/remix](https://tailwindcss.com/docs/guides/remix);
> - [https://redux-toolkit.js.org/tutorials/quick-start](https://redux-toolkit.js.org/tutorials/quick-start);

Let's create our app:

```bash
$ npx create-remix@latest retheme
> Just the basic
> Remix App Server
> TypeScript
> Y
$ cd retheme
$ npm install -D tailwindcss @fontsource/roboto @heroicons/react daisyui
$ npx tailwindcss init
$ npm install @reduxjs/toolkit react-redux
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
  daisyui: {
    themes: ["light", "dark"],
    darkTheme: "dark",
  },
};
```

Create redux reducer/slice for managing the theme at `app/features/theme/themeSlice.ts`:

```ts
import type { PayloadAction} from "@reduxjs/toolkit";
import { createSlice } from "@reduxjs/toolkit"

export const COOKIE_NAME = "theme"

export const ThemeStates = ["light", "dark"]

export type ThemeState = typeof ThemeStates[0] | typeof ThemeStates[1]

export const initialState: ThemeState = "light"

export const themeSlice = createSlice({
    name: "theme",
    initialState: initialState as ThemeState,
    reducers: {
        changeTheme: (state) => {
            const newTheme = state === "light" ? "dark" : "light"
            setThemeCookie(newTheme)
            return newTheme
        },
        setTheme: (state, action:PayloadAction<ThemeState>) => {
            const newTheme = action.payload
            if (newTheme !== 'dark' && newTheme !== 'light') {
                return state
            }
            setThemeCookie(newTheme)
            return newTheme
        }
    }
})

export const { changeTheme, setTheme } = themeSlice.actions
export default themeSlice.reducer

/** Saves current theme in plain cookie */
function setThemeCookie(theme:ThemeState){
    if (typeof window === "object") {
        document.cookie = `${COOKIE_NAME}=${theme}; samesite=lax; max-age=${60*60*24*365}`
    }
}
```

Create a redux store and add themeSlice to it at `app/features/store.ts`:

```js
import { configureStore } from "@reduxjs/toolkit";
import themeReducer from "./theme/themeSlice";

export const store = configureStore({
    reducer: {
        theme: themeReducer
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


Create a useTheme hook for getting the current theme in `app/features/theme/useTheme.ts`:

```ts
import { useEffect } from "react"
import { useAppDispatch, useAppSelector } from "~/hooks/redux"
import type { ThemeState} from "./themeSlice";
import { setTheme } from "./themeSlice"
import { useRouteLoaderData } from "@remix-run/react"
import type { CookieSettings } from "../cookieSettings/types";


/** Hook for getting current theme reagardles if it is client side of Server Side rendering */
export function useTheme() {
    const dispatch = useAppDispatch()
    // get theme from redux state
    const theme = useAppSelector((state) => state.theme)

    // get theme from cookie (usefull when server side rendering)
    const cookieSettings = useRouteLoaderData("root") as CookieSettings
    const cookieTheme = cookieSettings.theme

    const effectiveTheme:ThemeState = (typeof window === "undefined") ? cookieTheme : theme

    useEffect(() => {
        // on update theme in redux from cookie on first load
        if (cookieTheme !== theme) {
            dispatch(setTheme(cookieTheme))
        }
    }, [])

    return effectiveTheme
}
```

Create `ThemeSwitcher` button for swithching between themes in `app/features/theme/ThemeSwitcher.tsx`:

```tsx
import { useAppDispatch, } from "~/hooks/redux"
import { SunIcon, MoonIcon } from "@heroicons/react/24/outline"
import { changeTheme, initialState as defaultTheme } from "./themeSlice"
import { useTheme } from "./useTheme"

/**
 * Button for top navigation bar for switching theme between light and dark
 */
export function ThemeSwitcher() {
    const dispatch = useAppDispatch()
    const theme = useTheme()

    return (
        <label className="items-center flex-none swap btn btn-sm btn-ghost btn-circle" title="Change theme">
            <input type="checkbox"
                checked={theme !== defaultTheme}
                onChange={() => dispatch(changeTheme())}
            />
            <SunIcon className="w-5 h-5 stroke-current md:w-6 md:h-6 swap-on" />
            <MoonIcon className="w-5 h-5 stroke-current md:w-6 md:h-6 swap-off" />
        </label>
    )
}
```


Create a helper for getting plain text cookies values on the backend in `app/helpers/cookieSettings.server.ts`.

```ts
import type { ThemeState } from "../features/theme/themeSlice";
import { COOKIE_NAME as THEME_COOKIE_NAME } from "../features/theme/themeSlice";
import { ThemeStates, initialState as defaultTheme } from "../features/theme/themeSlice"

/**
 * Returns plain value from cookie
 * @param request current request
 * @param name name of the cookie value
 * @returns null or value as string
 */
export function getCookieValue(request: Request, name: string): string | null {
    const cookieHeader = request.headers.get("Cookie")
    if (cookieHeader === null) {
        return null
    }

    const cookies = Object.fromEntries(cookieHeader.split("; ").map(v => v.split(/="?([^"]+)"?/)))
    const value = cookies[name]
    if (value === undefined) {
        return null
    }
    return value as string
}

/**
 * Returns theme from cookie or default theme
 */
export function getThemeFromCookie(request: Request): ThemeState {
    const themeInCookie = getCookieValue(request, THEME_COOKIE_NAME)

    if (themeInCookie && ThemeStates.includes(themeInCookie)) {
        return themeInCookie as ThemeState
    }
    return defaultTheme as ThemeState
}
```

Create `app/helpers/cookieSettings.types.ts`:

```ts
import type { ThemeState } from "../features/theme/themeSlice";
/**
 * Settings saved as plain cookie values used for
 * theming, language and similar settings to be
 * saved across visits to the app
 */
export interface CookieSettings {
    theme: ThemeState
}
```

Change `app/root.tsx` to:
- use redux provider by extracting `App` component into `AppHtml` component and surrounding it with the redux provider
- add a loader to load theme settings from cookies on the first render (SSR).

```tsx
import type { LinksFunction, LoaderArgs } from "@remix-run/node";
import type {
    ShouldRevalidateFunction} from "@remix-run/react";
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
import { useTheme } from "./features/theme/useTheme";
import { getThemeFromCookie } from "./helpers/cookieSettings.server";
import type { CookieSettings } from "./helpers/cookieSettings.types";


export default function App() {
    return (
        <Provider store={store}>
            <AppHtml />
        </Provider>
    );
}

/** Returns user settings from cookies, loaded only once */
export async function loader({ request }: LoaderArgs) {
    const theme = getThemeFromCookie(request)

    return {
        theme
    } as CookieSettings
}
export const shouldRevalidate: ShouldRevalidateFunction = () => false

export const links: LinksFunction = () => [
    { rel: "stylesheet", href: StyleSheet },
    { rel: "stylesheet", href: FontStyles },
];

/**
 * Separeted HTML component to be able to access redux store and obtain theme
 */
function AppHtml() {
    const theme = useTheme()

    return (
        <html lang="en" data-theme={theme}>
            <head>
                <meta charSet="utf-8" />
                <meta name="viewport" content="width=device-width,initial-scale=1" />
                <Meta />
                <Links />
            </head>
            <body className="box-border m-0 text-base, bg-base-200 [font-kerning:normal] min-h-screen">
                <Outlet />
                <ScrollRestoration />
                <Scripts />
                <LiveReload />
            </body>
        </html>
    );
}
```

Lets now use the theme change button in our application by simply replacing `app/routes/_index.tsx` where we will add a navigation bar and button to change the theme:

```tsx
import type { V2_MetaFunction } from "@remix-run/react";
import { Outlet } from "@remix-run/react";
import { ThemeSwitcher } from "~/features/theme/ThemeSwitcher";

export const meta: V2_MetaFunction = () => {
    return [{ title: "Remix Theme Change" }];
};

export default function Index() {
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
                        <ThemeSwitcher />
                    </div>
                </nav>
            </div>
            <p className="max-w-sm">
                Lorem, ipsum dolor sit amet consectetur adipisicing elit. Sint, aut modi.
                Animi exercitationem aspernatur hic impedit sint distinctio!
                Aperiam illum eaque perspiciatis distinctio eligendi dignissimos explicabo asperiores facere velit qui.
            </p>
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

![Changing theme demo](/assets/retheme_demo.gif)


The demo code is available at [https://github.com/PredragPeranovic/retheme](https://github.com/PredragPeranovic/retheme).

If you have any suggestion or question you can create an Issue on GitHub or contact me on Twitter at [@peranp](https://twitter.com/peranp).
