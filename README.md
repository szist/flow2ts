# Flow => Typescript

Flow gets less and less library support, while typescript seems like the go-to choice for newer projects.
This documents the migration process I used to convert a React web app.

## Setup

```bash
npm install -g @babel/cli @babel/core
npm install --no-save @zxbodya/babel-plugin-flow-to-typescript

# install parallel
brew install parallel
# or
sudo apt isntall parallel 
```

Actually there are more plugin options. Choose the one that's most updated.
The base package: `babel-plugin-flow-to-typescript`, but more up-to date alternatives atm: 
`@steelbrain/babel-plugin-flow-to-typescript` or `@zxbodya/babel-plugin-flow-to-typescript`

**Warning!!!** Babel configs in the project might mess with the conversion. It's a good idea, to create a secondary config with just the plugin - `.babelrc.flow2ts.json`

```json
{
    "plugins": ["@zxbodya/babel-plugin-flow-to-typescript"]
}
```

## Conversion process

### Check files to be converted

First, review the files that are going to be transformed:

Convert these to tsx:
```bash
grep -iRl --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -l "React"
```

Convert these to ts:
```bash
grep -iRl --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -L "React"
```

where `./app` contains the source code of the application. If you have more folders, just append after `./app`. Also if you have more extension (like jsx, add `--include=\*.jsx` to the grep param)

### Files not converted

Review files that are not going to be converted:

```bash
grep -iRL --include=\*.js "@flow" ./app
```

It might be a good idea to also convert these to typescript, but there might be more fixes necessary after the conversion. With javascript interop these should still work, but it will limit typescript's ability to check things.

### Test conversion

Test the babel conversion process on a single file, to see if it works as expected:

```bash
babel --config-file ./.babelrc.flow2ts.json ./app/test_file.js -o ./app/test_file.tsx
```

### Git workaround

Git won't be able to detect moving files, if they are changed/transpiled in the same commit.
The best approach I found was to:

1. move all files using `git mv` from `.js` to `.ts(x)`
1. commit
1. move all files back that need to be transpiled
1. transpile all files (see below)
1. fix issues/formatting
1. commit

```bash
# move files
grep -iRl --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -l "React" | xargs -n1 sh -c 'git mv "$0" "${0%.js}.tsx"'
grep -iRl --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -L "React" | xargs -n1 sh -c 'git mv "$0" "${0%.js}.ts"'

git commit

# move files back
grep -iRl --include=\*.tsx "@flow" ./app | xargs -n1 sh -c 'mv "$0" "${0%.tsx}.js"'
grep -iRl --include=\*.ts "@flow" ./app | xargs -n1 sh -c 'mv "$0" "${0%.ts}.js"'

#continue below
```


### Convert files

Use the grep lines above to get a list of files to convert =>
```bash
SELECT_SCRIPT | parallel "babel {} -o {.}.tsx --config-file ./.babelrc.flow2ts.json && rm {}"
# or just to ts
SELECT_SCRIPT | parallel "babel {} -o {.}.ts --config-file ./.babelrc.flow2ts.json && rm {}"
```

### [Optional] Move other files

Use the grep commands for the files not converted, or just

```bash
# files using React
grep -iRL --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -l "React" | xargs -n1 sh -c 'git mv "$0" "${0%.js}.tsx"'
# files not using react
grep -iRL --include=\*.js "@flow" ./app | tr '\n' '\0' | xargs -0 grep -L "React" | xargs -n1 sh -c 'git mv "$0" "${0%.js}.ts"'
```

but if you ran the convertion scripts even before this, you can skip the check for flow:

```bash
#files using react
grep -iRl --include=\*.js "React" ./app | xargs -n1 sh -c 'git mv "$0" "${0%.js}.tsx"'
# files not using react
grep -iRL --include=\*.js "React" ./app | xargs -n1 sh -c 'git mv "$0" "${0%.js}.ts"'
```

### Formatting

At this point formatting is probably really bad in each file ☹ 
One way to fix it if there is prettier in place after staging all files.

It's a good idea to configure prettier and eslint before doing this. See below.

```
npm i --no-save pretty-quick
npx pretty-quick --staged
```

