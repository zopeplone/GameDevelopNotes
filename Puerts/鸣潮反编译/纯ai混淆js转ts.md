
## 代码导入和风格转换
1. 在文件头部的 `require` 全部替换为 `import` ，对于xxx_1，xxx_2的导入，合并并全部都通过解构的方式导入模块内变量
2. 对于动态导入的require，如果是同步的则不需要管，如果是`new Promise`包起来的require，用import()函数来代替
3. 对于UE，cpp，puerts，全部都使用命名空间导入
4. 对于纯js模块，如 `EventSystem_1.EventSystem.RemoveWithTarget` 则使用解构导入，将`EventSystem_1`去掉，解构导入`EventSystem`
5. 对于defult导入的js，如`TsAiController.js`，这种在其他地方使用时，都是 `TsAiController_1.default` 来使用，这种很好判断，直接default导入即可
6. 在文件末尾，的`(((exports.AppLinks = AppLinks).S_e = !1), (AppLinks.u2n = new Map()));`，可以识别出导出了哪些东西，以及初始化了这个类中的哪些static变量，识别出的static变量必须要放到class中定义
7. 在文件头部的require下，用逗号还声明了许多模块内的常量，这些也需要用另外的const去处理，每个常量一个const
8. `(Object.defineProperty(exports, "__esModule", { value: !0 })` 对于文件顶部的这个defineProperty不需要
9. 将`void 0` 改为 undefined，将`!0`改为true, `!1`改为false
## Ts和变量名解混淆

1. 对于每个混淆后的变量名，你需要