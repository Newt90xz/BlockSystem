# Blocksystem
Blocksystem is a project template designed to facilitate the development of applications using Vue 3, Vite, and TypeScript.

## Requirements
Before running the project, make sure you have installed:
- Node.js (recommended version: 18+)
- npm or yarn
You can download Node.js here:
https://nodejs.org/

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

## Project Setup
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
