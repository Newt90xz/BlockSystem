# Blocksystem
Block System is a Vue 3 application built around **`Blocks.vue`** — an interactive, educational grid component that lets users place, drag, group, and split unit blocks. It is designed to support mathematical intuition around grouping and place value (units and tens).
 
`App.vue` serves as a reference implementation showing how to configure and embed `Blocks.vue` in a real application.

## Requirements
Before running the project, make sure you have installed:
- Node.js (recommended version: 18+)
- npm or yarn

You can download Node.js here:
https://nodejs.org/


##  Setup
Navigate to the root directory of the project and run the following command in the terminal:
```sh
npm install
```
This command will automatically install Vue, Vite, TypeScript, and the required @vue/tsconfig configurations for the project.

### Running the Application.
To start the development server, run:
```sh
npm run dev
```
This will launch the application in development mode using Vite.


## Recommended IDE Setup

[VS Code](https://code.visualstudio.com/) + [Vue (Official)](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Recommended Browser Setup

- Chromium-based browsers (Chrome, Edge, Brave, etc.):
  - [Vue.js devtools](https://chromewebstore.google.com/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd)
  - [Turn on Custom Object Formatter in Chrome DevTools](http://bit.ly/object-formatters)
- Firefox:
  - [Vue.js devtools](https://addons.mozilla.org/en-US/firefox/addon/vue-js-devtools/)
  - [Turn on Custom Object Formatter in Firefox DevTools](https://fxdx.dev/firefox-devtools-custom-object-formatters/)

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) to make the TypeScript language service aware of `.vue` types.

## Customize configuration

See [Vite Configuration Reference](https://vite.dev/config/).
