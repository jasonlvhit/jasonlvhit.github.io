---
layout: post
title: 敷衍了事的CUDA实验 —— 用CUDA加速PageRank算法
description: "cuda技术实现并行化pagerank算法"
category: articles
tags: [life, 算法, C++]
---
最近实在是太忙了，可是这个实验又不得不做，组员又没有提供什么帮助，只好顶着头皮硬上，最后只用CUDA做了一点点的优化，却也能稍稍看出些效果，CUDA的加速效果在大数据的应用上的确很出色。  
<br>
PageRank这个算法，在夏天读的一本Collective Programming Intelligence这本书上首先看到，这本书中给出了许多基本的机器学习的算法，但都涉猎不深，唯一的优势是这本书给出了算法详细的Python代码。  

##PageRank

PageRank算法是由Google公司创始人Larry Page院士和Sergey Brin共同提出，算法一Larry Page的姓命名，应用在Google的搜索引擎排名中。  
<br>
PageRank的算法思想很简单，利用互联网网页之间的相互关系，由他们自己推选出重要度高的网页。PageRank算法模拟了一种投票机制，假如A网页有指向B网页的一条链接，那么我们就认为A网页给B网页投了一票；同时我们也认定，重要度高的网页投的票的重要度也高，简单说，我们认为，一个重要网页所指向的网页也相对重要。就像习总如果投了我一票，我当然就很NB的了。  
<br>
为了权衡这种网页之间的重要度排名，PageRank算法给每一个网页一个PR值，PR值越高，网页重要度越高。  
<br>
所以我们也可以很轻松的理解PageRank算法中给我们的这个数理化计算公式： 
<figure>
    <a href="http://jasonlv.cn/images/pagerank.jpg"><img src="http://jasonlv.cn/images/pagerank.jpg"></a>
    <figcaption>PageRank</figcaption>
</figure>  
<br> 
L(v)代表v网页的总链接数，也就是总出度。为什么做这样的处理，其实也很简单，因为所有的网页只能投的票数是相同的，那好吧，大家都只能投一票，所以也就只好用这一票除上这总出度数了。  
<br>
在另一个角度来看一下PageRank，其实PageRank模拟了一种互联网的随机浏览模式。一个很无聊的人，很无聊的从一个随机的网页开始胡乱的浏览，于是他会点击当前网页的链接，跳转到其他的网页，这不就与PageRank的投票思想不约而同了嘛。另一方面，其实在人们的浏览中，会停下来，不继续点击，这里，便有了PageRank算法中的另一个参数：damping factor，阻尼系数。  
<br>
z在这里，d是阻尼系数，d必须是小于1的。这个公式的含义是，一个用户会有d的概率跳转到别的页面，有1-d的概率停滞在当前页面。d的值经过统计，大约为0.85。所以这里我们就得到我们最终使用的PR值计算公式。  
<figure>
    <a href="http://jasonlv.cn/images/damp.jpg"><img src="http://jasonlv.cn/images/damp.jpg"></a>
    <figcaption>PageRank</figcaption>
</figure>  
<br>
这里我引用Programming Collective Intelligence 第4章的例子，计算PR(A):  
<figure>
    <a href="http://jasonlv.cn/images/PRA.jpg"><img src="http://jasonlv.cn/images/PRA.jpg"></a>
    <figcaption></figcaption>
</figure>  
<br>
我们设定d为0.85，用上面的公式有：  
<figure>
    <a href="http://jasonlv.cn/images/CPRA.jpg"><img src="http://jasonlv.cn/images/CPRA.jpg"></a>
    <figcaption></figcaption>
</figure>  
<br>


##开始我们的实验 —— 数据准备

怎样把互联网中网页的关系虚拟化为计算机可计算的形式呢，对于PageRank，这其实很简单，我们很容易想到把互联网中的每一个节点的关系虚拟为一个有向图。所以PageRank算法，也就变成了求解有向图中节点重要度的算法。  
<br>
所以我们写的第一部分代码，是一个随机生成有向图的脚本  
<br>
{% highlight py %}
#Jason Lyu 2013 .11 .30
 
import sys
import random
 
