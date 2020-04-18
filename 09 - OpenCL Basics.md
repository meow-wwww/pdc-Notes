# PDC09 - OpenCL Basics

OpenCL本质提供了一个host和不同devices之间的接口（interface）

## 为什么学OpenCL？和CUDA的对比

cuda：nvidia自己推的，同样代码用cuda写一般性能更高(nvidia对于open CL的实现没有cuda好)，写GPU应该选择cuda
openCL : 出现更晚，没有cuda流行，更复杂，只是授课用，对异构概念更清晰(Cuda掩盖了很多)，可以写multithread也可以写GPU  

## OpenCL特点简述

OpenCL旨在实现异构设备上的portable并行计算。portable现实目前不可能，但这个framework在功能上是可以做到的（如写一个CPU多核代码，在GPU上执行）

## OpenCL概念与术语

注：OpenCL都是抽象的概念，和物理硬件并无一一对应关系

1. **host**：(通用的平台，跑起串行通用程序)。通用计算，资源的协调。哪里跑library哪里就是host，一般是x86 cpu
2. **device**: 执行kernel code（特殊写出来的，需要被并行的代码）（物理上host 和device可以是一个）可以通过library操作的计算设备。
   OpenCL支持不同device，只要vendor支持了opencl的standard的compiler和runtime即可
3. **runtime API** : **loads** and **excutes** kernels across devices。host和device之间通过runtime API实现信息交互(和message passing有点像，不用share address space）对device的好处：不用支持share address space, 把这个代价expose给了程序员。
4. **context**: 资源管理上的辅助概念。（可以同时有多个context，比如当一个主板有多个PCIE插槽，插不同显卡）就是把device和Host连起来的环境，可以创建多个context
5. **program**: device code。分为source code和binary code。source code和bin都管理在program。OpenCL用区分source和bin的方式实现portable，bin可以编译成不同device上的
6. **kernel**：可以被host端定义，但在device端执行的函数。和纯粹在device上执行不一样。
7. **memory objects**: 分别buffer和images。buffer：最常用。images: GPU上硬件优化过的东西，因为常用而被OpenCL支持，性质和buffer不太一样,memory layout不一样。mem obj对于host来说是一个可以操纵的把手，因为device上的硬件不能直接访问。host通过memory object，在device上面申请内存（可能在未来某个时刻）
8. **command queue**: 做synchronization的工具，可以做cpu和gpu之间的，也可以做不同kernel之间的synchronization的规则。最主要是host 和 device的memory和program counter的控制（没有openCL的话，所有设备都要读PCIE的标准，影响生产力，因此提出用OpenCL对PCIE进行包装）。  用来给device传指令。分为三种：
   1. memory commands(data transfers)
   2. kernel execution
   3. synchronization

## SIMD讨论

![1587186524580](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587186524580.png)

多个计算单元（ALU/PE）用不同data执行同一条指令。定义计算单元数量为width。

常见例子：GPU、x86 SSE

好处：减少control flow和instr hardware数量。



## OpenCL Platform Model

![1587186098599](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587186098599.png)

上图中，线条上面的是OpenCL部分，下面是device部分。

每个device有自己的compiler，opencl其实是定义了device和host之间的。
各个硬件自己把source变成bin。compiler是每个device的厂商自己定义的!

![1587187063568](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587187063568.png)

模型定义：一个host有多个device，每个device 有多个compute unit，每个unit有多个PE ，每个PE有自己的PC

host: 哪里跑library哪里就是host，一般是x86 cpu
device：可以通过library操作的计算设备

context：就是把device和Host连起来的环境，可以创建多个context

command queue: 来给device传指令

memory object: 对于host来说是一个可以操纵的把手，因为device上的硬件不能直接访问。host通过memory object，在device上面申请内存（可能在未来某个时刻）

kernel arguments:  argument有两种:
1. 基本类型 int  bool
2. 一个mem obj，让kernel code可以每次在device上面找数据，而不是到host上取数据

executing the kernel:  运行一个kernel时要specify的内容有： kernel, arg, index，表达完以后，device就清楚自己要做什么了。然后使用command queue ,告诉device开始执行



## OpenCL流程（略讲）

![1587194033079](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587194033079.png)

0. 初始化device。声明context, 选择一个context，通过clCreateCommandQueue声明一个command queue
1. create buffers。在device端建立buffer，把input data通过clEnqueueWriteBuffer传到device端
2. build program, select kernel。在device端compile program和kernel
3. set arguments, enqueue kernel。通过clSetKernelArg设置kernel, arg, index; 通过clEnqueueNDRangeKernel执行kernel
4. read back result。用clEnqueueReadBuffer把需要的数据读回host。

## 编程模型

- data prallel：利用不同work items实现。work item和memory obj之间的一一映射。
- Task parallel: 创建多个kernel，每个kernel运行在独立的index space。主要依赖于device怎么处理kernel间的并行
- 同步： work items间有限同步

## work-item模型

![1587196377122](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587196377122.png)

## memory模型

![1587196478653](C:\Users\Thinkpad\AppData\Roaming\Typora\typora-user-images\1587196478653.png)

