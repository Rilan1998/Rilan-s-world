adlist.h(c) - A generic doubly linked list implementation
adlist是一个存储双向链表的数据结构文件。
链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。
作为一种常用数据结构，链表内置在很多高级的编程语言里面，因为Redis使用的C语言没有内置这种数据结构，所以Redis构建了自己的链表实现。
链表在Redis中的应用非常广泛，比如列表键的底层实现之一就是链表。
当一个列表键包含了数量比较多的元素，有或者列表中好汉的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现。
使用到链表的功能有列表键，发布于订阅，慢查询，监视器等，服务器本身还使用链表来保存多个客户端的状态信息，以及使用了链表来构建客户端输出缓冲区。
这里分析的版本是redis3.0，具体版本可在
https://github.com/antirez/redis/tree/3.0
下载


逐句分析：

#ifndef __ADLIST_H__
#define __ADLIST_H__
这两句和头文件里的最后一句
#endif /* __ADLIST_H__ */
共同起到包含adlist.h并避免重复包含的作用。



一，数据结构部分：
/* Node, List, and Iterator are the only data structures used currently. */
意思是listNode、list和listIter是当前使用的仅有三种数据结构。


typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;  //通用实现，可以存放任意类型的数据
} listNode;
链表节点ListNode。
节点存储了三个指针变量：
分别用于指向前一节点，下一节点，指向自身储存的数据。


// list迭代器
typedef struct listIter {
    listNode *next;
    int direction;  //用于控制链表遍历的方向
} listIter;
访问链表的迭代器listIter。
存储了一个指针变量，一个int型变量：
*next用于指向链表中的某个节点，
direction表示迭代器访问的方向，与该变量匹配的有两个宏在h文件末尾，是
#define AL_START_HEAD 0
表示从头结点到尾节点的正向迭代，
#define AL_START_TAIL 1
表示从尾节点到头结点的逆向迭代。


// list数据结构
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);  //用于复制ptr值，实现深度复制
    void (*free)(void *ptr);  //释放对应类型结构的内存
    int (*match)(void *ptr, void *key);  //自定义匹配key
    unsigned int len;
} list;
链表结构list。
提供了
1，*head，*tail两个节点指针分别指向链表的头部和尾部。
2，*(*dup)，(*free)， (*match)三个函数指针。
*(*dup)用于复制链表中节点的值，
(*free)用于释放链表中节点的值，
(*match)用于匹配链表中节点的值，对比链表节点所保存的值和另一个输入值是否相等。
3，无符号整数变量len表示链表的长度。




二，函数声明与实现：
list *listCreate(void);
* Create a new list. The created list can be freed with
 * AlFreeList(), but private value of every node need to be freed
 * by the user before to call AlFreeList().
 *
 * On error, NULL is returned. Otherwise the pointer to the new list. */


list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
新建一个链表。
如果分配内存失败，返回空值；
如果分配内存成功：依次分配list各项的值。
返回链表。
这里面用到了zmalloc这个函数，这个函数包含于zmalloc.c这个文件中，是redis自己实现的函数。回头再发这个文件的分析。


void listRelease(list *list);
/* Free the whole list.
 *
 * This function can't fail. */
void listRelease(list *list)
{
    unsigned int len;
    listNode *current, *next;

    current = list->head;
    len = list->len;
    while(len--) {
        next = current->next;
        if (list->free) list->free(current->value);//释放当前value占用的内存
        zfree(current);//释放该节点结构体占用的内存空间
        current = next;
    }
    zfree(list);
}
链表释放。
从链表头部开始递归至尾部，逐个释放节点。
值得注意的是 
if (list->free) list->free(current->value);
这一句，关于函数指针的使用，我暂时没弄清楚list->free这个函数指针在哪里被赋的值。


list *listAddNodeHead(list *list, void *value);
/* Add a new node to the list, to head, contaning the specified 'value'
 * pointer as value.
 *
 * On error, NULL is returned and no operation is performed (i.e. the
 * list remains unaltered).
 * On success the 'list' pointer you pass to the function is returned. */
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++;
    return list;
}
向链表头插入一个新节点，值为value。
如果分配内存失败，返回NULL；
如果分配内存成功：
    如果原链表为空链表，则头节点和尾节点都为新节点（原先都为NULL），
        新节点的前一节点和下一节点都为空。
    如果原链表不是空链表，则新节点前一节点为原链表尾节点，下一节点为NULL，
        原尾节点的下一节点为新节点，将新节点赋值给尾节点。
    链表长度加一，返回新链表。


