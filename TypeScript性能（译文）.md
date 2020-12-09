> # 🛑 请先仔细阅读全文，然后再给出结论！

有一些便捷的方法通过配置TypeScript，便可获得更快的编译和代码编写时体验。
而越早采用这些实践则越好。
除了最佳实践之外，本文还给出一些用于调查导致编译/代码编写时体验差的常用技巧，一些常见的修复方法，行文最后，给出几种帮助TypeScript团队调查问题的常用方法。

- [编写易于编译的代码](#编写易于编译的代码)
  * [优先采用接口，而不是交叉类型](#优先采用接口，而不是交叉类型)
  * [使用类型注解](#使用类型注解)
  * [优先采用基本类型，而不是联合类型](#优先采用基本类型，而不是联合类型)
- [使用项目引用](#using-project-references)
- [配置 `tsconfig.json` 或 `jsconfig.json`](#配置 `tsconfig.json` 或 `jsconfig.json`)
  * [指定文件](#指定文件)
  * [控制 `@types` 的包含](#控制 `@types` 的包含)
  * [增量项目编译](#增量项目编译)
  * [跳过 `.d.ts` 检查](#跳过 `.d.ts` 检查)
  * [使用更高效的差异检查](#使用更高效的差异检查)
- [配置其他构建工具](#配置其他构建工具)
  * [同步的类型检查](#同步的类型检查)
  * [独立的文件生成](#独立的文件生成)
- [调查问题](#调查问题)
  * [禁用编辑器插件](#禁用编辑器插件)
  * [`extendedDiagnostics`](#extendeddiagnostics)
  * [`showConfig`](#showconfig)
  * [`traceResolution`](#traceresolution)
  * [单独运行 `tsc` 命令](#单独运行 `tsc` 命令)
  * [升级依赖](#升级依赖)
- [常见问题](#常见问题)
  * [`include` 和 `exclude`配置错误](#`include` 和 `exclude`配置错误)
- [提交问题](#提交问题)
  * [报告编译器性能问题](#报告编译器性能问题)
    + [分析编译器](#分析编译器)
  * [报告代码编写时性能问题](#报告代码编写时性能问题)
    + [记录TSServer日志](#记录TSServer日志)
      - [在Visual Studio Code中收集TSServer日志](#在Visual Studio Code中收集TSServer日志)

# 编写易于编译的代码

## 优先采用接口，而不是交叉类型

大多数情况下，对象类型的简单类型别名与接口的作用非常相似。

```ts
interface Foo { prop: string }

type Bar = { prop: string };
```

但是，一旦你需要将两个或更多类型组合起来，你可以选择用接口来继承这些类型，或者创建一个新的类型别名，该别名由众多类型执行交叉操作得到，这时两者之间的差异就开始变得不可忽略了。

接口创建了一个单一的扁平化的对象类型来检测属性冲突，这些冲突通常必须被解决！ 另一方面，交叉类型只是递归地合并属性，在某些情况下会产生 `never`类型。 接口在使用一致性上更胜一筹，而交叉类型的类型别名不能在部分其他交叉类型中使用。 接口之间的类型关系也被缓存，这点总体上与交叉类型相反。 最后一个值得注意的区别是，当针对交叉类型进行检查时，在针对“有效”/“平铺”的类型进行检查之前，会检查类型的每个组成部分。 

正因如此，建议使用 `interface`/`extends`来做类型继承，而不是创建交叉类型。

```diff
- type Foo = Bar & Baz & {
-     someProp: string;
- }
+ interface Foo extends Bar, Baz {
+     someProp: string;
+ }
```

## 使用类型注解

添加类型注解，特别是返回值类型的注解，可以节省编译器的大量工作。
在某种程度上，这是因为具名类型往往比匿名类型（编译器可能也会推断出）更快捷高效，它减少了编译器读写声明文件所花费的时间（例如，在增量构建场景中）。

类型推断非常便捷，所以没有必要普遍要求增加类型注解——但是，如果你已经检测出代码的某个部分运行低效，那么尝试一下这个方法可能会有用。

```diff
- import { otherFunc } from "other";
+ import { otherFunc, otherType } from "other";

- export function func() {
+ export function func(): otherType {
      return otherFunc();
  }
```

## 优先采用基本类型，而不是联合类型

联合类型很棒——它们让你可以描述一种类型的可能取值的范围。

```ts
interface WeekdaySchedule {
  day: "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday";
  wake: Time;
  startWork: Time;
  endWork: Time;
  sleep: Time;
}

interface WeekendSchedule {
  day: "Saturday" | "Sunday";
  wake: Time;
  familyMeal: Time;
  sleep: Time;
}

declare function printSchedule(schedule: WeekdaySchedule | WeekendSchedule);
```

然而，这也是有代价的。 
每次将一个参数传递给`printSchedule`时，必须将其与联合类型的每个元素进行比较。 
对于只有两个元素的联合类型来说，这是不耗费资源具有可操作性的。 
但是，如果你的联合类型有数十个以上的元素，对编译速度造成的影响则不可忽略。
例如，为了从联合类型中消除冗余成员，元素必须两两比较，这是平方关系的复杂度。 
这种检查可能发生在对大型联合类型进行交叉操作时，在每项联合成员间执行交叉操作可能导致大量类型需要被去除。 
规避这种情况的一种方法是使用派生子类型，而不是联合类型。

```ts
interface Schedule {
  day: "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday" | "Saturday" | "Sunday";
  wake: Time;
  sleep: Time;
}

interface WeekdaySchedule extends Schedule {
  day: "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday";
  startWork: Time;
  endWork: Time;
}

interface WeekendSchedule extends Schedule {
  day: "Saturday" | "Sunday";
  familyMeal: Time;
}

declare function printSchedule(schedule: Schedule);
```

一个更贴近实际应用的例子是，当试图对每个内置的DOM元素类型进行建模描述时。 
在这个例子中，最好创建一个带有公共成员的基本类型`HtmlElement` ，让比如`DivElement`、`ImgElement`等类型来继承，而不是创建一个详尽的联合类型` DiveElement |/*...*/ | ImgElement | /*...*/`.

# 使用项目引用

当使用TypeScript构建有一定复杂度的代码库时，将代码库组织成几个独立的*项目*是很有帮助的。
每个项目都有自己的`tsconfig.json`，可依赖于其他项目。 
这有助于避免在一次编译中加载过多文件，也使某些代码库从展示结构上更容易组合。 

有一些非常基本的方法可以[将代码库拆解成项目](https://www.typescripttlang.org/docs/handbook/project-references.html)。 
例如，一个程序可能由一个客户端项目、一个服务器项目和一个两者共享的项目组成。

```
              ------------
              |          |
              |  Shared  |
              ^----------^
             /            \
            /              \
------------                ------------
|          |                |          |
|  Client  |                |  Server  |
-----^------                ------^-----
```

测试案例也能够拆解到各自的项目中。

```
              ------------
              |          |
              |  Shared  |
              ^-----^----^
             /      |     \
            /       |      \
------------  ------------  ------------
|          |  |  Shared  |  |          |
|  Client  |  |  Tests   |  |  Server  |
-----^------  ------------  ------^-----
     |                            |
     |                            |
------------                ------------
|  Client  |                |  Server  |
|  Tests   |                |  Tests   |
------------                ------------
```

一个经常被问及的问题是“一个项目的体积在多大范围内是合适的？”。 
这与问题“一个函数应该写多长？”或者“一个类应该写多长？”本质上很类似，而且，在很大程度上，这取决于经验。
一种常见的拆分JS/TS代码的方法是使用文件夹。 
受此启发，如果代码关联性足够强，可放在同一个文件夹中，它们应归属到同一个项目中。 
另外，避免将项目设置得过大或者过小。 
如果一个项目的体积比其他所有项目的总和还要大，这是一个警告信号。 
同样，最好避免较多的单文件项目，因为开销会增加。 

你可以[在这里阅读更多关于项目引用的资料](https://www . typescripttlang . org/docs/handbook/project-references . html)。

# 配置 `tsconfig.json` 或 `jsconfig.json`

TypeScript和JavaScript用户总是可以用一个`tsconfig.json`文件来对编译进行配置。
对JavaScript用户而言，[`jsconfig.json` 文件也可以对代码编写时体验进行配置](https://code.visualstudio.com/docs/languages/jsconfig)。

## 指定文件

你应该始终确保你的配置文件不会一下子包含太多文件。 

在`tsconfig.json`中，有两种方法可以指定项目中的文件。

* `files` 列表
* `include` 和 `exclude` 列表

这两项之间的主要区别在于，`files` 选项预期被设置成源文件的文件路径列表，而`include`/`exclude`则使用通配符来匹配文件。

虽然指定`files`将使TypeScript能够快速直接加载文件，但如果您的项目中有过多文件而没有几个一级入口点，那么加载过程将会变得笨重而复杂。
此外，向你的`tsconfig.json`中添加新文件很容易被遗漏，这意味着你可能会遇到奇怪的编辑器行为，新文件会被错误地解析。 
所有这些都很笨重而复杂。

` include`/`exclude` 不需要具体指定文件列表，但代价是：必须通过遍历包含的目录来查找文件。 
当执行*大量*文件夹查找时，这势必会拉低编译过程的速度。 
此外，有时编译会包含许多不必要的`.d.ts` 文件和测试文件，这会增加编译时间和内存开销。 
最后，虽然`exclude` 设置了合理的默认值，但某些配置(如在mono-repos单体源代码库项目中)意味着像`node_modules `这样的“沉重”的文件夹最终仍可能被包括在内。 

我们建议如下最佳实践原则:

* 仅指定项目中的“可输入”的文件夹(即，指想要包含其内部源代码以进行编译/分析的文件夹)。
* 同一个文件夹中不要混入其他项目的源文件。
* 如果将测试文件与其他源文件放置在同一个文件夹中，那么给这些测试文件特殊的命名，以便它们可以很容易地被排除在外。
* 在源代码目录中应该将体积大的构建产物文件和例如 `node_modules` 这样的依赖文件排除在外。

注: 如果`exclude` 列表没有设置， 那么`node_modules`是默认被排除在外的；
一旦往这个列表中添加了内容，就必须明确地将`node_modules`也添加到该列表中。

如下这个正确配置的 `tsconfig.json` 具体说明了这个规则。

```json5
{
    "compilerOptions": {
        // ...
    },
    "include": ["src"],
    "exclude": ["**/node_modules", "**/.*/"],
}
```

## 控制 `@types` 的包含

默认情况下，无论是否导入，TypeScript都会自动包含它在`node_modules`文件夹中找到的每个`@types` 包。
这是为了当使用Node.js、Jasmine、Mocha、Chai时，在某些情况下“正常工作”。由于这些工具/包不是导入进来的，它们只是被加载到全局环境中。 

有时，这种策略会降低编译和编写时的程序构建时间，甚至会导致多个全局包出现声明冲突的问题，引发如下错误

```
Duplicate identifier 'IteratorResult'.
Duplicate identifier 'it'.
Duplicate identifier 'define'.
Duplicate identifier 'require'.
```

如果不需要全局包，修改工作非常简单，只需在`tsconfig.json`/`jsconfig.json`文件中，给 [ `"types"` 选项](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html#types-typeroots-and-types)配置成空值即可。

```json5
// src/tsconfig.json
{
   "compilerOptions": {
       // ...

       // Don't automatically include anything.
       // Only include `@types` packages that we need to import.
       "types" : []
   },
   "files": ["foo.ts"]
}
```

如果仍然需要一些全局包，那么将它们添加到 `types` 字段中即可。

```json5
// tests/tsconfig.json
{
   "compilerOptions": {
       // ...

       // Only include `@types/node` and `@types/mocha`.
       "types" : ["node", "mocha"]
   },
   "files": ["foo.test.ts"]
}
```

## 增量项目构建

`--incremental`标志允许TypeScript将上次编译时的状态保存到`.tsbuildinfo`文件中。
该文件用于找出自上次运行以来可能要重新检查/重新触发构建的最小文件集，与TypeScript的`--watch`模式的机制很类似。

在使用项目引用功能时，使用`composite`标志位，默认情况下会启用增量编译，但是可为所有选择加入的项目带来相同的加速。

## 跳过 `.d.ts` 检查

默认情况下，TypeScript会对项目中的所有`.d.ts`文件进行完整的重新检查，以发现问题和不一致之处； 然而，这通常是不必要的。
大多数时候， `.d.ts` 文件已经可以正常运行——类型之间相互继承的方式已经得到过验证，然而，这些重要的声明无论如何都被检查。

TypeScript提供了使用`skipDefaultLibCheck`标志来跳过附带的`.d.ts`文件（例如`lib.d.ts`）的类型检查的方法。

另外，你也可以启用`skipLibCheck`标志来跳过对编译中*所有*`.d.ts`文件的检查。

这两个选项通常会在`.d.ts`文件中隐藏配置错误和冲突，因此我们建议*仅*为提高构建速度才使用它们。

## 使用更高效的差异检查

Dog列表是Animal列表吗？
也就是说，` List<Dog > `可赋值给` List<Animals > `吗？ 
找到答案的直接方法是逐个成员地对类型进行结构上的比较。 
不幸的是，这可能开销很大。 
然而，如果我们掌握足够关于`List<T>`知识，我们可以将这种赋值检查简化为确定 `Dog` 是否可赋值给`Animal`(即不考虑`List<T>`的每个成员)。 
(特别是我们需要知道类型参数`T`的[差异](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) 。) 
只有启用`strictFunctionTypes` 标志，编译器才能充分利用这种潜在的加速(否则，它会使用较慢但更宽松的结构检查)。 
因此，我们建议构建时使用`--strictFunctionTypes`选项，（在`--strict`模式下该选项是默认开启的)。

# 配置其他构建工具

考虑到对TypeScript的编译工作通常是在使用其他构建工具的情况下完成的，尤其是在编写可能涉及打包工具的web应用程序时。 
虽然我们只能对少数构建工具提出建议，但理想情况下，这些技术可以被推广。

除了阅读本节之外，还可阅读选用的构建工具的性能，例如:

* [ts-loader关于 *快速构建*的章节](https://github.com/TypeStrong/ts-loader#faster-builds)
* [awesome-typescript-loader关于 *性能问题*的章节](https://github.com/s-panferov/awesome-typescript-loader/blob/master/README.md#performance-issues)

## 同步的类型检查

类型检查通常需要来自其他文件的信息，与转换/生成代码等其他步骤相比相对消耗性能。 
因为类型检查可能需要更长一点的时间，它会影响内部开发循环——换句话说，您可能会经历更长的编写/编译/运行循环，这可能会令人沮丧。 

由于这个原因，一些构建工具可以在单独的进程中运行类型检查，而不会阻塞代码生成。
虽然这意味着在TypeScript在构建工具中报错之前可能会运行无效代码，但你通常还是会首先在编辑器中看到错误，并且阻碍你运行正确代码的时间不会太长。

采用这个机制的一个实例是Webpack插件 [`fork-ts-checker-webpack-plugin`](https://github.com/TypeStrong/fork-ts-checker-webpack-plugin) ，或者 [awesome-typescript-loader](https://github.com/s-panferov/awesome-typescript-loader) 有时也采取这种策略。

## 单独的文件生成

默认情况下，TypeScript的生成需要语义信息，这些信息可能不是本地文件拥有的。
这是为了理解如何生成诸如 `const enum`和 `namespace`之类的特性。
但是需要检查*其他*文件以输出构建产物文件，可能会使生成速度变慢。

对需要非本地信息的特性的需求某种程度上少见——可以使用常规的 `enum`代替`const enum`，并且可以使用模块代替`namespace`。
因此，TypeScript提供了`isolatedModules`编译选项，当使用非本地信息支持的特性时，会给出报错信息。
启用`isolatedModules`意味着您的代码库对于使用TypeScript API（例如，`transpileModule`）或其他编译器（例如Babel）的工具是安全的。

例如，使用单独的文件转换，以下代码无法在运行时正常运行，因为预计`const enum`值将会被内联进来； 但是幸运的是，`isolatedModules`会在早期提示我们。

```ts
// ./src/fileA.ts

export declare const enum E {
    A = 0,
    B = 1,
}

// ./src/fileB.ts

import { E } from "./fileA";

console.log(E.A);
//          ~
// error: Cannot access ambient const enums when the '--isolatedModules' flag is provided.
```

> **牢记:** `isolatedModules` 不会自动使代码生成更快——它只是在当你使用不受支持的功能时，会提示你。
> 您正在寻求解决的是不同的构建工具和API中进行单独模块生成。

通过使用如下工具，能够很好的实现单独的文件生成达成这个目标。

* [ts-loader](https://github.com/TypeStrong/ts-loader) 提供 [ `transpileOnly` 标志位](https://github.com/TypeStrong/ts-loader#transpileonly-boolean-defaultfalse) ，它通过使用 `transpileModule`来执行单独文件生成任务。
* [awesome-typescript-loader](https://github.com/s-panferov/awesome-typescript-loader) 提供 [`transpileOnly` 标志位](https://github.com/s-panferov/awesome-typescript-loader/blob/master/README.md#transpileonly-boolean) ，它通过使用 `transpileModule`来执行单独文件生成任务。
* [TypeScript的 `transpileModule` API](https://github.com/microsoft/TypeScript/blob/a685ac426c168a9d8734cac69202afc7cb022408/lib/typescript.d.ts#L5866) 能够被直接使用。
* [awesome-typescript-loader 提供 `useBabel` 标志位](https://github.com/s-panferov/awesome-typescript-loader/blob/master/README.md#usebabel-boolean-defaultfalse)。
* [babel-loader](https://github.com/babel/babel-loader) 使用独立的方式来编译文件 (但是自身不提供类型检查）。
* [gulp-typescript](https://www.npmjs.com/package/gulp-typescript) 能够执行单独文件生成任务，只须 `isolatedModules` 选项被启用。
* [rollup-plugin-typescript](https://github.com/rollup/rollup-plugin-typescript) ***只*** 执行单独文件编译任务.
* [ts-jest](https://kulshekhar.github.io/ts-jest/) 能够通过将 [`isolatedModules` 标志位设置成true`]isolatedModules: true(.
* [ts-node](https://www.npmjs.com/package/ts-node) 能够检测[`tsconfig.json`包含`"ts-node"`字段中的`"transpileOnly"`选项，并且也有个`--transpile-only`标志](https://www.npmjs.com/package/ts-node#cli-and-programmatic-options).

# 调查问题

下面是一些可以提示是哪里可能出了错误的方法。

## 禁用编辑器插件

插件会影响到编写代码时的体验。
尝试禁用插件（尤其是与JavaScript / TypeScript相关的插件），看看是否可以解决某些性能和响应时间方面的问题。

某些编辑器还具有自己的性能问题的排查指南，所以考虑仔细阅读它们。
例如，Visual Studio Code也有自己的网页介绍 [性能问题](https://github.com/microsoft/vscode/wiki/Performance-Issues) 。

## `extendedDiagnostics`

运行TypeScript时，可以使用编译选项`--extendedDiagnostics`，它可以打印输出编译器在各个环节的消耗时间。

```
Files:                         6
Lines:                     24906
Nodes:                    112200
Identifiers:               41097
Symbols:                   27972
Types:                      8298
Memory used:              77984K
Assignability cache size:  33123
Identity cache size:           2
Subtype cache size:            0
I/O Read time:             0.01s
Parse time:                0.44s
Program time:              0.45s
Bind time:                 0.21s
Check time:                1.07s
transformTime time:        0.01s
commentTime time:          0.00s
I/O Write time:            0.00s
printTime time:            0.01s
Emit time:                 0.01s
Total time:                1.75s
```

> 注意，`Total time` 不是它之前列出的所有时间的总和，因为有一些重叠任务和另外一些工作的耗费时间没有被测算。

对大多数用户来说，最重要的信息如下:

| 字段                 | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| `Files`              | 程序包含的文件的数量 (可使用 `--listFiles` 看到具体是那些文件)。 |
| `I/O Read time`      | 读取文件系统所花费的时间——包括遍历`includes`文件夹的时间。   |
| `Parse time`         | 扫描和解析程序所花费的时间                                   |
| `Program time`       | 执行读取文件系统、扫描和解析程序以及程序图谱的其他计算综合起来所花费的时间。这些步骤在这里是混合和组合的，因为文件一旦通过`import`和`export`被包括进来，就需要被解析和加载。 |
| `Bind time`          | 构建单个文件本地的各种语义信息所花费的时间。                 |
| `Check time`         | 对程序执行类型检查花费的时间。                               |
| `transformTime time` | 将TypeScript抽象语法树（代表源文件的树结构）重写成能够在老的运行时环境执行的格式所花费的时间。 |
| `commentTime`        | 在输出文件中计算注释要花费的时间。                           |
| `I/O Write time`     | 在磁盘上写/更新文件花费的时间。                              |
| `printTime`          | 计算输出文件的字符串表示形式，并将其写到磁盘所花费的时间。   |

考虑到这些信息，你也许会问一些问题：

* 文件数量/代码行数是否与你项目中的文件数量大致对应？如果没有，请尝试运行`-- listFiles`。
* `Program time` 或 `I/O Read time` 是否相当高? [确保 `include`/`exclude` 设置配置正确](#misconfigured-include-and-exclude).
* 其他时间似乎不对劲吗? [也许你想要提出问题!](#提交问题) 能够帮助诊断问题的一些方法如下：
  * 如果 `printTime` 很大，那么运行时开启`emitDeclarationOnly`选项。
  * 阅读关于 [报告编译器性能问题](#报告编译器性能问题)的说明。

## `showConfig`

运行`tsc`时，编译时是使用了怎样的配置，并不总是显而易见的，特别是由于`tsconfig.json`可以继承其他配置文件。
 `showConfig`可以说明`tsc`为调用过程所做的计算。

```sh
tsc --showConfig

# or to select a specific config file...

tsc --showConfig -p tsconfig.json
```

## `traceResolution`

运行时使用 `traceResolution` 选项有助于解释编译过程中*为什么*某个文件被包含进来了。 这个解释内容有点冗长，所以你可以将输出重定向到一个文件中。

```sh
tsc --traceResolution > resolution.txt
```

如果您发现一个不应该被包含进来的文件，你可能需要在`tsconfig.json`中修复`include`/`exclude`列表，或者，可能需要调整其他设置，如`types`, `typeRoots`, 或`paths`。

## 单独运行 `tsc` 

在大多数情况下，用户使用第三方构建工具(如Gulp, Rollup, Webpack等)时会遇到性能低下的问题。 通过运行`tsc --extendedDiagnostics`来发现使用TypeScript和这些构建工具之间的主要差异，从而可能表明是外部环境的错误配置或低效导致。

留意一些问题：

* `tsc`和集成TypeScript的构建工具之间的构建时间有很大不同吗？
* 如果构建工具提供诊断功能，那么TypeScript和构建工具的之间的解析是否有差异？
* 是否是构建工具*自身的配置*导致的问题？
* 是否是构建工具关于*TypeScript集成*的配置导致的问题(例如ts-loader的配置?)

## 升级依赖

有时TypeScript的类型检查会受到计算密集型的`d.ts`文件的影响。
这种情况很少见，但也有可能发生。
升级到较新版本的TypeScript(这可能更有效)或较新版本的`@types`包(也可能做了某项回滚操作)通常可以解决这个问题。

# 常见问题

问题解决后，你可能想浏览一些针对常见问题的解决方法。
如果以下解决方案没能起作用，则值得[提交问题](#提交问题)。

## 错误配置了 `include` 和 `exclude`

正如上文所述， `include`/`exclude`选项可能有多种错误使用方式。

| 问题                                         | 原因                             | 解决                                       |
| -------------------------------------------- | -------------------------------- | ------------------------------------------ |
| 深层次文件夹中的`node_modules意外被包含进来  | *`exclude` 未被设置*             | `"exclude": ["**/node_modules", "**/.*/"]` |
| 深层次文件夹中的`node_modules意外被包含进来  | `"exclude": ["node_modules"]`    | `"exclude": ["**/node_modules", "**/.*/"]` |
| 某些隐藏的文件（例如.git文件）被意外包含进来 | `"exclude": ["**/node_modules"]` | `"exclude": ["**/node_modules", "**/.*/"]` |
| 包含了预期之外的文件                         | *`include` 未被设置*             | `"include": ["src"]`                       |

# 提交问题

如果您的项目已经正确且最佳地配置，则可能要[提交问题](https://github.com/microsoft/TypeScript/issues/new/choose)。

关于性能问题的最佳报告形式应当要要求该问题*可重现*且有*较少*的重现步骤。
换句话说，应该提供只包含少量文件并且可从git上克隆下来的代码库。
它们不需要与构建工具进行外部集成——他们可通过`tsc`命令调用，或者是使用TypeScript API的独立代码。
调用或者启动方式繁琐的代码库将不会被优先处理。

我们知道这并不总是容易实现的——特别是因为很难在代码库中分离出问题代码，并且因为共享代码可能会导致出现知识产权方面的问题。
在某些情况下，如果我们认为此问题具有很大的影响面，则团队将愿意邮寄保密协议（NDA）。

无论是否可以重现，在提交问题时遵循以下指导将有助于我们为你提供性能问题的修复。

## 报告编译器性能问题

有时，你可能会在构建时和编写场景时遇到性能问题。
在这些情况下，最好聚焦于TypeScript编译器。

首先，应该使用最新版本的TypeScript以确保你没有重复提交一个已经解决掉的问题：

```sh
npm install --save-dev typescript@next

# or

yarn add typescript@next --dev
```

编译器性能问题应包括

* 已安装的TypeScript的版本号(i.e. `npx tsc -v` or `yarn tsc -v`)
* TypeScript运行所基于的Node的版本号 (i.e. `node -v`)
* 使用编译选项 `extendedDiagnostics`运行时产生的输出内容 (`tsc --extendedDiagnostics -p tsconfig.json`)
* 理想情况下，有一个可以展示遇到问题的一个项目。
* 分析编译器的输出日志 (`isolate-*-*-*.log` 和 `*.cpuprofile` 文件)

### 分析编译器

运行附带`--trace-ic`选项的Node.js v10+版本，以及编译TypeScript时附带`--generateCpuProfile`选项，产生的诊断追溯信息对团队来说很重要：

```sh
node --trace-ic ./node_modules/typescript/lib/tsc.js --generateCpuProfile profile.cpuprofile -p tsconfig.json
```

这里 `./node_modules/typescript/lib/tsc.js` 可替换为你所安装的版本的TypeScript编译器的路径， `tsconfig.json` 可以为任何TypeScript配置文件。
`profile.cpuprofile`是你选择的输出文件。

这将生成两个文件：

* `--trace-ic` 生成文件 `isolate-*-*-*.log` (例如 `isolate-00000176DB2DF130-17676-v8.log`)。
* `--generateCpuProfile` 生成你所命名的文件。 上面这个例子中, 是命名为 `profile.cpuprofile`的文件。

> ⚠ 警告: 这些文件可能包含工作空间中的信息，包括文件路径和源代码。
> 这两个文件可被以纯文本形式读取，你可以在将它们作为GitHub问题的附加信息之前对其进行修改。 （例如，去掉可能暴露仅供内部使用的文件路径的相关信息）。
>
> 但是，如果你有关于在GitHub上公开发布这些内容的任何疑虑，请告诉我们，你可以私下共享详细信息。

## 报告代码编写时性能问题

能感知的代码编写时性能通常受许多因素影响，并且在TypeScript团队控制范围内的唯一事情是JavaScript / TypeScript语言服务的性能，以及该语言服务在与某些编辑器（例如Visual Studio，Visual Studio Code，Visual Studio for Mac和Sublime Text）中的集成性能。
确保在编辑器中关闭了所有第三方插件，以确定是否是TypeScript本身存在问题。

涉及代码编写时性能方面问题稍微多一点，但是也适用同样的思路：理想的可克隆的最小代码仓库，尽管在某些情况下，团队将能够签署保密协议来排查和分离出问题。

包含`tsc --extendedDiagnostics` 产生的输出文件是不错的上下文， 不过最有用的还是记录TSServer日志。

### 记录TSServer日志

#### 在Visual Studio Code中收集TSServer日志

1. 打开命令行面板，然后
   * 通过 `Preferences: Open User Settings`打开全局设置
   * 通过`Preferences: Open Workspace Settings`打开本地项目设置
1. 设置选项 `"typescript.tsserver.log": "verbose",`
1. 重启VS Code，复现问题
1. 在VS Code中, 运行 `TypeScript: Open TS Server log` 命令
1. 这将可以打开 `tsserver.log` 文件。

⚠ 警告: TSServer日志可能包含工作空间中的信息，包括文件路径和源代码。如果你有关于在GitHub上公开发布这些内容的任何疑虑，请告诉我们，你可以私下共享详细信息。