def generate_graph(vertex_number):
    """Generate a Graph.
       which will use in CUDA PageRank test.
       the generate file of Graph will be like this :
 
       0 1
       0 2
       0 4
       1 0
       1 2
       1 3
       2 0
       ...
       ..
       .
       9999 2
       ...
       ..
       .
 
       the first number of each line represent the vertex of source, 
       and the second number represent the 
       source's out-degree_vertex.
 
    """
 
    
    with open("Graph.txt", mode = 'w', encoding = "utf-8") as graph_file:
        i = 0
        while(i < vertex_number):
            number_of_out_degree = random.randint(0, vertex_number/2)
            #print(number_of_out_degree)
            list_of_out_degree = []
            start = 0
 
            while(start < number_of_out_degree):
                out_degree = random.randint(0, vertex_number)
                if out_degree == i or out_degree in list_of_out_degree:
                    continue
                else:
                    start += 1
                    list_of_out_degree.append(out_degree)
 
            list_of_out_degree.sort()
            
            for out_degree in list_of_out_degree:
                graph_file.write(str(i) + ' ' + str(out_degree) + '\n')
                print(str(i) + ' ' + str(out_degree))
                        
            i += 1
 
 
if __name__ == '__main__':
 
    vertex_number = sys.argv[1]
    #print(int(vertex_number))
    generate_graph(int(vertex_number))
 
{% endhighlight %}
            
这个脚本会生成一个Graph.txt文件，这其中有每一条边的信息  
<br>
在整个程序的开始，我们会从这个文件读取所有的节点信息，并建立一个用矩阵表示的有向图。所以如果我们有5000个节点，那么我们就会建立一个5000*5000的邻接矩阵。  

##计算每个节点的PR值

在真正计算PageRank值之前，还有几个没有解决的问题：  
<br>
在我们的PR值计算公式中，我们可以看到，我们利用了已知的PR值计算新的PR值，所以PR值需要一个初始化，以开始第一代计算。  
<br>
OK,我们知道如何计算PR值，那么我们可以预见，在初始化PR值之后，每一次的PR值计算都会产生一个新的PR值结果，因为所关联的PR值在计算中是不断变化的。所以PageRank的计算是一个迭代过程，数学证明可得到结论：PR值的差异最后会收敛，也就是在足够次数的迭代之后，PR值会趋于稳定。  
<br>
所以首先我们需要一个迭代终止条件,当两次迭代的PR值不变，退出迭代
{% highlight cpp %}
#define END_WEIGHT 1e-7
 
//END condition: when the PR value stable
bool END(float a[], float b[])
{
    float sum = 0;
    for (int i = 0; i < numberOfVertex ; ++i)
    {
        sum += abs(a[i] - b[i]);
    }
    cout << sum << endl;
    if (sum < END_WEIGHT)
    {
        return true;
    }
 
    return false;
}
{% endhighlight %}
            
接下来的故事变得简单，计算每个节点的出度和，之后计算每个计算的PR值，当终止条件达成，终止迭代：

{% highlight cpp %}

#define numberOfVertex  500
#define Max_Iteration_Number 10000
#define Alpha 0.85
 
#define InitPageRankValue 6
 