list *listAddNodeTail(list *list, void *value);
/* Add a new node to the list, to tail, contaning the specified 'value'
 * pointer as value.
 *
 * On error, NULL is returned and no operation is performed (i.e. the
 * list remains unaltered).
 * On success the 'list' pointer you pass to the function is returned. */
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}
向链表末尾插入一个新节点，值为value。
如果分配内存失败，返回NULL；
如果分配内存成功：
    新节点赋值为value。
    如果原链表为空链表，则头节点和尾节点都为新节点（原先都为NULL），
        新节点的前一节点和下一节点都为空。
    如果原链表不是空链表，则新节点前一节点为原链表尾节点，下一节点为NULL，
        原尾节点的下一节点为新节点，将新节点赋值给尾节点。
    链表长度加一，返回新链表。


list *listInsertNode(list *list, listNode *old_node, void *value, int after);

list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;//分配内存
    node->value = value;
    if (after) { 
        node->prev = old_node;//设置当前节点的前置节点为某个节点
        node->next = old_node->next;//设置当前节点的后置节点为某个节点的后置节点
        if (list->tail == old_node) {
            list->tail = node; //如果old_node是链表尾部，则更新尾部
        }
    } else {//表示插入到某个节点之前
        node->next = old_node; //设置当前节点的后置节点为某个节点
        node->prev = old_node->prev;//设置当前节点的前置节点为某个节点的前置节点
        if (list->head == old_node) {
            list->head = node;//如果某个节点原本为链表头节点，更新链表头结点
        }
    }
    if (node->prev != NULL) {
        node->prev->next = node;//如果当前链表的前置节点不为空，则设置当前节点的前置节点的后置节点为当前节点
    }
    if (node->next != NULL) {
        node->next->prev = node;//如果当前链表的后置节点不为空，则设置当前节点的后置节点的前置节点为当前节点
    }
    list->len++;
    return list;
}
在链表list中插入节点在指定节点old_node的前或后（取决于after的值，若after为NULL，则插在old_node前，反之插在old_node后），值为value。
如果分配内存失败，返回NULL；
如果分配内存成功：
    如果是插入在指定节点后：
        设置新节点的前置节点为指定节点；
        设置新节点的后置节点为指定节点的后置节点；
        如果指定节点是尾节点；
            将尾节点指针指向新节点；
    如果是插入在指定节点前：
        设置新节点的后置节点为指定节点；
        设置新节点的前置节点为制定节点的前置节点；
        如果指定节点为头结点；
            将头结点指针指向新节点；
    如果新节点的前置节点不为空；
        将其后置节点设为新节点；
    如果新节点的后置节点不为空；
        将其前置节点设为新节点；
    链表长度加一，返回新链表。


void listDelNode(list *list, listNode *node);
/* Remove the specified node from the specified list.
 * It's up to the caller to free the private value of the node.
 *
 * This function can't fail. */
void listDelNode(list *list, listNode *node)
{
    if (node->prev)
        node->prev->next = node->next;
    else
        list->head = node->next;
    if (node->next)
        node->next->prev = node->prev;
    else
        list->tail = node->prev;
    if (list->free) list->free(node->value);
    zfree(node);
    list->len--;
}
从链表中删除给定节点。
如果节点有前置节点；
    将其前置节点的后置节点改为被删除节点的后置节点；
否则；
    将头结点指针指向被删除节点的后置节点；
如果节点有后置节点；
    将后置节点的前置节点改为被删除节点的前置节点；
否则；
    将尾节点指针指向被删除节点的前置节点；
释放当前value占用的内存；
释放该节点结构体占用的内存空间；


listIter *listGetIterator(list *list, int direction);
/* Returns a list iterator 'iter'. After the initialization every
 * call to listNext() will return the next element of the list.
 *返回列表迭代器“iter”。初始化之后，对listNext（）的每个调用都将返回列表的下一个元素。
 * This function can't fail. */
listIter *listGetIterator(list *list, int direction)
{
    listIter *iter;

    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
    if (direction == AL_START_HEAD)
        iter->next = list->head;
    else
        iter->next = list->tail;
    iter->direction = direction;
    return iter;
}
为list创建一个迭代器iterator。
申请内存；
如果迭代方向为正；
    将迭代器指向头结点；
如果迭代方向为逆；
    迭代器指向尾节点；
返回迭代器iter；
listNode *listNext(listIter *iter);//返回迭代器iter指向的当前节点并更新iter  


