# Informer LocalStore源码解析


 按照惯例，先上图，cache对应Informer中的Local Store。

![img](./informer.jpeg)

直接撸源码（基于1.12版本，有删减，不影响主要逻辑）

```go
// 代码源自client-go/tools/cache/store.go
type Store interface {
   Add(obj interface{}) error
   Update(obj interface{}) error
   Delete(obj interface{}) error
   List() []interface{}
   ListKeys() []string
   Get(obj interface{}) (item interface{}, exists bool, err error)
   GetByKey(key string) (item interface{}, exists bool, err error)
 
   // Replace will delete the contents of the store, using instead the
   // given list. Store takes ownership of the list, you should not reference
   // it after calling this function.
   // 用传入的内容替换原有内容
   Replace([]interface{}, string) error
   Resync() error
}
```

Store为最基本的存储接口，提供增删改查基本功能，要求对象有唯一键，键的计算方式由接口的具体实现决定，很好理解。

```go
// 代码源自client-go/tools/cache/index.go
// Indexer is a storage interface that lets you list objects using multiple indexing functions
type Indexer interface {
   Store
   // Retrieve list of objects that match on the named indexing function
   Index(indexName string, obj interface{}) ([]interface{}, error)
   // IndexKeys returns the set of keys that match on the named indexing function.
   IndexKeys(indexName, indexKey string) ([]string, error)
   // ListIndexFuncValues returns the list of generated values of an Index func
   ListIndexFuncValues(indexName string) []string
   // ByIndex lists object that match on the named indexing function with the exact key
   ByIndex(indexNtame, indexKey string) ([]interface{}, error)
   // GetIndexer return the indexers
   GetIndexers() Indexers
 
   // AddIndexers adds more indexers to this store.  If you call this after you already have data
   // in the store, the results are undefined.
   AddIndexers(newIndexers Indexers) error
}
```

Indexer在Store基础上扩展了索引能力，就好比给数据库添加的索引，以便查询更快，那么肯定需要有个结构来保存索引

```go
// 代码源自client-go/tools/cache/index.go
// IndexFunc knows how to provide an indexed value for an object.
type IndexFunc func(obj interface{}) ([]string, error)
  
// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String      //sets.String为map[string]struct{}类型，减少内存占用
 
// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc
 
// Indices maps a name to an Index
type Indices map[string]Index
```

到这里是不是一脸懵逼？？！！，光看注释根本不知道这是要干啥。经过搜索发现client-go里只用到了一个IndexFunc，即MetaNamespaceIndexFunc，对应的IndexName为namespace，所以，一般情况下Indexers总是这个值{"namespace":MetaNamespaceIndexFunc}，举个栗子：

假如我要从Store中获取Key为default/test-app的Pod信息，那么上面的三个map中的内容分别为

Indexers: {"namespace":MetaNamespaceIndexFunc}

Index:{"default":{"default/test-app":}}

Indices: {"namespace":{"default":{"default/test-app":}}}

如果还是不懂，就先往下看，看Indexer的具体实现

```go
// 代码源自client-go/tools/cache/store.go
type cache struct {
   // cacheStorage bears the burden of thread safety for the cache
   cacheStorage ThreadSafeStore
   // keyFunc is used to make the key for objects stored in and retrieved from items, and
   // should be deterministic.
   keyFunc KeyFunc
}
  
// KeyFunc knows how to make a key from an object. Implementations should be deterministic.
type KeyFunc func(obj interface{}) (string, error)
  
// 代码源自client-go/tools/cache/thread_safe_store.go
type ThreadSafeStore interface {
   Add(key string, obj interface{})
   Update(key string, obj interface{})
   Delete(key string)
   Get(key string) (item interface{}, exists bool)
   List() []interface{}
   ListKeys() []string
   Replace(map[string]interface{}, string)
   Index(indexName string, obj interface{}) ([]interface{}, error)
   IndexKeys(indexName, indexKey string) ([]string, error)
   ListIndexFuncValues(name string) []string
   ByIndex(indexName, indexKey string) ([]interface{}, error)
   GetIndexers() Indexers
 
   // AddIndexers adds more indexers to this store.  If you call this after you already have data
   // in the store, the results are undefined.
   AddIndexers(newIndexers Indexers) error
   Resync() error
}
 
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
   lock  sync.RWMutex
   items map[string]interface{}
 
   // indexers maps a name to an IndexFunc
   indexers Indexers
   // indices maps a name to an Index
   indices Indices
}
```

