# gin的路由

在前面的《gin是如何运行的》，我们讲解了简单的路由注册的一个过程。现在我们将详细的讲解路由注册的全过程，请系好安全带！马上开车！！！✈✈✈

## 前言

gin利用的前缀树方式注册路由。

![trie tree](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924185223.jpeg)

HTTP请求的路径恰好是由`/`分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

```go
func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.POST("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.GET("/ping/:where", func(c *gin.Context) {
		w:=c.Param("where")
		c.JSON(200,gin.H{
			"where":w,
		})
	})
	r.POST("/ping/hello/*", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"message": "hello",
		})
	})
	err := r.Run()
	if err != nil {
		return 
	} // 监听并在 0.0.0.0:8080 上启动服务
}
```

我会根据上面的函数，一起Debug看看，gin是如何注册一个路由树，以及如何匹配的。

## 路由树

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")
	//对三个参数的判断的处理
	debugPrintRoute(method, path, handlers)
	//打印相关信息
    
    //根据请求方式去构建了一个method树 ，都放在一样的请求方式的路由
	root := engine.trees.get(method)
    //如果没有，那么建立一个新的节点
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
    //注册路由，添加节点
	root.addRoute(path, handlers)

	// Update maxParams
	if paramsCount := countParams(path); paramsCount > engine.maxParams {
		engine.maxParams = paramsCount
	}
}
```

上次我们在这里打了断点，了解到了gin注册一个GET函数的一具体过程。现在我们进入一探究竟，gin到底怎么构建一个路由树的。

gin实现路由树的地方`tree.go`

进去后会看到一个tree的结构。

```go
type methodTree struct {
   method string //请求方式
   root   *node //子节点
}

type methodTrees []methodTree
```

```go
func (trees methodTrees) get(method string) *node {
	for _, tree := range trees {
		if tree.method == method {
			return tree.root
		}
	}
	return nil
}
```

当我们在注册一个函数时候，gin会先查询当前的路由树有没有这个请求方式，如果没有会返回一个空的node，之后作为根节点添加相关地址和执行函数。

既然他是以node作为每一个路由的对象，那么我们来看看node的结构。

```go
type node struct {
   path      string
   indices   string
   wildChild bool
   nType     nodeType
   priority  uint32
   children  []*node // child nodes, at most 1 :param style node at the end of the array
   handlers  HandlersChain
   fullPath  string
}
```

感觉看着这些字段名称还是不了解他的作用是什么，先不管，看看他的怎样构建的路由树？

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
   fullPath := path
   n.priority++

   // Empty tree
    //判断当前请求当时的根节点是否为空
   if len(n.path) == 0 && len(n.children) == 0 {
      n.insertChild(path, fullPath, handlers)
      n.nType = root
      return
   }

   parentFullPathIndex := 0

walk:
   for {
      // Find the longest common prefix.
      // This also implies that the common prefix contains no ':' or '*'
      // since the existing key can't contain those chars.
       //记录最长的公共前缀的字符数
      i := longestCommonPrefix(path, n.path)

      // Split edge
       //如果前缀字符数小当前根节点的长度，说明更新最长公共前缀
      if i < len(n.path) {
         child := node{//将当前路径分割出来，作为子节点
            path:      n.path[i:],//分割不相同的部分
            wildChild: n.wildChild,
            indices:   n.indices,
            children:  n.children,
            handlers:  n.handlers,
            priority:  n.priority - 1,
            fullPath:  n.fullPath,
         }

         n.children = []*node{&child}//添加子节点
         // []byte for proper unicode char conversion, see #65
         n.indices = bytesconv.BytesToString([]byte{n.path[i]})
         n.path = path[:i]
         n.handlers = nil
         n.wildChild = false
         n.fullPath = fullPath[:parentFullPathIndex+i]
      }

      // Make new node a child of this node
       //使当前节点（没有前缀的路径）成为新节点（最长公共前缀路径）的子节点
      if i < len(path) {
         path = path[i:]//获取路径
         c := path[0]//获取第一个字符 用于后面判断是：，*还是/

         // '/' after param
         if n.nType == param && c == '/' && len(n.children) == 1 {
            parentFullPathIndex += len(n.path)
            n = n.children[0]
            n.priority++
            continue walk
         }

         // Check if a child with the next path byte exists
         for i, max := 0, len(n.indices); i < max; i++ {
            if c == n.indices[i] {
               parentFullPathIndex += len(n.path)
               i = n.incrementChildPrio(i)
               n = n.children[i]
               continue walk
            }
         }

         // Otherwise insert it
         if c != ':' && c != '*' && n.nType != catchAll {
            // []byte for proper unicode char conversion, see #65
            n.indices += bytesconv.BytesToString([]byte{c})
            child := &node{
               fullPath: fullPath,
            }
            n.addChild(child)
            n.incrementChildPrio(len(n.indices) - 1)
            n = child
         } else if n.wildChild {
            // inserting a wildcard node, need to check if it conflicts with the existing wildcard
            n = n.children[len(n.children)-1]
            n.priority++

            // Check if the wildcard matches
            if len(path) >= len(n.path) && n.path == path[:len(n.path)] &&
               // Adding a child to a catchAll is not possible
               n.nType != catchAll &&
               // Check for longer wildcard, e.g. :name and :names
               (len(n.path) >= len(path) || path[len(n.path)] == '/') {
               continue walk
            }

            // Wildcard conflict
            pathSeg := path
            if n.nType != catchAll {
               pathSeg = strings.SplitN(pathSeg, "/", 2)[0]
            }
            prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
            panic("'" + pathSeg +
               "' in new path '" + fullPath +
               "' conflicts with existing wildcard '" + n.path +
               "' in existing prefix '" + prefix +
               "'")
         }

         n.insertChild(path, fullPath, handlers)
         return
      }

      // Otherwise add handle to current node
      if n.handlers != nil {
         panic("handlers are already registered for path '" + fullPath + "'")
      }
      n.handlers = handlers
      n.fullPath = fullPath
      return
   }
}
```

先别急，这样单纯看代码是不会理解gin的构建路由树的思想，我们慢慢打断点，了解他的🐂思想。

## 打断点

![image-20210924191940138](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924191940.png)

当前注册的是第一个GET函数，当前的路由树是空的。`n.insertChild(path, fullPath, handlers)` 这个函数执行将当前的一些信息都给到了n这个节点。至于nType是干嘛，我们先别管。

![image-20210924193715625](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924193715.png)

然后路由树就会多一个GET树上的根节点。

![image-20210924194255600](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924194255.png)

接下来是注册的是第一个POST函数。其实他的流程和注册GET函数一致。

![image-20210924194531771](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924194531.png)

接下来注册的是第二个GET函数`get /ping/:where`。

他获取的不是一个空的node节点了，而是一个之前是根节点的`get /ping` 的节点信息

![image-20210924195415910](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924195415.png)

随后经过判断当前节点存在path，将`parentFullPathIndex := 0` 然后会进入到后面的walk执行函数里面。