/* Return the next element of an iterator.
 * It's valid to remove the currently returned element using
 * listDelNode(), but not to remove other elements.
 *
 * The function returns a pointer to the next element of the list,
 * or NULL if there are no more elements, so the classical usage patter
 * is:
 *
 * iter = listGetIterator(list,<direction>);
 * while ((node = listNext(iter)) != NULL) {
 *     doSomethingWith(listNodeValue(node));
 * }
返回迭代器的下一个元素。
使用listDelNode（）删除当前返回的元素是有效的，但不删除其他元素。
函数返回一个指向列表下一个元素的指针，如果没有其他元素，则返回空值，因此经典用法模式为：
 * iter = listGetIterator(list,<direction>);
 * while ((node = listNext(iter)) != NULL) {
 *     doSome thingWith(listNodeValue(node));
 * }
 * */
listNode *listNext(listIter *iter)
{
    listNode *current = iter->next;

    if (current != NULL) {
        if (iter->direction == AL_START_HEAD)
            iter->next = current->next;
        else
            iter->next = current->prev;
    }
    return current;
}
返回迭代器的下一个元素。
如果迭代器现指的元素非空；
    如果是正向迭代；
        返回节点current为当前节点的后置节点；
    如果是逆向迭代；
        返回节点current为当前节点的前置节点；
返回节点current；


void listReleaseIterator(listIter *iter); //释放iter迭代器
/* Release the iterator memory */
void listReleaseIterator(listIter *iter) {
    zfree(iter);
}


list *listDup(list *orig);//拷贝表头为orig的链表并返回
/* Duplicate the whole list. On out of memory NULL is returned.
 * On success a copy of the original list is returned.
 *
 * The 'Dup' method set with listSetDupMethod() function is used
 * to copy the node value. Otherwise the same pointer value of
 * the original node is used as value of the copied node.
 *
 * The original list both on success or error is never modified. */
list *listDup(list *orig)
{
    list *copy;
    listIter *iter;
    listNode *node;

    if ((copy = listCreate()) == NULL) //创建一个表头
        return NULL;

    //设置新建表头的处理函数
    copy->dup = orig->dup;
    copy->free = orig->free;
    copy->match = orig->match;

     //迭代整个orig的链表，重点关注此部分。

     //为orig定义一个迭代器并设置迭代方向;
    iter = listGetIterator(orig, AL_START_HEAD);

    //迭代器根据迭代方向不停迭代，相当于++it
    while((node = listNext(iter)) != NULL) {
        void *value;

        //复制节点值到新节点
        if (copy->dup) {//如果定义了list结构中的dup指针，则使用该方法拷贝节点值。
            value = copy->dup(node->value);
            if (value == NULL) {
                listRelease(copy);
                listReleaseIterator(iter);
                return NULL;
            }
        } else
            value = node->value;//获得当前node的value值
        if (listAddNodeTail(copy, value) == NULL) { //将node节点尾插到copy表头的链表中
            listRelease(copy);
            listReleaseIterator(iter);
            return NULL;
        }
    }
    listReleaseIterator(iter);//自行释放迭代器
    return copy;//返回拷贝副本
}
链表复制。
创建局部变量新链表copy，迭代器iter，链表节点node；

为复制的链表创建一个表头

复制链表的三个函数指针；

设置迭代器与迭代方向；
开始迭代；
    如果定义有dup函数；
    则调用dup函数进行值复制；
    如果调用dup函数复制后的值为NULL；（个人思考是dup函数出错）
        释放新复制的链表；
        释放迭代器；
        返回空值；
    否则；
        复制节点值value；
    如果将新生成的node节点插入到copy尾部失败；（尾插入失败）
        释放新复制的链表；
        释放迭代器；
        返回空值；（复制失败）
释放迭代器；
返回拷贝副本；


listNode *listSearchKey(list *list, void *key); //在list中查找value为key的节点并返回
/* Search the list for a node matching a given key.
 * The match is performed using the 'match' method
 * set with listSetMatchMethod(). If no 'match' method
 * is set, the 'value' pointer of every node is directly
 * compared with the 'key' pointer.
 *
 * On success the first matching node pointer is returned
 * (search starts from head). If no matching node exists
 * NULL is returned. */