void PageRank(float *Graph, float PR[])
{
    //Display the Graph:
/*
    for (int i = 0; i < vertex; ++i)
    {
        for (int j = 0; j < vertex; ++j)
        {
            printf("%f\t", *(Graph +i*vertex +j));
        }
        printf("\n");
    }
*/
    //Calculate the sum of out-degree of every vertex
    //eg. the sum of every line
    clock_t begin, end;
    float PR_Temp[numberOfVertex ];
 
    begin = clock();
    int iter = 0;  //迭代次数
    for (int m = 0; m < Max_Iteration_Number; ++m)
    {
        iter++;
        float sumOfOutDegree[numberOfVertex ];
        for (int i = 0; i < numberOfVertex ; ++i)
        {
            sumOfOutDegree[i] = 0.0;
        }
 
        //Calculate the sum of degree of each vertex
        for (int i = 0; i < numberOfVertex ; ++i)
        {
            float sum = 0;
            for (int j = 0; j < numberOfVertex ; ++j)
            {
                sum += *(Graph +i*numberOfVertex  +j);
            }
            sumOfOutDegree[i] = sum;
        }
 
        //Calculate the PR value of every vertex.
        for (int i = 0; i < numberOfVertex ; ++i)
        {
            float sum = 0;
            int k = 0;
            for (int j = i; j < numberOfVertex *numberOfVertex  ; 
                                                    j += numberOfVertex )
            {
                if (*(Graph + j) == 1)
                {
                    if(sumOfOutDegree[k] != 0)
                        sum += PR[k] / sumOfOutDegree[k];
                }
                k++;
                //printf("%f\n", sum);
            }
            PR_Temp[i] = (1 - Alpha)  + Alpha*(sum);
        }
 
        if (END(PR_Temp, PR))
        {
            break;
        }
        else{
            for (int i = 0; i < numberOfVertex ; ++i)
            {
                PR[i] = PR_Temp[i];
            }
        }
 
    }
    end = clock();
    printf("Calculate %d iteration of PageRank value cost us:%d ms.\n",
                                                        iter, end - begin);
 
}

{% endhighlight %}           
这个函数中有计时函数，使用Windows计时，需要引入头文件 windows.h

##用CUDA稍作优化

CUDA优化的思想很简单，在CPU上，我们使用1个线程，串行的计算每个节点的PR值，在上面的代码中，我们也能看到，这个串行算法的时间复杂度为O(n^2)。使用CUDA，我们可以使用与节点数量相同的线程，用一个线程计算一个节点的PR值。在理想情况下，我们可以预见这是一个非常强的优化策略。  
<br>
所以CUDA的函数分为两部分，首先在单个线程上计算单个节点的出度数

{% highlight cpp %}
//Calculate Sum of out_degree of each vertex.
__global__ void claculateSumOfOutDegree(float * sumOfOutDegree,
                                             const float* Graph)
{
    int i = blockDim.x * blockIdx.x + threadIdx.x;
 
    if (i < numberOfVertex )
    {
        sumOfOutDegree[i]  = 0;
        for (int j = 0; j < numberOfVertex ; ++j)
        {
            sumOfOutDegree[i] += *(Graph +i*numberOfVertex  +j);
        }
    }
 
}
{% endhighlight %}           
然后，在单个线程上，计算单个节点的PR值：

{% highlight cpp %}
//PageRank value calculate function
 
__global__ void PRAdd(float *PR, const float* Graph, 
                            const float * sumOfOutDegree)
{
    int i = blockDim.x * blockIdx.x + threadIdx.x;
 
    if (i < numberOfVertex )
    {
        float sum = 0.0;
        int k = 0;
        for (int j = i; j < numberOfVertex *numberOfVertex  ;
                                         j += numberOfVertex )
        {
            if (*(Graph + j) && sumOfOutDegree[k])
            {
                sum += PR[k] / sumOfOutDegree[k];
 
            }
            k++;
            //printf("%f\n", sum);
        } 
        PR[i] = Alpha  + (1 - Alpha)*sum;
    }
}
            
{% endhighlight %}           
##Where amazing happens.

如我们所想，CUDA的确可以获得更好的性能表现，在保证结果正确的情况下，CUDA可以给我们如下的实验结果：

<figure>
    <a href="http://jasonlv.cn/images/500.JPG"><img src="http://jasonlv.cn/images/500.JPG"></a>
    <figcaption>500 * 500 matrix</figcaption>
</figure>  
<br>

我们可以得到这样的统计数据：
<figure>
    <a href="http://jasonlv.cn/images/77.jpg"><img src="http://jasonlv.cn/images/77.jpg"></a>
    <figcaption></figcaption>
</figure>  
<br>

在数据量越大的情况下，CUDA的表现越出色。但这也同时要求你有一个强大的存储器。  
<br>
还是希望学弟碰到这个实验能够全身心投入，做的出色。  
<br>
一些参考资料：  
<br>
Wiki: [PageRank](http://en.wikipedia.org/wiki/PageRank)  
<br>
[Parallel PageRank computation using GPUs](http://dl.acm.org/citation.cfm?id=2350751)