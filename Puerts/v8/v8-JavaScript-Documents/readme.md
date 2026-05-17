# 《Chrome V8源码》系列技术文章，3~4天一篇，持续更新中...   
![avatar](v8.png)
# 内容特点  
V8涉及的技术十分广泛，包括了操作系统、编译技术、计算机系统结构等多方面知识。《Chrome V8源码》系列文章从基础说起，对V8内存分配、Isolate创建、handles概念、builtin、codegen、编译等多方面进行详细讲解。**本文章正在更新中，3~4天一篇。**   
本文章的讲解风格是直接面对V8源码、分析V8源码的执行过程和重要数据结构，力求为读者展示一个“活动”的V8引擎。  
# 读者定位  
想深入理解V8源码，从中学习优秀设计思想，亦或是做V8安全研究的读者，等等。总之，想读V8源码的读者，请进来看看。  
# 作者介绍  
灰豆，V8粉丝一枚，正在努力做一名V8源码的朗读者。   
微信：qq9123013 备注：v8交流 
# 文章位置  
**知乎** ： https://www.zhihu.com/people/v8blink
# 不足之处  
文章中理论知识很少。我也想多讲解些理论，但个人能力和时间精力确实有限，无法做全面的讲解。文中有很多的不足与纰漏，欢迎批评指正。  

# 文章目录  
目前的最新目录如下：   
[1.V8环境搭建](./chap1-0829/README.md)  
[2.鸟瞰V8运行过程，形成大局观](./chap2-0914/README.md)  
[3.看V8编译流程，学习词法分析](./chap3-0915/README.md)  
[4.V8词法分析源码讲解，Token字生成](./chap4-0917/README.md)  
[5.V8语法分析器源码讲解](./chap5-0918/README.md)  
[6.bytecode字节码生成](./chap6-0922/README.md)  
[7.V8堆栈框架 Stack Frame](./chap7-0925/README.md)  
[8.解释器Ignition](./chap8-0929/README.md)  
[9.Builtin源码分析](./chap9-1002/README.md)  
[10.V8 Execution源码分析](./chap10-1003/README.md)  
[11.字节码调度 Dispatch机制](./chap11-1006/README.md)  
[12.JSFunction源码分析](./chap12-1009/README.md)  
[13.String类方法的源码分析](./chap13-1020/README.md)  
[14.看V8如何表示JS的动态类型](./chap14-1019/README.md)  
[15.运行时辅助类，给V8加钩子函数](./chap15-1024/README.md)  
[16.运行时辅助类，详解加载与调用过程](./chap16-1025/README.md)  
[17.JS对象的内存布局与创建过程](./chap17-1026/README.md)  
[18.利用汇编看V8，洞察看不见的行为](./chap18-1031/README.md)  
[19.V8 Isolate核心组件：编译缓存](./chap19/README.md)  
[20.编译链1：语法分析，被遗忘的细节](./chap20/README.md)  
[21.编译链2：Token和AST，被忽略的秘诀](./chap21/README.md)  
[22.编译链3：Bytecode的秘密——常量池](./chap22/README.md)  
[23.编译4：数据结构、重要的自动机](./chap23/README.md)  
[24.编译5：SharedFunction与JSFunction的渊源](./chap24/readme.md)  
[25.最难啃的骨头——Builtin！](./chap25/readme.md)  
[26.Bytecode Handler，字节码的核心](./chap26/readme.md)  
[27.神秘又简单的dispatch_table_](./chap27/readme.md)  
[28.分析substring源码和隐式约定](./chap28/readme.md)  
[29.CallBuiltin()调用过程详解](./chap29/readme.md)  
[30.Ignition到底做了什么？](./chap30/readme.md)  
[31.Ignition到底做了什么？（二）](./chap31/readme.md)  
[32.字节码和 Compiler Pipeline 的细节](./chap32/readme.md)  
[33. Lazy Compile 的技术细节](./chap33/readme.md)  
[34. 终级优化技术 Turbofan](./chap34/readme.md)  
[35. String.prototype.CharAt 源码分析](./chap35/readme.md)  
[36. String.prototype.concat 源码分析](./chap36/readme.md)  
[37. String.prototype.match 源码分析](./chap37/readme.md)  
[38. replace 技术细节、性能影响因素](./chap38/readme.md)  
[39. String.prototype.split 源码分析](./chap39/readme.md)  
[40. Runtime substring 详解](./chap40/readme.md)  
[41. Runtime_StringTrim 源码、触发条件](./chap41/readme.md)  
[42. InterpreterEntryTrampoline 与优化编译](./chap42/readme.md)  
[43. Turbofan 源码分析](./chap43/readme.md)  
[44. Runtime_StringToArray 源码、触发条件](./chap44/readme.md)  
[45. JavaScript API 源码分析（1）](./chap45/readme.md)  
[46. BigInt的疑惑](./chap46/readme.md)  
[47. "Equal" 与 "StrictEqual" 为什么不同](./chap47/readme.md)  
[48. 弱类型加法的奥秘，"+" 源码分析](./chap48/readme.md)  
[49. new String 和 String 的本质区别](./chap49/readme.md)  
[50.JS 数值转换的内幕，Number() 源码分析](./chap50/readme.md)  
[51. 揭开 bind 和 call 的神秘面纱](./chap51/readme.md)  
[52. 解密JSON序列化、stringify源码分析](./chap52/readme.md)  
[53. Inline Cache 源码（一）](./chap53/readme.md)  
[54. Inline Cache 源码（二）](./chap54/readme.md)  
[55. 优化技术综述，如何提升 JS 运行速度](./chap55/readme.md)  
[56. GC 垃圾回收 1](./chap56/readme.md)  