listNode *listSearchKey(list *list, void *key)
{
    listIter *iter;
    listNode *node;

    iter = listGetIterator(list, AL_START_HEAD);
    while((node = listNext(iter)) != NULL) {
        if (list->match) {
            if (list->match(node->value, key)) {
                listReleaseIterator(iter);
                return node;
            }
        } else {
            if (key == node->value) {
                listReleaseIterator(iter);
                return node;
            }
        }
    }
    listReleaseIterator(iter);
    return NULL;
}
查找特定值的节点；
创建迭代器，设置迭代方向为正；
迭代开始：
    如果有匹配函数；
        使用匹配函数进行匹配；
    否则；
        如果节点值与key值匹配；
        释放迭代器；
        返回节点； 
释放迭代器；
返回NULL（没有匹配的项）；


listNode *listIndex(list *list, long index); //返回下标为index的节点地址
/* Return the element at the specified zero-based index
 * where 0 is the head, 1 is the element next to head
 * and so on. Negative integers are used in order to count
 * from the tail, -1 is the last element, -2 the penultimate
 * and so on. If the index is out of range NULL is returned. 
 * 在指定的基于零的索引处返回元素，
 * 其中0是头，1是头旁边的元素，依此类推。
 * 负整数用于从尾部计数，-1是最后一个元素，-2是倒数第二个元素，依此类推。
 * 如果索引超出范围，则返回空值。
 * */
listNode *listIndex(list *list, long index) {
    listNode *n;

    if (index < 0) {
        index = (-index)-1;
        n = list->tail;
        while(index-- && n) n = n->prev;
    } else {
        n = list->head;
        while(index-- && n) n = n->next;
    }
    return n;
}
返回给定下标的节点；
如果下标小于零；
    下标去相反数；
    起始节点指向尾节点，逆向迭代直到匹配；
    （这里没有用到迭代器，若下标超出范围，返回指向的最后一个节点的前置，即返回NULL）
否则；
    起始节点指向头节点，正向迭代直到匹配；
    （若下标超出范围，返回指向的最后一个节点的后置，返回NULL）
返回node；


void listRewindTail(list *list, listIter *li); //将迭代器li重置为list的头结点并且设置为正向迭代
/* Create an iterator in the list private iterator structure */
/*在私有链表结构中创建正向迭代器*/
void listRewind(list *list, listIter *li) {
    li->next = list->head;
    li->direction = AL_START_HEAD;
}


void listRewind(list *list, listIter *li); //将迭代器li重置为list的尾结点并且设置为逆向迭代
/* Create an iterator in the list private iterator structure */
/*在私有链表结构中创建逆向迭代器*/
void listRewindTail(list *list, listIter *li) {
    li->next = list->tail;
    li->direction = AL_START_TAIL;
}


void listRotate(list *list);//将尾节点插到头结点
/* Rotate the list removing the tail node and inserting it to the head. */
/*旋转列表，移除尾部节点并将其插入头部。*/
void listRotate(list *list) {
    listNode *tail = list->tail;

    if (listLength(list) <= 1) return;

    /* Detach current tail */
    list->tail = tail->prev;
    list->tail->next = NULL;
    /* Move it as head */
    list->head->prev = tail;
    tail->prev = NULL;
    tail->next = list->head;
    list->head = tail;
}
将尾节点插到头结点

创建局部变量tail保存尾节点；
如果链表长度小于等于一，直接返回，结束函数；
将尾指针指向尾节点的前置节点；
将现在尾指针指向的节点的后置节点设为NULL；
将头结点的前置节点设为tail；
将tail前置节点设为NULL；
头结点指针指向tail；




三，宏函数部分
#define listLength(l) ((l)->len)                    
//返回链表l长度，即节点数量

#define listFirst(l) ((l)->head)                    
//返回链表l的头结点地址

#define listLast(l) ((l)->tail)                     
//返回链表l的尾结点地址

#define listPrevNode(n) ((n)->prev)                 
//返回节点n的前置节点地址

#define listNextNode(n) ((n)->next)                 
//返回节点n的后置节点地址

#define listNodeValue(n) ((n)->value)               
//返回节点n的节点值

#define listSetDupMethod(l,m) ((l)->dup = (m))      
//设置链表l的复制函数为m方法

#define listSetFreeMethod(l,m) ((l)->free = (m))    
//设置链表l的释放函数为m方法

#define listSetMatchMethod(l,m) ((l)->match = (m))  
//设置链表l的比较函数为m方法

#define listGetDupMethod(l) ((l)->dup)              
//返回链表l的赋值函数

#define listGetFree(l) ((l)->free)                  
//返回链表l的释放函数

#define listGetMatchMethod(l) ((l)->match)          
//返回链表l的比较函数