## Setup typescript

Install preset
```
npm i --save-dev @babel/preset-typescript
npm i --save typescript ts-node
```

### webpack

```
{
  resolve: {
    extensions: [".ts", ".tsx", ".js", ".json"],
  }
}
```

### babel

```js
{
  presets: [
    "@babel/preset-env",
    "@babel/preset-react",
    ["@babel/preset-typescript", { allExtensions: true, isTSX: true }]
  ]
}
```


### eslint

```bash
npm i --saved-dev @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

Make sure you have up-to-date eslint-config-prettier and eslint-plugin-prettier configs.

```js
{
  parser: "@typescript-eslint/parser",
  "extends": [
    ...
    "plugin:@typescript-eslint/eslint-recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier",
    ...
    "prettier/@typescript-eslint"
  ]
  plugins: [...'@typescript-eslint'],
  parser: '@typescript-eslint/parser',
  settings: {
    'import/resolver': {
      node: {
        extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],
      },
    },
  }
}
```

### prettier

```
{
  "parser": "typescript"
}
```


### global imports

tsconfig.json, assuming you have global imports from app/ folder
```
  "baseUrl": ".",
  "paths": {
    "*": ["node_modules/*", "app/*"]
  }
```

alternatively you could do per directory that are used globally:
```
  "baseUrl": ".",
  "paths": {
    "util/*": ["app/util/*"],
    "components/*": ["app/components/*],
    ...
  }
```

### example typescript configs

The `dom` lib is required to allow `window` and other global types.
In the beginning `strict` might be helpful to migrate things over slowly.

```json
{
    "compilerOptions": {
      "target": "es6",
      "lib": [
        "esnext",
        "dom"
      ],
      "allowJs": true,
      "skipLibCheck": true,
      "esModuleInterop": true,
      "allowSyntheticDefaultImports": true,
      "strict": false,
      "forceConsistentCasingInFileNames": true,
      "module": "commonjs",
      "moduleResolution": "node",
      "resolveJsonModule": true,
      "isolatedModules": true,
      "noEmit": true,
      "jsx": "react",
      "baseUrl": ".",
      "paths": {
        "*": ["node_modules/*", "app/*"]
      }
    },
    "exclude": [
      "node_modules"
    ]
  }
  ```


## Tests

### Typescript checks

```bash
npm i -D node-typescript ts-node typescript
```

add to scripts:

```json
{
  "scripts": { 
    "tsc": "tsc"
  }
}
```
Optionally you can add the typescript compiler tests to your static tests.

### Jest

```bash
npm i -D ts-jest @types/jest
```

Example `jest.config.js`:

```js
module.exports = {
  preset: "ts-jest/presets/js-with-babel", /* transpile  js with babel */
  testEnvironment: "jsdom", // for jest and enzyme, otherwise node
  collectCoverageFrom: [
    "**/app/**/*.{js,ts,tsx}",
  ],
  globals: {
    "ts-jest": {
      diagnostics: false /* TODO disable once typescript gets fixed */
    }
  },
  moduleDirectories: ["node_modules", "app"],
  moduleNameMapper: {
    "\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga|(css|pcss)\\?global)$":
      "<rootDir>/config/tests/fileMock.js"
  },
  setupFiles: ["<rootDir>/config/tests/setupTests.ts"],
  setupFilesAfterEnv: ["<rootDir>/config/tests/setupTestsFramework.js"],
  transform: {
    ".+\\.(pcss|css|styl|less|sass|scss)$": "<rootDir>/node_modules/jest-css-modules-transform"
  },
  transformIgnorePatterns: ["/node_modules/(?!(lodash-es|other_es6_library)).+\\.js$"]
};
```

### enzyme

```bash
npm install --save-dev @types/enzyme @types/enzyme-adapter-react-16
```

## Resources

* migrating a large codebase: [link](https://blog.usejournal.com/migrating-a-flow-react-native-app-to-typescript-c74c7bceae7d)
* using babel-loader instead of ts-loader: [link](https://www.mattzeunert.com/2019/10/23/migrating-your-webpack-typescript-build-from-ts-loader-to-babel-loader.html)