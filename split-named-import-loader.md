### 描述
> 将具名导入的模块拆分为默认导入
```js
// input
import { a, b, c as d } from "@/utils"

// output
import a from "@/utils/a"
import b from "@/utils/b"
import d from "@/utils/c"
```
### 准备
```bash
pnpm add @babel/parser @babel/traverse @babel/traverse @babel/types -D
```
### 实现
```js
// split-named-import-loader.js
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generate = require("@babel/traverse").default;
const t = require("@babel/types");

module.exports = function (source) {
  const callback = this.async();
  const options = this.getOptions() || {};
  const splitModule = options.splitImportModule || [];

  try {
    const ast = parser.parse(source, {
      sourceType: "module",
      plugins: ["typescript", "jsx"],
    });

    const importsToAdd = [];

    traverse(ast, {
      ImportDeclaration(path) {
        const { node } = path;
        const sourceValue = node.source.value;

        if (!splitModule.includes(sourceValue)) return;

        const specifiers = node.specifiers.filter((s) => s.type === "ImportSpecifier");

        if (specifiers.length === 0) return;

        for (const specifier of specifiers) {
          const importedName = specifier.imported.name;
          const localName = specifier.local.name;
          const isRenamed = specifier.imported.name !== specifier.local.name;

          let newSource;
          if (sourceValue.endsWith("/")) {
            newSource = `${sourceValue}${importedName}`;
          } else {
            newSource = `${sourceValue}/${importedName}`;
          }

          const newNode = t.importDeclaration(
            [t.importDefaultSpecifier(t.identifier(isRenamed ? localName : importedName))],
            t.stringLiteral(newSource),
          );

          importsToAdd.push(newNode);
        }

        path.replaceWithMultiple(importsToAdd);
      },
    });

    const output = generate(ast, { jsescOption: { quotes: "double" } }).code;
    callback(null, output);
  } catch (err) {
    callback(err);
  }
};
```
