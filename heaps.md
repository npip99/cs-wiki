## Let’s consider a problem:

We propose a data structure that allows us to quickly find the minimal element, remove the minimal element, and add an arbitrarily element to the structure, all in O(logn).

### The situation
So: Let’s say, you are running a stock market. At any time, people can post their stock as “for sale”, and list what price they want to sell their stock at, just like as if they were selling oranges, many sellers sell oranges for many prices. And, at any time, an investor might arrive, and want to know what the cheapest stock available is. Obviously, the investor doesn’t care that someone wants to sell a stock for $9 if an $8 stock is available for the same share in the same company. The investor may either pass on the offer, or may buy that stock going at the lowest price.


###  What we have to accomplish
Now, we have to solve the following problem: 10,000 orders happen per second, and there are 1,000,000 stocks from this company being sold at a given moment. And, let’s say there are 10,000 companies each with that many stocks. Don’t worry too much about those specific numbers, the important part, is that’s a lot of stuff happening! We, the stock market, get paid in transaction fees, so we are obligated to fill every single order we possible can to offer the best possible service to investors, and thus allow ourselves to profit the most.


###  Possible solution?
What we could do, is keep every stock along with the seller’s email address and the seller’s sale price, in a long list. Then, every single time an order comes in, we search through the entire list for the lowest price stock, and then sell that stock to that investor by removing it from the list.

###  Well, maybe not
We can try this, but it won’t work. This is very expensive, we can’t search the list of 1 million stocks, every single time for the 10k orders per second, for all 10k companies. This is computationally infeasible. We would only be able to serve very few orders, causing us to lose money, because we aren’t properly filling these orders and collecting the transaction fees that we were instructed to collect.

###  Okay, why not keep it sorted?
Okay, let’s say we pre-sort the list of stocks for sale. Now, it’s easy to sell the cheapest stock, it’s just the bottom of the list. Okay, but now, let’s say stocks range from $8.04 to $20.41 in sale price. And someone says they want to sell a stock for $16.98. Well, that kind of sucks, we need to now shift the entire list up one in order to make room for putting that sale in the middle of the list. This potentially means shifting 500,000 stocks. This is expensive. This will cause our systems to fail and not properly process orders in time.

###  Okay, then what’s the solution?
Let’s make clear the problem:
We need a way to easily add to a list, We need a way to easily check what the smallest item in that list is, and we need a way to easily remove the smallest item from the list. And, that’s all we need to do. If we can do that, then we are done. And we need to do all three of those things, as fast we can.

Let’s learn how to solve that problem.

Tree-based representation

![A Tree](assets/images/1.png)

All these numbers are basically random, but it’s a tree, and if we make the right choices for what numbers to put in the tree, we’ll be able to do powerful things. An important property here is that this tree has 27 nodes, but is only 5 levels high. In general, you can fit a large number of nodes in a few number of levels.

![Now, a Heap:](assets/images/2.png)

This is a tree, but it is also a heap. A heap is a special kind of tree, can you see what’s unique about this tree? Well, it’s that every node, has a smaller number than it’s two child nodes. The fact that every node is smaller than its two child nodes, is called the **heap property**. Does it have the properties that we’re looking for? Well, we can certainly find the smallest node from that list very easily. It’s right there at the top, any by the **heap property**, we can guarantee that the smallest node is always at the very top.

Okay, but can we remove the smallest node easily? We need to do that in-case someone buys the cheapest stock in the list.

![](assets/images/3.png)

Okay, we removed the smallest item. But now what? It’s not a tree any more. Okay, let’s try to make it a tree again:

![](assets/images/4.png)

Okay, it’s a tree. But not’s not a heap anymore, we’ve violated the heap property, because 18 > 10. Well, in that case, let’s just swap 18 with its child.

![](assets/images/5.png)

Now, 10 > 18, so the heap property has now been maintained along the blue connection. However, it remains that 18 > 16 and 18 > 13. So, the heap property is failing. Well, let’s try swapping 18 and 16.

![](assets/images/6.png)

Um, well 16 > 13. That didn’t help. What if we swapped 18 and 13?

![](assets/images/7.png)

Ah okay, we fixed that node. 13 > 16, and 13 > 18. Still, 18 > 17, but that’s an easy fix. 

![](assets/images/8.png)

Bam, we’re done. Now it’s a heap again! This only took 4 operations to move the 18 all the way down to the bottom, even though there were 13 items in the tree. If there were 1023 items in the tree, it would have only been 10 operations, because the tree would only be 10 levels tall. 

![](assets/images/9.png)

The key behind trees, is that a lot of nodes can fit, in a tree that’s only 6 levels high. Look at how many nodes there are. At 20 levels, that’s 1 million nodes. So, we can remove the cheapest stock in our stock market in only 20 operations, instead of 1 million operations. That’s 50,000 times faster. That’s the difference between a 20ms transaction, and a 17 minute transaction. This is why we use heaps!

Okay, what about adding a new stock to the list? Well, it’s basically the same process, but in reverse.

![](assets/images/10.png)

We’re trying to add 5 to the list. We just need to bubble him to the top, swapping him with each parent:

![](assets/images/11.png)

Until it just looks like this:

![](assets/images/12.png)

And bam you have a heap again.

Usage in C++:
```cpp
std::priority_queue<int> q;
for(int n : {1,8,5,6,3,4,0,9,7,2})
        q2.push(n);
std::cout << q.top();
q.pop();
std::cout << q.top();
```

The above code will print “0”, and then “1”. It’s called a priority queue, because it always gives you the nodes in order of priority (Here the highest priority node is considered to be the lowest element, a so-called min-queue). All you have to do to add stocks to the list, and buy the cheapest stock, is call “q.push” and “q.pop”. To know the smallest element in the list, just so that you know the price before buying a stock, all you have to do is “q.top()”. And all of these functions will run in 20 operations or less with 1 million stocks for sale, as opposed to the originally infeasible 1 million operations per order.

Since this is our first data structure, this problems will essentially use heaps and only heaps.

Important Details:
Heaps are best implemented as binary complete trees. Binary, as in all nodes have two children. And complete, as in all rows above the last one must be full, and the last one must be filled from left-to-right.

![](assets/images/13.png)

Complete trees are simpler and easier to use.

### Example problems:

[Little Monk and ABD](https://www.hackerearth.com/practice/data-structures/trees/heapspriority-queues/practice-problems/algorithm/little-monk-and-abd/)

[Little Monk and Williamson](https://www.hackerearth.com/practice/data-structures/trees/heapspriority-queues/practice-problems/algorithm/little-monk-and-williamson/)

### Example challenge:

Code up a data structure / class with the following methods:

* Push
* Pop
* Top

Utilizing a heap as the internal workings, to guarantee that no operation will take very long. In essence, implement a priority queue utilizing a heap.
