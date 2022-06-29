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