上面代码很好理解，cache主要由线程安全的map[string]interface{}实现，Go的旧版本中没有sync.Map，这里还是利用sync.RWMutex实现，在这里又看到了上面出现的indexers和indices，还出现了一个新的函数类型KeyFunc，背后其实也只对应一个函数MetaNamespaceKeyFunc，下面通过分析cache.Add方法的实现来具体看下indexers、indices、KeyFunc是如何协调工作的

```go
// Add inserts an item into the cache.
func (c *cache) Add(obj interface{}) error {
   key, err := c.keyFunc(obj)
   if err != nil {
      return KeyError{obj, err}
   }
   c.cacheStorage.Add(key, obj)
   return nil
}
```

代码很简单，就是通过KeyFunc计算出obj的键，然后放到字典中，继续看字典的Add实现

```go
func (c *threadSafeMap) Add(key string, obj interface{}) {
   c.lock.Lock()
   defer c.lock.Unlock()
   oldObject := c.items[key]
   c.items[key] = obj
   c.updateIndices(oldObject, obj, key)
}
```

也很简单，取出当前key对应的obj，更新Indices，这里终于出现了我们关注的数据结构，继续看

```
// updateIndices modifies the objects location in the managed indexes, if this is an update, you must provide an oldObj
// updateIndices must be called from a function that already has a lock on the cache
func (c *threadSafeMap) updateIndices(oldObj interface{}, newObj interface{}, key string) {
   // if we got an old object, we need to remove it before we add it again
   if oldObj != nil {
      c.deleteFromIndices(oldObj, key)
   }
   for name, indexFunc := range c.indexers {
      indexValues, err := indexFunc(newObj)
      if err != nil {
         panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
      }
      index := c.indices[name]
      if index == nil {
         index = Index{}
         c.indices[name] = index
      }
 
      for _, indexValue := range indexValues {
         set := index[indexValue]
         if set == nil {
            set = sets.String{}
            index[indexValue] = set
         }
         set.Insert(key)
      }
   }
}
```

到这里，KeyFunc、Indices、Indexers都出现过了，这段代码也很简单，如果之前存在旧对象，先删除对应的Indices，然后遍历Indexers，根据每个IndexFunc计算出对应的IndexValues，获取对应Name的Index，不存在则创建，继续遍历之前得到的IndexValues，获取对应的set，不存在则创建，把key插入到set中，完成整个操作。还没懂的话，试着自己举个栗子，就拿obj为Pod类型为例，一步步下来，看下每个数据结构都存放什么就懂了。

## 总结

会不会有人有疑问，所有的对象都存在一个索引键下面，这样的效率岂不是太低了?其实client-go为每类对象都创建了Informer(Informer内有Indexer)，所以存储在相同索引建下的对象都是同一类，这个问题自然也就没有了。cache就类似数据库的Table，而且带了索引功能，可以试想一下自己想要实现一个带索引功能的缓存的话，怎么设计数据结构

- 对象如何存储？每个对象都有唯一键，通过KeyFunc计算出来，不同对象的键不重复，可以用map实现，对应上面的ThreadSafeMap
- 索引策略有哪些？用map[string]Func实现，key代表该索引策略名字，Func为策略，传入对象，返回对象在该策略下的索引值，比如key为namespace，Func(obj)的返回值就是该obj的namespace，如default，这里要注意，Func应该返回数组，例如如果策略为Label，一个对象的Label是有多个的，索引值就会有多个，对应上面的Indexers
- 索引值和对象的关系如何存储？一个索引值可能回应多个对象，如按照Label索引，需要一个map来存储，key为索引的值，值为对象集合，但是因为对象有唯一键，所以值可以用对象的唯一键的集合来减少内存消耗，但是这种数据结构在查询某个对象是否满足指定的索引值时需要遍历值的集合，很简单吗，值也用map来寸就可以了，key为对象的键，值为空结构体，即值得类型为map[string]struct{}，对应sets.String，整个数据结构对应上面的Index
- 索引策略和索引值的关系如何存储？很简单，一个map就搞定了，key为策略名字，value为Index，对应上面的Indices

上面基本可以实现功能了，还有一个可以考虑的问题：计算对象唯一键的方法、索引相关的三个数据结构放在哪里，k8s的实现里，KeyFunc是调用者和cache都知道具体的算法，索引相关的数据结构都放在了threadSafeMap里。





 
