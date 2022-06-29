# Typescript class transformation bug

- `npm install`
- `npm test`

When targetting ES5, the following transformer causes Typescript to crash during the Typescript Class transformation step:

```typescript
import * as ts from 'typescript';

const transformer: (program: ts.Program) => ts.TransformerFactory<ts.SourceFile> = (program: ts.Program) => {
    const transform: ts.TransformerFactory<ts.SourceFile> = (context: ts.TransformationContext) => {
        return sourceFile => {
            console.log(`Transforming source file '${sourceFile.fileName}'...`);
            function visitor(node: ts.Node) {
                if (ts.isClassDeclaration(node)) {
                    return ts.factory.updateClassDeclaration(
                        node,
                        ts.factory.createNodeArray(node.decorators),
                        node.modifiers,
                        node.name,
                        node.typeParameters,
                        node.heritageClauses,
                        node.members
                    )
                }
                return ts.visitEachChild(node, visitor, context);
            }

            return ts.visitNode(sourceFile, visitor);
        }
    };

    return transform;
};

export default transformer;
```

It crashes very specifically on a class with a static property that has an initializer, like so:

```typescript
class A {
    static stuff = 'things';
}
```

Result of `npm test`:

```
PS D:\Dev\typescript-rtti\ts-transform-bug> npm test

> test
> npm run clean && cd transformer && npm run build && cd .. && cd program && npm run build


> clean
> rimraf program/dist transformer/dist


> build
> tsc -b


> build
> ttsc

Transforming source file 'D:/Dev/typescript-rtti/ts-transform-bug/program/src/main.ts'...
D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:117612
                throw e;
                ^

Error: Debug Failure. Invalid cast. The supplied value [object Object] did not pass the test 'isCallExpression'.
    at Object.cast (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:1847:25)
    at visitTypeScriptClassWrapper (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:101166:27)
    at visitCallExpression (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:101086:24)
    at visitorWorker (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:98718:28)
    at visitor (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:98639:44)
    at visitNode (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:89314:23)
    at Object.visitEachChild (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:89826:236)
    at visitVariableDeclaration (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:100136:30)
    at visitorWorker (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:98680:28)
    at visitor (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:98639:44)
PS D:\Dev\typescript-rtti\ts-transform-bug> 
```

Changing out 

```typescript
ts.factory.createNodeArray(node.decorators),
```

For

```typescript
[].concat(node.decorators)
```

Yields:

```
Transforming source file 'D:/Dev/typescript-rtti/ts-transform-bug/program/src/main.ts'...
D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:117612
                throw e;
                ^

TypeError: Cannot read property 'expression' of undefined
    at transformDecorator (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:92611:43)
    at Object.map (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:638:29)
    at transformAllDecoratorsOfDeclaration (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:92472:50)
    at generateConstructorDecorationExpression (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:92589:40)
    at addConstructorDecorationStatement (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:92577:30)
    at visitClassDeclaration (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:92066:13)
    at visitTypeScript (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:91924:28)
    at visitorWorker (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:91711:24)
    at sourceElementVisitorWorker (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:91736:28)
    at saveStateAndInvoke (D:\Dev\typescript-rtti\ts-transform-bug\node_modules\typescript\lib\typescript.js:91649:27)
```