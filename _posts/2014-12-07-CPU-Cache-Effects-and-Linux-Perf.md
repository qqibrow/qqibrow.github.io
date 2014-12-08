---
layout: post
title: CPU Cache Effects and Linux Perf
---

CPU cache is a small and fast memory unit that CPU access. Understanding how it works and how to use it is crucial for high performance programming. 

One good startpoint to learn CPU cache is ["gallery of processor cache effects”](http://igoro.com/archive/gallery-of-processor-cache-effects/) from Igor Ostrovsky, in which some code samples and analysis graph are used to show how CPU cache could affect the program. There is a Chinese translation version of it from the famous coolshell. After reading it, you will have a clear picture how it works and could answer most of relative interview questions correctly :)

Since I am studying system performance analysis recently, I think it’s best that I could get metrics about CPU cache and show some comparison results. Perf is the best tool to do this. 

[Perf](http://en.wikipedia.org/wiki/Perf_%28Linux%29) is a performance analyzing tool in Linux which collect both kernel and userspace events and provide some nice metrics. It’s been widely used in my team to find bottomneck of CPU-bound applications. 

Let’s see how to use perf to show CPU cache effects. Below is a simple program which modified based on Example 6 from [Igor’s post](http://igoro.com/archive/gallery-of-processor-cache-effects/).

{% highlight c++ %}
#include <cstdio>
#include <cstdlib>
#include <chrono>
#include <thread>
#include <vector>
#include <assert.h>

using namespace std;
using namespace std::chrono;

#define LENGTH  (1024 * 1024 * 32)

void UpdateAtPosition(int* position) {
    for(int i = 0; i < LENGTH; ++i) {
        *position = *position + 3;
    }
}
int main(int argc, char * argv[])
{
    if(argc < 3) {
        printf("Too few arguments\n");
        return -1;
    }
    int* arr = new int[LENGTH];

    auto t0 = high_resolution_clock::now();

    int num_thread = argc - 1;
    std::vector<thread> threads;
    for(int i = 0; i < num_thread; ++i) {
        const int pos = atoi(argv[1 + i]);
        assert(pos >= 0 && pos < LENGTH && "Pass in position must be in  the range");
        threads.push_back(thread(UpdateAtPosition, arr + pos));
    }

    for(auto& th: threads) {
        th.join();
    }

    auto t1 = high_resolution_clock::now();
    nanoseconds total_nanoseconds = std::chrono::duration_cast<nanoseconds>(t1 - t0);
    printf("Time exlipse %ld\n", total_nanoseconds.count());
    delete[] arr;
    return 0;
}
{% endhighlight %}

The logic is pretty simple. Just create sever thread and each of the thead constantly operator one one position of the array. The positions are passed-in parameteres. The questions is: Is the excution time related to the pass in visiting positions? Here are some results.

{% highlight bash %}
perf stat -d ./cache_line_test 0 1 2 3
Time eclipse 324802703 ns
perf stat -d ./cache_line_test 16 32 48 64
Time eclipse 195028694 ns
{% endhighlight %}
Woo, The result shows that visiting 4 continuous elements is almost 1 second slower than visiting 4 far-away elements. If you read the post above, you will understand the reason is that in the first case elements on the same cache line are operated constantly, causing false sharing and therefore degrade the performance. It’s the same as visiting resources protected by a global lock, according to [Herb Sutter’s great explanation](http://www.drdobbs.com/parallel/maximize-locality-minimize-contention/208200273?pgno=1)

Well, Let’s use perf to see what really happened. Here we use ``perf stat`` command. 

{% highlight bash %}
perf stat -d ./cache_line_test 0 1 2 3
Time exlipse 324802703

 Performance counter stats for './cache_line_test 0 1 2 3':

       1288.050638 task-clock                #    3.930 CPUs utilized
               185 context-switches          #    0.144 K/sec
                 8 cpu-migrations            #    0.006 K/sec
               395 page-faults               #    0.307 K/sec
     3,182,411,312 cycles                    #    2.471 GHz                     [39.95%]
     2,720,300,251 stalled-cycles-frontend   #   85.48% frontend cycles idle    [40.28%]
       764,587,902 stalled-cycles-backend    #   24.03% backend  cycles idle    [40.43%]
     1,040,828,706 instructions              #    0.33  insns per cycle
                                             #    2.61  stalled cycles per insn [51.33%]
       130,948,681 branches                  #  101.664 M/sec                   [51.48%]
            20,721 branch-misses             #    0.02% of all branches         [50.65%]
       652,263,290 L1-dcache-loads           #  506.396 M/sec                   [51.24%]
        10,055,747 L1-dcache-load-misses     #    1.54% of all L1-dcache hits   [51.24%]
         4,846,815 LLC-loads                 #    3.763 M/sec                   [40.18%]
               301 LLC-load-misses           #    0.01% of all LL-cache hits    [39.58%]

0.327715621 seconds time elapsed


perf stat -d ./cache_line_test 16 32 48 64
Time exlipse 195028694

 Performance counter stats for './cache_line_test 16 32 48 64':

        744.442001 task-clock                #    3.761 CPUs utilized
               112 context-switches          #    0.150 K/sec
                 7 cpu-migrations            #    0.009 K/sec
               395 page-faults               #    0.531 K/sec
     1,714,456,744 cycles                    #    2.303 GHz                     [38.96%]
     1,274,421,957 stalled-cycles-frontend   #   74.33% frontend cycles idle    [41.60%]
        76,097,831 stalled-cycles-backend    #    4.44% backend  cycles idle    [44.15%]
       987,253,691 instructions              #    0.58  insns per cycle
                                             #    1.29  stalled cycles per insn [54.34%]
       126,220,377 branches                  #  169.550 M/sec                   [55.34%]
            13,691 branch-misses             #    0.01% of all branches         [54.22%]
       677,471,016 L1-dcache-loads           #  910.039 M/sec                   [52.62%]
            36,992 L1-dcache-load-misses     #    0.01% of all L1-dcache hits   [50.51%]
             7,896 LLC-loads                 #    0.011 M/sec                   [37.75%]
               503 LLC-load-misses           #    6.37% of all LL-cache hits    [35.60%]

       0.197919841 seconds time elapsed
{% endhighlight %}

It’s pretty clear, isn’t it? You can see actually the second program reduce the cache miss rate from 1.54% to 0.01% and as as result dramatically reduce LLC-loads(which is the last level CPU cache) number by 99.9%, from 4,846,815 to 7,896. Perf tells us everything. Great!

Here is another simple program to show the performance gain using cache line padding technique. 

{% highlight C++ %}



{% endhighlight %}

program may load x and y together into CPU since they may in the same cache line. However, two separate threads will constantly visit x and y, causing contention issues. If you manually specify additional padding for both variables in the code, you can avoid this false sharing effect, which shows clear in the perf stat reports.


Time exlipse 137767393

Performance counter stats for './cache_line_test2':

236.315031 task-clock # 1.684 CPUs utilized
27 context-switches # 0.114 K/sec
3 cpu-migrations # 0.013 K/sec
361 page-faults # 0.002 M/sec
498,424,764 cycles # 2.109 GHz [42.32%]
326,001,170 stalled-cycles-frontend # 65.41% frontend cycles idle [44.44%]
104,706,411 stalled-cycles-backend # 21.01% backend cycles idle [46.42%]
387,431,049 instructions # 0.78 insns per cycle
# 0.84 stalled cycles per insn [56.90%]
66,160,017 branches # 279.965 M/sec [57.73%]
9,012 branch-misses # 0.01% of all branches [54.35%]
277,850,622 L1-dcache-loads # 1175.764 M/sec [50.99%]
20,624 L1-dcache-load-misses # 0.01% of all L1-dcache hits [47.61%]
2,911 LLC-loads # 0.012 M/sec [34.05%]
645 LLC-load-misses # 22.16% of all LL-cache hits [40.02%]

0.140328188 seconds time elapsed

Time exlipse 148839896

Performance counter stats for './cache_line_test2':

290.209878 task-clock # 1.917 CPUs utilized
32 context-switches # 0.110 K/sec
1 cpu-migrations # 0.003 K/sec
360 page-faults # 0.001 M/sec
658,944,383 cycles # 2.271 GHz [38.34%]
434,914,294 stalled-cycles-frontend # 66.00% frontend cycles idle [40.55%]
167,556,509 stalled-cycles-backend # 25.43% backend cycles idle [45.42%]
355,704,318 instructions # 0.54 insns per cycle
# 1.22 stalled cycles per insn [55.85%]
61,630,750 branches # 212.366 M/sec [57.10%]
9,637 branch-misses # 0.02% of all branches [55.29%]
269,576,295 L1-dcache-loads # 928.901 M/sec [53.15%]
4,372,094 L1-dcache-load-misses # 1.62% of all L1-dcache hits [50.49%]
2,223,770 LLC-loads # 7.663 M/sec [36.74%]
938 LLC-load-misses # 0.04% of all LL-cache hits [34.65%]

0.151364416 seconds time elapsed

In summary, understanding how CPU cache works is pretty helpful to write high performance C++ code. At the same time, perf stat could help us to verify our cache line padding techniques does work.  You can find all the sample code here. And for more details about perf, please refer to brendangregg’s awesome post.
