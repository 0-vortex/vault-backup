# Hot ViteJS rewrite

## Recommended templates

Stuff that I found or was recommended to me from the open source community and some initial interesting pointers to look at:
- vite-react-ts-extended [x]
- vite-react-ts-vite-template [x] cypress+husky
- vite-tropical [x] fela+islands
- vite-vitamin [x] suspense+stylelint
- vite-vue-vben-admin [x] vite+plugins
- vite-stravital [x] eslint+typescript
- vite-vital [x] splash+tailwind
- vite-reactts17-chakra-jest-husky [x] chakra

A bigger list of templates and plugins can be found at the "Veet Compatibles" list on GitHub: https://github.com/stars/0-vortex/lists/veet-compatibles

## Vite template

Ideally, it should be:

```shell
npm create vite@latest test -- --template open-sauced-vite-ts-template-test
```

But it's probably going to be:

```shell
npx degit open-sauced/vite-ts-template myApp
```

## Ongoing rewrite

This section is mostly relevant to [open-sauced/hot#140](https://github.com/open-sauced/hot/pull/140), it's not an up-to-date tracker but a scratchlist.

### issues

- font-awesome icons styling on header
- eslint vite cache gets stuck sometimes
- used \<any\> in some vague places
- incomplete postcss configuration/packages
- no onegraph stuff, at all

### TODO

- vite pwa configuration
- enforce type checking
- split components better
