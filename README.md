
目录* [执行模型](https://github.com)
	+ [Iterator Model](https://github.com)
	+ [Materialization Model](https://github.com)
	+ [Vectoriazation Model](https://github.com)
	+ [对比](https://github.com)
* [数据访问方式](https://github.com):[豆荚加速器](https://yirou.org)
	+ [Sequential Scan](https://github.com)
	+ [Index Scan](https://github.com)
	+ [Multi\-Index Scan](https://github.com)
* [Halloween Problem](https://github.com)
* [表达式求值](https://github.com)

## 执行模型


执行模型（Processing Model）定义了数据库系统如何执行一个查询计划。


### Iterator Model


基本思想：采用树形结构组织操作符，然后中序遍历执行整棵树，最终根结点的输出就是整个查询计划的结果。


每个操作符（Operator）实现如下函数：


* `Next()`
	+ 返回值：一个tuple或者EOF。
	+ 执行流程：循环调用孩子结点的`Next()`函数。
* `Open()`和`Close()`：类似于构造和析构函数。


![image-20241118105113714](https://my-pic.miaops.sbs/2024/11/image-20241118105113714.png)


输出从底部向顶部（Bottom\-To\-Top）汇聚，且支持流式操作，所以又称为Valcano Model，Pipeline Model。


### Materialization Model


基本思想：操作符不是一次返回一个数据，暂存下所有数据，一次返回给父结点。


相比于Iterator Model，减少了函数调用开销，但是中间结果可能要暂存磁盘，IO开销大。


![image-20241118105733041](https://my-pic.miaops.sbs/2024/11/image-20241118105733041.png)


可以向下传递一些暗示（hint），如`Limit`，避免扫描过多的数据。


更适用于OLTP而不是OLAP。


### Vectoriazation Model


基本思想：操作符返回一批数据。


结合了Iterator Model和Materialization Model的优势，既减少了函数调用，中间结果又不至于过大。


可以采用SIMD指令加速批数据的处理。


![image-20241118110928540](https://my-pic.miaops.sbs/2024/11/image-20241118110928540.png)


### 对比




| **特性** | **Iterator Model** | **Materialization Model** | **Vectorization Model** |
| --- | --- | --- | --- |
| **数据处理单位** | 单条记录（tuple\-at\-a\-time） | 整个中间结果（table\-at\-a\-time） | 批量记录（vector/batch\-at\-a\-time） |
| **性能** | 函数调用开销高，效率低 | 延迟高，内存/I/O 开销大 | 函数调用开销低，SIMD 加速性能优异 |
| **内存使用** | 内存需求低 | 内存需求高 | 中等 |
| **I/O 开销** | 低 | 高 | 中等 |
| **缓存利用率** | 差 | 差 | 高 |
| **复杂性** | 实现简单 | 中等 | 实现复杂 |
| **适用场景** | 小型数据集，流式处理 | 中间结果复用的复杂查询 | 大型数据集，需高性能计算的场景 |


## 数据访问方式


主要有三种数据访问方式：


1. 全表扫描（Sequential Scan）
2. 索引扫描（Index Scan）
3. 多索引扫描（Multi\-Index Scan）


### Sequential Scan


全表扫描的优化手段：


![image-20241118113337122](https://my-pic.miaops.sbs/2024/11/image-20241118113337122.png)


Data Skipping方法：


1. 只需要大致结果：采样估计。
2. 精确结果：Zone Map


![image-20241118113508953](https://my-pic.miaops.sbs/2024/11/image-20241118113508953.png)


Zone Map基本思想：化整为零，提前对数据页进行聚合。


执行 `Select * From table Where val > 600`时，下面的页可以直接跳过。


![image-20241118113722074](https://my-pic.miaops.sbs/2024/11/image-20241118113722074.png)


### Index Scan


如何确定使用哪个索引：数据分布。


![image-20241118114047331](https://my-pic.miaops.sbs/2024/11/image-20241118114047331.png)


### Multi\-Index Scan


基本思想：根据每个索引上的谓词，独立找到满足条件的数据记录（Record），然后根据连接谓词进行操作（并集，交集，差集等）。


![image-20241118114343292](https://my-pic.miaops.sbs/2024/11/image-20241118114343292.png)


## Halloween Problem


对于UPDATE语句，需要追踪更新过的语句，否则会出现级联更新的问题。


![image-20241118114850271](https://my-pic.miaops.sbs/2024/11/image-20241118114850271.png)


\<999, Andy\>执行更新，走索引扫描：


1. 移除索引
2. 更新Tuple，\<1099, Andy\>
3. 插入索引
4. （约束检查）


此时，如果不对\<1099, Andy\>进行标记，他满足Where子句，会被重新更新一次。


## 表达式求值


基本思想：采用树形结构，构建表达式树，用中序遍历方式执行所有求值动作，根结点的求值结果就是最终值。


![image-20241118115507962](https://my-pic.miaops.sbs/2024/11/image-20241118115507962.png)



> 数据库中哪些地方采用了树结构：
> 
> 
> * B\+树：存储。
> * 树形结构\+中序遍历求值：查询计划，表达式求值。


优化手段：JIT Compilatoin。将热点表达式计算结点视为函数，编译为内联机器码，而不是每次都遍历结点。


![image-20241118120356183](https://my-pic.miaops.sbs/2024/11/image-20241118120356183.png)


