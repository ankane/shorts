## Package Your JavaScript Libraries With Rollup

[Rollup](https://rollupjs.org/guide/en) is a great tool for building libraries.

> [“Webpack for apps, and Rollup for libraries”](https://medium.com/webpack/webpack-and-rollup-the-same-but-different-a41ad427058c)

Run:

```sh
yarn add rollup rollup-plugin-buble rollup-plugin-commonjs rollup-plugin-node-resolve rollup-plugin-uglify --dev
```

Add to `package.json`:

```json
{
  "main": "dist/my-project.js",
  "module": "dist/my-project.esm.js",
  "scripts": {
    "build": "rollup -c"
  }
}
```

Add `dist/` to your `.gitignore`.

Create `rollup.config.js` with:

```es6
import buble from "rollup-plugin-buble";
import commonjs from "rollup-plugin-commonjs";
import pkg from "./package.json";
import resolve from "rollup-plugin-node-resolve";
import uglify from "rollup-plugin-uglify";

const input = "src/index.js";
const outputName = "MyProject";
const external = Object.keys(pkg.peerDependencies || {});
const esExternal = external.concat(Object.keys(pkg.dependencies || {}));
const banner =
`/*
 * ${pkg.name}
 * ${pkg.description}
 * ${pkg.repository.url}
 * v${pkg.version}
 * ${pkg.license} License
 */
`;

export default [
  {
    input: input,
    output: {
      name: outputName,
      file: pkg.main,
      format: "umd",
      banner: banner
    },
    external: external,
    plugins: [
      resolve(),
      commonjs(),
      buble()
    ]
  },
  {
    input: input,
    output: {
      name: outputName,
      file: pkg.main.replace(/\.js$/, ".min.js"),
      format: "umd"
    },
    external: external,
    plugins: [
      resolve(),
      commonjs(),
      buble(),
      uglify()
    ]
  },
  {
    input: input,
    output: {
      file: pkg.module,
      format: "es",
      banner: banner
    },
    external: esExternal,
    plugins: [
      buble()
    ]
  }
];
```

And run:

```sh
yarn build
```

:fire:
