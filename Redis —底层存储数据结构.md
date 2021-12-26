# Redis —底层存储数据结构
## 存储结构SDS（简单动态字符串）
> 字符串是Redis中最为常见的数据存储类型，其底层实现是简单动态字符串(simple dynamic string)  
* 二进制安全的数据结构
	C语言会在解析数据的时候忽略 ‘\0’ 后的数据
* 提供内存预分配，避免了频繁的内存分配
```【append】【setbit】
	char buf[] = ‘’……’’ //数据
	len		//字符串的长度
	free	//buf中还有多少使用空间
			//空间换时间，在分配空间时，会预先分配略大的空间
			//成倍扩容（len+addlen）* 2
			//当长度达到1024*1024时（1M），往后扩容每次增加1M
```
```Redis 3.2以后根据数据长度len区分了更多的string类型
len		//占4字节，可表示范围0 ～ 2^32-1
alloc	//分配的空间大小；alloc - len = free
flags	//dr5类型中，前面表示类型，后面表示长度
		//其余的len和alloc在flags前面，flags中前面表示类型，后面闲置
```
* 兼容C语言函数库（数据尾部追加 ‘\0’）
## 存储结构字典
> Redis 的字典使用哈希表作为底层实现  
### 哈希表
1. hash函数将key散列取模，获得数组索引
2. 在数组索引对应的位置存放
3. 使用链表+头插法解决hash碰撞
4. 渐进式rehash：同一个位置太多数据会 *2 扩容。在查询到来时，优先查找旧哈希表内的桶数据，存在则将整个桶迁移至新数组内。不存在则查询新哈希表。有轮训机制确保完成数据的迁移。
### 字典（dictht）
```dictht
typedef struct dictht{
dictEntry **table;		//哈希表数组
unsigned **long** size;	//哈希表大小（2^n）
unsigned **long** sizemask;//哈希表大小掩码，用于计算索引值
							//总是等于 size-1
unsigned **long** used;	//该哈希表已有业务数据的数量
}dictht
// used/size = 1 式进行扩容
//任意数 % 2^n优化 --> 任意数 & (2^n - 1) [变成位运算]
```
```dictEntry
typedef struct dictEntry{
void *key;	//键
union{		//值
void *val; 	//指向了redisObject
uint64_tu64;
int64_ts64;
}v;
struct dictEntry *next;	//指向下一个哈希表节点
							//形成链表,解决hash冲突
}dictEntry
```
```redisObject
type		//约束客户端的命令
encoding	//值的编码方式
lru		//内存淘汰相关
refcount	//引用计数管理内存
ptr		//指针，指向正式的存储位置（SDS）
```
### string的编码方式
1. raw：即为SDS
2. int：长度在整数范围内，若可转化为整型值，ptr不再指向地址而是存储该值。
	节省存储空间；节省IO
3. embstr：缓存行64byte 而redisObject大小为16byte；选用该类型开辟连续空间储存数据，防止为了填充缓冲行多次IO

	
