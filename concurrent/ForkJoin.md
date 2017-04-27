
Fork Join的思路是“分治法”，将划分之后的任务分派给不同的计算资源，可以并行的完成任务。当计算分别完成之后，最后再合并回来。
简单来看，就是一个递归的分解和合并，直到任务小到可以接受的程度。

“工作窃取”，也就是设计论文原文中提到的 Work Stealing。

队列机制，分解的任务应该用支持Work Stealing的双端队列维护起来。
对于子任务的分解，可以从后端取出分解再放入，而对于WorkStealing则可以从头部取出，放入其他队列的尾部。

JDK1.7中已经给出了Fork Join的实现。
在Java SE 7的API中，有以下5个类：
- ForkJoinPool 实现ForkJoin的线程池
- ForkJoinWorkerThread  实现ForkJoin的线程
- ForkJoinTask<V> 一个描述ForkJoin的抽象类
- RecursiveAction 无返回结果的ForkJoinTask实现
- RecursiveTask<V> 有返回结果的ForkJoinTask实现

http://www.molotang.com/articles/694.html

http://www.molotang.com/articles/696.html

http://www.molotang.com/articles/706.html