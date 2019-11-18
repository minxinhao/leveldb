# Skip_list解读

因为需要阅读LSM的相关文章，有提到skip list，对于跳表这个名字非常好奇，所以过来学习一下。
主要参考了[一篇博客](http://www.cppblog.com/mysileng/archive/2013/04/06/199159.html)。这也是第一次写学习笔记，为了锻炼一下自己的学习习惯。
参考的博客原文对skip list基本内容介绍的也很清楚了，这里主要写一下自己在阅读过程中卡壳的地方。

## 1.数据结构定义

    typedef  struct nodeStructure  
    {  
        int key;  
        int value;    
        struct nodeStructure *forward[1];  
    }nodeStructure;  

这里定义的nodeStructure是指skip list中的一列(这一点可以通过定义的createNode函数看出来)，因为每一列的key、value相同，所以实际上不需要重复存储，这是我疑惑的第一个地方。forward指向一列中向下(?向上)的下一个节点。当forward为NULL时代表这一列结束。给出的nodeStructure中只有forward指针，没有指向左或者右的指针，这是我疑惑的第二个地方，不清楚在同一层的链表中进行向右移动(毕竟每一层都没有链表)。

    typedef  struct skiplist  {  
        int level;    
        nodeStructure *header;  
    }skiplist;  

skiplist结构定义了skiplist当前的高度level和第一列header。

### 2.函数实现

对于数据结构定义的疑惑只能通过阅读他实现的功能函数来解决。

#### 1.createSkiplist

    skiplist* createSkiplist()  {  
        skiplist *sl=(skiplist *)malloc(sizeof(skiplist));    
        sl->level=0;    
        sl->header=createNode(MAX_LEVEL-1,0,0);    
        for(int i=0;i<MAX_LEVEL;i++){    
            sl->header->forward[i]=NULL;    
        }  
        return sl;  
    }

初始化声明一个skiplist的时候首先申请一列最大高度的nodeStructure作为skiplist的header。然后每一个header的forward数组置为NULL.这里就合我之前理解的nodeStructure代表一列发生了矛盾。而且forward是一个大小为1的指针数组，这里却进行了越界访问。所以这里只能认为nodeStructure就是代表一层链表继续分析,forward指向的是链表中下一个元素。
而要继续弄清楚这里实现的skiplist结构，我认为只能通过insert、find等功能函数进行分析，通过和理论给出的skiplist find、insert操作对比来分析代码定义的数据结构。

#### 2.insert

给出的算法中，insert首先利用查找功能，找到最底层中比给定key小的最大key的位置。而实现的代码中对应的部分是:

    nodeStructure *update[MAX_LEVEL];  
    nodeStructure *p, *q = NULL;  
    p=sl->header;  
    int k=sl->level;  
    //从最高层往下查找需要插入的位置  
    //填充update  
    for(int i=k-1; i >= 0; i--){  
        while((q=p->forward[i])&&(q->key<key))  
        {  
            p=q;  
        }  
        update[i]=p;  
    } 

这里的k是skiplist的层数，在遍历每一层的过程中，通过while循环查找每一层比给定的key小的最大key的位置。通过这个while循环可以进一步确认这里的forward确实是代表一层链表。用update数组记录下每一层需要插入数据的位置。

>正当我以为这样就可以结束的时候，有出现了一段让我感觉不对劲的代码。

    q=createNode(k,key,value);  
    //逐层更新节点的指针，和普通列表插入一样  
    for(int i=0;i<k;i++)  
    {  
        q->forward[i]=update[i]->forward[i];  
        update[i]->forward[i]=q;  
    }  

显然如果按照forward存放的是节点右边的节点的指针，那么这个更新则完全不正确，然后意识到之前的分析错误了。再次分析createNode和createSkiplist的代码可以知道了forward的用法。

    nodeStructure* createNode(int level,int key,int value) {        
        nodeStructure *ns=(nodeStructure *)malloc(sizeof(nodeStructure)+level*sizeof(nodeStructure*));    
        ns->key=key;    
        ns->value=value;    
        return ns;    
    } 

这里给ns分配空间的时候malloc了level+1个nodeStructure指针大小的空间(nodestructure中包含的)。而createSkiplist的时候传递给createNode的参数是MAX_LEVEL-1。所以分析可以知道，forward数组的第一个元素(forward[0])存放的是该层右边的元素，而从1～MAX_LEVEL-1的元素对应第i层链表的对应位置。
