[Why Systolic Architectures?](https://www.cs.virginia.edu/~smk9u/CS3330S20/kung_-_1982_-_why_systolic_architectures.pdf)

为什么要设计脉动阵列这样的架构？

- Simple and regular design（简单和规则的设计）

​		首先就是成本效益一直是专用系统的关注点，由于一个专用系统的功能往往十分有限，所以就需要它的成本足够低来弥补这一劣势。而其成本又被分为设计成本和器件成本。由于集成电路技术的进步，器件成本正在迅速下降，所以关键因素还是设计成本。所以采用脉动阵列这个简单又规则的硬件架构，可以很快地完成芯片的设计和实现。

- Concurrency and communication（并发和通信）

​		基本上有两种方法可以构建一个快速的计算机系统。一种是使用快速器件，另一种是使用并发。随着器件速度增长放缓，现在计算系统速度的提高大都来自并发使用。在高并发的过程中，协调和通信也变得尤为重要。

- Balancing computation with I/O（平衡计算和I/O）

​		如图所示，图的上半部分是传统计算系统的模型，一个处理单元需要从存储器里读取数据，处理后再写回到存储器。**而数据的存取速度往往大幅度低于数据处理的速度**，这也就导致了整个计算系统的速度受限于存取（I/O）速度。所以脉动架构的思想就是希望如何将这来着不易的数据更加有用起来，实际上就是重复使用数据。如图的下半部分所示，第一个数据读出后首先进入第一个处理单元（PE），处理后传入下一个单元，同时第二个数据读出进入第一个处理单元，以此类推，这样数据就可以在处理单元在多流动一会，这样就可以在较小的内存带宽下实现高计算吞吐量。

​		当然，脉动架构还有很多其他的优势，比如模块化易于扩展，简单和规则的数据和控制流，使用简单和规则的单元，消除了全局广播、扇入和（可能）快速响应时间。

​	<img src="../img/Why Systolic Architectures/Basic principle of a systolic system.png" alt="image-20210605130409034"  />

​		不难看出，脉动阵列结构设计简单，成本低，但灵活性较差，所以只适合于特定运算。作者用了大量的篇幅介绍了脉动阵列实现卷积运算的方法。首先作者给出了卷积运算的定义，如下图所示：![image-20210605134721337](D:\Typora笔记文件夹\斯坦福CS217\the convolution computation.png)

下面作者给出了使用脉动阵列实现卷积运算的几种方法，以下只介绍一种（xi's are broadcast, wi's stay, and yi's move systolically）：

<img src="../img/Why Systolic Architectures/broadcast inputs, move results, weights stay.png" alt="image-20210605135039898"  />

X值广播到各个PE，W值预先存放在PE中并且保持不变，而部分结果Y通过脉动的方式在阵列间向右传递。具体过程就是：

<img src="../img/Why Systolic Architectures/IMG_2948.JPG" alt="IMG_2948" style="zoom: 25%;" />

显然，在第三个时刻就能得到卷积运算的第一个结果，此后依然会不断输出Y的值。文中还给出了其他实现卷积的方式，如*broadcast inputs, move weights, results stay；fan-in results, move inputs, weights stay*。这几种都属于“(Semi-) systolic convolution arrays with global data communication”。另外还有一大类是“(Pure-) systolic convolution arrays without global data communication”，包括*results stay, inputs and weights move in opposite directions；results stay, inputs and weights move in the same direction but at different speeds；weights stay, inputs and results move in opposite direction；weights stay, inputs and results move in the same direction but at different speeds*。

​		接下来介绍脉动阵列如何实现矩阵乘法，主要有两种做法。一种方法就是在脉动阵列中流动的是X值和W值，结果Y保存在每个PE中，[如图所示](https://www.cs.hmc.edu/courses/2001/spring/cs156/html08/slides08.pdf)。

​		<img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_1-163428739074714.png" alt="image-20210605142318178"  />

<img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_2.png"  />

<img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_3.png"  /><img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_4.png" alt="image-20210605142602407"  /><img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_5.png" alt="image-20210605142628373"  /><img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication__6.png" alt="image-20210605142713147"  /><img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_7.png" alt="image-20210605142739445"  /><img src="../img/Why Systolic Architectures/Systolic Matrix Multiplication_8.png" alt="image-20210605142824756"  />

在Harvey Mudd College Spring 2001 [CS156](https://www.cs.hmc.edu/courses/2001/spring/cs156/)中，讲解了如何从另一个角度（将相乘的矩阵的行和列看作两条相互滑过的条带）更改的理解脉动阵列实现矩阵乘法。另一种方法就是在脉动阵列中流动的数据是X值和Y值的中间结果，W值存储在每个PE中，以下不作阐述。需要注意的就是数据进入脉动阵列时需要调整好一定的顺序才能得到准确的矩阵运算结果。

​		总结起来，脉动架构有如下一些特征：（1）阵列由多个同构的PE构成，可以是一维或二维，串行、阵列或树的结构；（2） PE功能相对简单，系统通过实现大量PE并行来提高运算的效率；（3） PE只能向相邻的PE发送数据（在一些二维结构中，可以向对角线发送数据），数据采用流水线的方式向“下游”流动，直到流出最后的PE。