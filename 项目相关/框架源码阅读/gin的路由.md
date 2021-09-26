# gin的路由

在前面的《gin是如何运行的》，我们讲解了简单的路由注册的一个过程。现在我们将详细的讲解路由注册的全过程，请系好安全带！马上开车！！！✈✈✈

## 前言

gin利用的前缀树方式注册路由。

![trie tree](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210924185223.jpeg)

HTTP请求的路径恰好是由`/`分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.GET("/ping/hello", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"message": "hello",
		})
	})
	r.GET("/about", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"message": "about",
		})
	})
	r.GET("/about/*work", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"work":c.Param("work"),
		})
	})
	r.GET("/:lang/intro", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"lang":c.Param("lang"),
		})
	})
	r.GET("/:lang/doc", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"lang":c.Param("lang"),
		})
	})
	err := r.Run()
	if err != nil {
		return 
	} // 监听并在 0.0.0.0:8080 上启动服务
}
```

我会根据上面的函数，一起Debug看看，gin是如何注册一个路由树，以及如何匹配的。

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

## 路由树

### 前导知识

gin实现路由树的地方`tree.go`

进去后会看到一个tree的结构。

```go
type methodTree struct {
   method string //请求方式
   root   *node //子节点
}

type methodTrees []methodTree //多个请求方式字组成 
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

只要路由树不为空，那么他每次返回的节点都是根节点。

node的结构

```go
type node struct {
    path      string // 当前节点相对路径（与祖先节点的 path 拼接可得到完整路径）
    indices   string // 所有孩子节点的path[0]组成的字符串
    children  []*node // 孩子节点
    handlers  HandlersChain // 当前节点的处理函数（包括中间件）
    priority  uint32 // 当前节点及子孙节点的实际路由数量(当前树的节点个数)
    nType     nodeType // 节点类型
    maxParams uint8 // 子孙节点的最大参数数量
    wildChild bool // 孩子节点是否有通配符（wildcard）
    fullPath  string // 路由全路径
}
```

nType的值

```go
const (
    static nodeType = iota // 普通节点，默认
    root // 根节点
    param // 参数路由，比如 /user/:id
    catchAll // 匹配所有内容的路由，比如 /article/*key
)
```

### 执行情况

**声明：执行代码里面，没有执行的我都删除了，不要抱着同一个函数前后代码不一样而感到疑惑！！**

**第一个GET路由注册**

```go
r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
```

他将执行的代码

```go
//一开始就是一个空的根节点
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path  //  /ping
	n.priority++ //实际路由的数量  1

	// Empty tree
    //判断当前的节点是否是一个空的节点
	if len(n.path) == 0 && len(n.children) == 0 {
		n.insertChild(path, fullPath, handlers)//处理路径的通配符以及插入路径和HandlerFunc
		n.nType = root//设置当前节点类型
		return
	}
}
//主要处理路径的通配符，没有通配符相关的就会处理的很简单
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
	for {
		// Find prefix until first wildcard
        //查询是否有通配符（":","*"）
        //没有回返回“”，-1，false
		wildcard, i, valid := findWildcard(path)
		if i < 0 { // No wildcard found
			break
		}
    }
	//没有通配符,就简单插入路径和HandlerFunc
	n.path = path
	n.handlers = handlers
	n.fullPath = fullPath
}
    
```

**第二个GET路由注册**

```go
r.GET("/ping/hello", func(c *gin.Context) {
   c.JSON(200,gin.H{
      "message": "hello",
   })
})
```

他将执行的代码

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path  // /ping/hello
	n.priority++   // 2
    //path的共同前缀位置
    parentFullPathIndex := 0
walk:
	for {
        //获取公共前缀字符串的长度
        i := longestCommonPrefix(path, n.path)
        if i < len(path) {
            //获取除去前缀字符串的路径
			path = path[i:]
            //获取路径的第一个字符
			c := path[0]

			// Otherwise insert it
            
			if c != ':' && c != '*' && n.nType != catchAll {
                //没有通配符的路径，插入节点
				// []byte for proper unicode char conversion, see #65
				n.indices += bytesconv.BytesToString([]byte{c})
				child := &node{
					fullPath: fullPath,
				}
                //将孩子节点添加进入之前的节点
				n.addChild(child)
                //给当前节点的孩子节点排序（由于现在的路由简单就没有排序）
				n.incrementChildPrio(len(n.indices) - 1)
                //将child引用到n，后面对n进行插入信息
				n = child
			} 
			n.insertChild(path, fullPath, handlers)
			return
		}
    }
}
// addChild will add a child node, keeping wildcards at the end
//添加子节点，将通配符保留最后。
func (n *node) addChild(child *node) {
    //如果当前根节点是具有通配符的，子节点添加，将通配符保留最后。
	if n.wildChild && len(n.children) > 0 {
		wildcardChild := n.children[len(n.children)-1]
		n.children = append(n.children[:len(n.children)-1], child, wildcardChild)
	} else {
        //没有通配符直接添加
		n.children = append(n.children, child)
	}
}
//主要处理路径的通配符，没有通配符相关的就会处理的很简单
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
	for {
		// Find prefix until first wildcard
        //查询是否有通配符（":","*"）
        //没有回返回“”，-1，false
		wildcard, i, valid := findWildcard(path)
		if i < 0 { // No wildcard found
			break
		}
    }
	//没有通配符,就简单插入路径和HandlerFunc
	n.path = path
	n.handlers = handlers
	n.fullPath = fullPath
}
```

![image-20210925155258862](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925155305.png)

![image-20210925155713027](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925155713.png)

**第三个路由注册**

```go
r.GET("/about", func(c *gin.Context) {
   c.JSON(200,gin.H{
      "message": "about",
   })
})
```

他将执行的代码

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path //  /about
	n.priority++   //3
    parentFullPathIndex := 0
walk:
    for {
		// Find the longest common prefix.
		// This also implies that the common prefix contains no ':' or '*'
		// since the existing key can't contain those chars.
		i := longestCommonPrefix(path, n.path) //1   前缀的字符串是"/"

		// Split edge
        //当前缀字符串的长度，小于根节点路径长度。将当前节点作为一个子节点，添加到新建立的前缀节点
		if i < len(n.path) {
            //建立孩子节点，将当前节点信息都复制给孩子节点
			child := node{
				path:      n.path[i:],
				wildChild: n.wildChild,
				indices:   n.indices,
				children:  n.children,
				handlers:  n.handlers,
				priority:  n.priority - 1,
				fullPath:  n.fullPath,
			}
			//将当前的节点的子节点直接替换成孩子节点
			n.children = []*node{&child}
			// []byte for proper unicode char conversion, see #65
            //获取子节点的首字符
			n.indices = bytesconv.BytesToString([]byte{n.path[i]})
            //当前节点的路径替换成前缀字符串
			n.path = path[:i]
			n.handlers = nil
			n.wildChild = false
            //当前节点的完整路径换成前缀字符串
			n.fullPath = fullPath[:parentFullPathIndex+i]
		}

		// Make new node a child of this node
        //添加新的节点
		if i < len(path) {
            //获取除去前缀字符串的路径
			path = path[i:]
            //获取路径的首字符
			c := path[0]
			// Check if a child with the next path byte exists
            //检查当前路径是否存在与子节点首字母一致的
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
                //添加路径的首字符
				n.indices += bytesconv.BytesToString([]byte{c})
                //建立孩子节点
				child := &node{
					fullPath: fullPath,// ”/about“
				}
                //添加孩子节点
				n.addChild(child)
             	//给当前节点的孩子节点排序（由于现在的路由简单就没有排序）
				n.incrementChildPrio(len(n.indices) - 1)
                //将孩子节点引用到n
				n = child
			} 
			//对孩子节点进行插入信息
			n.insertChild(path, fullPath, handlers)
			return
		}

		// Otherwise add handle to current node
        //对于出现重复的路径的处理，覆盖之前的HandlerFunc
		if n.handlers != nil {
			panic("handlers are already registered for path '" + fullPath + "'")
		}
		n.handlers = handlers
		n.fullPath = fullPath
		return
	}
}
//添加子节点，将通配符保留最后。
func (n *node) addChild(child *node) {
    //如果当前根节点是具有通配符的，子节点添加，将通配符保留最后。
	if n.wildChild && len(n.children) > 0 {
		wildcardChild := n.children[len(n.children)-1]
		n.children = append(n.children[:len(n.children)-1], child, wildcardChild)
	} else {
        //没有通配符直接添加
		n.children = append(n.children, child)
	}
}
```

![image-20210925163423246](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925163423.png)

当前构建的路由树

![image-20210925163634350](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925163634.png)

经过这三个简单的路由注册，了解到比较详细的构建路由树的过程。

### 动态路由

**通配符*路由**

```go
r.GET("/about/*work", func(c *gin.Context) {
   c.JSON(200,gin.H{
      "work":c.Param("work"),
   })
})
```

他将执行的代码

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path // "/about/*work"
	n.priority++ //4
    parentFullPathIndex := 0
walk:
	for {
		// Find the longest common prefix.
		// This also implies that the common prefix contains no ':' or '*'
		// since the existing key can't contain those chars.
        //  前缀的字符串是"/"
		i := longestCommonPrefix(path, n.path) //1

		// Make new node a child of this node
		if i < len(path) {
			path = path[i:] //"about/*work"
			c := path[0] //a

			// 检查下一个路径字节的子节点是否存在
            //如果存在的话，当前的路由树就是子树了
			for i, max := 0, len(n.indices); i < max; i++ {
                //当前路径首字母与根节点的子节点首字母相同
				if c == n.indices[i] {
                    //当前节点前缀的位置
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i) //1
                    //将当前路由树换成匹配成功的子树
					n = n.children[i]
                    //回到一开始，当前的walk的下面的代码没有再次进行了。
					continue walk
				}
			}
	}
//接下来的代码时 执行continue walk后的过程！！！！！请不要出现思维混乱的状况
walk:
	for {
		// Find the longest common prefix.
		// This also implies that the common prefix contains no ':' or '*'
		// since the existing key can't contain those chars.
		i := longestCommonPrefix(path, n.path) //5 前缀字符串 "/about"
		// Make new node a child of this node
		if i < len(path) {
            //"/*work"
			path = path[i:]
            //"/"
			c := path[0]

			// Otherwise insert it
			if c != ':' && c != '*' && n.nType != catchAll {
				// []byte for proper unicode char conversion, see #65
                //"/"
				n.indices += bytesconv.BytesToString([]byte{c})
                //建立孩子节点
				child := &node{
					fullPath: fullPath, //"/about/*work"
				}
                //添加子节点
				n.addChild(child)
                //这里没有排序
				n.incrementChildPrio(len(n.indices) - 1)
                //将当前子树换成子节点
				n = child
			} 
			n.insertChild(path, fullPath, handlers)
			return
		}
	}
}
// 增加给定孩子的优先级并在必要时重新排序
func (n *node) incrementChildPrio(pos int) int {
    //获取当前节点的子节点
	cs := n.children
    //给对应的子树 节点数量+1
	cs[pos].priority++
	prio := cs[pos].priority //2

	// Adjust position (move to front)
    //会根据数量进行交换，把子树节点多的子树放到前面
	newPos := pos // 1
	for ; newPos > 0 && cs[newPos-1].priority < prio; newPos-- {
		// Swap node positions
		cs[newPos-1], cs[newPos] = cs[newPos], cs[newPos-1]
	}

	// 如果上面的顺序发生了变化，那么就需要构建新的索引字符串
	if newPos != pos {
		n.indices = n.indices[:newPos] + // Unchanged prefix, might be empty
			n.indices[pos:pos+1] + // The index char we move
			n.indices[newPos:pos] + n.indices[pos+1:] // Rest without char at 'pos'
	}

	return newPos
}
//当前节点对应的是之前建立的孩子节点
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
	for {
		// Find prefix until first wildcard
		wildcard, i, valid := findWildcard(path) //在当前的路径找到通配符"*" 返回 1 ，"*work" , "true"
        //没有通配符会退出
		if i < 0 { // No wildcard found
			break
		}

		// The wildcard name must not contain ':' and '*'
        //检测路径是否含有多个通配符
		if !valid {
			panic("only one wildcard per path segment is allowed, has: '" +
				wildcard + "' in path '" + fullPath + "'")
		}
		
		// check if the wildcard has a name
        //检测路径的通配符是否有名字
        
		if len(wildcard) < 2 {
			panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
		}
        //当通配符是 ":",这次没有执行可以不看
		if wildcard[0] == ':' { // param
		}

		// catchAll
        //检测当前通配符* 路径是否在path的最后
        //wildcard是通配符+名字
		if i+len(wildcard) != len(path) {
			panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
		}
		//路径存在冲突 通配符* 后面不能接/
		if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
			panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
		}
		//总结：通配符 * 后面不能接路径和/  
        
		// currently fixed width 1 for '/'
		i--
        //检查通配符* 前面是否有/
		if path[i] != '/' {
			panic("no / before catch-all in path '" + fullPath + "'")
		}

		n.path = path[:i]
		//通配符的第一个节点路径为空
		// First node: catchAll node with empty path
        //创建孩子节点
		child := &node{
			wildChild: true,//存在通配符的子节点
			nType:     catchAll, //节点类型-匹配全部
			fullPath:  fullPath, //"/about/*work"
		}
		//添加孩子节点
		n.addChild(child)
        //设置索引的字符串
		n.indices = string('/')
        //更换后续操作对象 ---上面的子节点
		n = child
        //节点数+1
		n.priority++
		//第二个节点是带有变量的。
		// second node: node holding the variable
		child = &node{
			path:     path[i:],//"/*work"
			nType:    catchAll, 
			handlers: handlers,
			priority: 1, 
			fullPath: fullPath,
		}
        //添加刚才创建的节点
		n.children = []*node{child}

		return
	}
}

```

子树构建情况：

![image-20210925180212692](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925180212.png)

路由构建情况

![image-20210925180443801](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925180443.png)

> 温馨提示：上面为空，是因为举例太特殊了。
>
> 如果是以下三个路由注册。
>
> `/about/ping/*work`
>
> `/about/ping`
>
> `/about/hello`
>
> 情况就会不一样了。
>
> ![image-20210926111834314](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210926111841.png)

**通配符：路由**

```go
r.GET("/:lang/intro", func(c *gin.Context) {
   c.JSON(200,gin.H{
      "lang":c.Param("lang"),
   })
})
```

他将执行的代码

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
    fullPath := path //"/:lang/intro"
	n.priority++ //5
    parentFullPathIndex := 0

walk:
	for {
		// Find the longest common prefix.
		// This also implies that the common prefix contains no ':' or '*'
		// since the existing key can't contain those chars.
		i := longestCommonPrefix(path, n.path) //1
		// Make new node a child of this node
		if i < len(path) {
            path = path[i:] // ":lang/intro"
            c := path[0] //":"

			// Check if a child with the next path byte exists
            //这里indices没有找到匹配的字符
			for i, max := 0, len(n.indices); i < max; i++ {
				if c == n.indices[i] {
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i)
					n = n.children[i]
					continue walk
				}
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
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
	for {
		// Find prefix until first wildcard
        //他的源码就会把通配符+名字都截取下来
        //i是通配符的位置
        wildcard, i, valid := findWildcard(path) //返回":lang" 0 true
		if i < 0 { // No wildcard found
			break
		}

		if wildcard[0] == ':' { // param
			if i > 0 {
				// Insert prefix before the current wildcard
                //截取通配符的前缀
                //这里没有执行
				n.path = path[:i]
				path = path[i:]
			}
			//创建孩子节点
			child := &node{
				nType:    param, 
                path:     wildcard,  //":lang"
                fullPath: fullPath,  //":lang/intro"
			}
            //添加节点
			n.addChild(child)
            //设置为true 说明这个节点下面有通配符 ：的节点
			n.wildChild = true
            //后续的操作节点换成了孩子节点
			n = child
            //节点数+1
			n.priority++

// 如果路径不以通配符结尾，则有
// 将是另一个以“/”开头的非通配符子路径
			if len(wildcard) < len(path) {
				path = path[len(wildcard):] //获取通配符后面的子路径
				//创建孩子节点
				child := &node{
					priority: 1,
					fullPath: fullPath,
				}
                //添加孩子节点
				n.addChild(child)
                //更换后续操作的节点
				n = child
				continue
                //continue之后他会为子路径建立一个新的节点，放到通配符节点下面。后面就不展示了
			}

			// Otherwise we're done. Insert the handle in the new leaf
			n.handlers = handlers
			return
		}
    }
	// If no wildcard was found, simply insert the path and handle
	n.path = path
	n.handlers = handlers
	n.fullPath = fullPath
}

```

![image-20210925203753064](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925203753.png)

上面所有路由注册情况：

![image-20210925203845024](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925203845.png)



### 总结：

> 希望你能通过上面的代码以及注释能看懂，路由构建的全过程！
>
> 当然其实因为路由比较简单，还有很多的代码没有讲解到，你可以自己探索！！！
>
> 声明：执行代码里面，没有执行的我都删除了，不要抱着同一个函数前后代码不一样而感到疑惑！！



## 路由组

其实路由组没有很特别的地方！别像太复杂了！！

核心代码：

```go
// Group 创建一个新的路由器组。 您应该添加所有具有公共中间件或相同路径前缀的路由。
// 例如，所有使用公共中间件进行授权的路由都可以分组。
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
   return &RouterGroup{
      Handlers: group.combineHandlers(handlers),
      basePath: group.calculateAbsolutePath(relativePath),
      engine:   group.engine,
   }
}
```

当你们建立了路由组v1的话，其实后续都是在这个路由组基础上加一些信息（basepath，Handlers）。带着这个路由信息去构建路由树。

![image-20210925205310412](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925205310.png)

## 如何查找路由

我之前讲过所有的路由请求，都是交到engine实现的`ServeHTTP` 里面。

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
   c := engine.pool.Get().(*Context)
   c.writermem.reset(w)
   c.Request = req
   c.reset()
	//这上面都是建立一个Context对象
    
    //这才是重点，处理请求的逻辑函数
   engine.handleHTTPRequest(c)

   engine.pool.Put(c)
}
```

发起一个请求

以`GET /ping` 为例

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method //获取请求方式  "GET"
	rPath := c.Request.URL.Path //获取请求的路径 "/ping"
	unescape := false

	// Find root of the tree for the given HTTP method
	t := engine.trees //获取engine的tree的结构
	for i, tl := 0, len(t); i < tl; i++ {
        //根据树查询请求方式
		if t[i].method != httpMethod {
			continue
		}
        //获取当前请求方式的根节点
		root := t[i].root
		// Find route in tree
        //获得路由相关信息 handlerfunc 完整路径
		value := root.getValue(rPath, c.params, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		if value.handlers != nil {
            //给context对象赋值
            //HandlerFunc
			c.handlers = value.handlers
            //完整路径
			c.fullPath = value.fullPath
            //执行HandlerFunc
			c.Next()
            //写入执行函数的内容
			c.writermem.WriteHeaderNow()
			return
		}
	}

	
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}

```

查找路由核心代码

```go
func (n *node) getValue(path string, params *Params, unescape bool) (value nodeValue) {
   var (
      skippedPath string
      latestNode  = n // 缓存最新节点
   )

walk: // Outer loop for walking the tree
   for {
      prefix := n.path //获取根节点的path 也就是前缀路径  ---》"/"
       //此时的请求路径长度大于根节点路径
      if len(path) > len(prefix) {
          //path路径的前缀等于根节点path
         if path[:len(prefix)] == prefix {
             //获取请求路径除去根节点的path后的路径 "ping"
            path = path[len(prefix):]

            // Try all the non-wildcard children first by matching the indices
             //获取路径的首字母
            idxc := path[0]  // "p"
            for i, c := range []byte(n.indices) {
                //根据索引找到了对应的子树
               if c == idxc {
                  //  strings.HasPrefix(n.children[len(n.children)-1].path, ":") == n.wildChild
                   //如果当前的根节点子节点存在通配符的路由，那么更新lastnode=n
                  if n.wildChild {
                     skippedPath = prefix + path
                     latestNode = &node{
                        path:      n.path,
                        wildChild: n.wildChild,
                        nType:     n.nType,
                        priority:  n.priority,
                        children:  n.children,
                        handlers:  n.handlers,
                        fullPath:  n.fullPath,
                     }
                  }
				//获取首字母匹配成功的子树
                  n = n.children[i]
                  continue walk
               }
            }
		//温馨提示：因为执行了continue walk后所以后面执行就是后续的walk了
   }
}
//这里接着上面的continue walk
walk: // Outer loop for walking the tree
	for {
		prefix := n.path //获取 “ping”
        //当前路径与当前节点匹配上了
			if path == prefix {
// 如果当前路径不等于 '/' 并且该节点没有注册的句柄并且最近匹配的节点有一个子节点
// 当前节点需要等于最新匹配的节点
                //找的是通配符的路径，我们现在的请求没有就跳过
			if latestNode.wildChild && n.handlers == nil && path != "/" {
				n = latestNode.children[len(latestNode.children)-1]
			}
			// We should have reached the node containing the handle.
			// Check if this node has a handle registered.
               //这里就是找到了节点
			if value.handlers = n.handlers; value.handlers != nil {
				value.fullPath = n.fullPath
				return
			}
			//其实后面的判断都是再找通配符的路由
                
			//如果此路由没有句柄，但此路由有通配符子级，则必须有此路径的句柄，并带有附加的尾部斜杠
			if path == "/" && n.wildChild && n.nType != root {
				value.tsr = true
				return
			}

			// 没有找到句柄。 检查此路径的句柄是否存在尾随斜杠推荐尾随斜杠
			for i, c := range []byte(n.indices) {
				if c == '/' {
					n = n.children[i]
					value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
						(n.nType == catchAll && n.children[0].handlers != nil)
					return
				}
			}

			return
		}
    }
       
       
```

以上就是简单的路由的全过程。

>  在这里我希望你能自己Debug，看看怎么获取动态参数的！

## 获取动态参数

这个过程其实不难，我简单说说。

举例：

当我发起`GET /ssss/intro` 这个请求时候。

* 他会先根据根节点的path，获取后续的节点，然后靠首字母查询子树，如果没有找到，就定义成是通配符的路由
* 因为通配符的路由都会放在最后，所以直接找根节点的最后一个子树（可以看`addChild`这个函数）
* 根据节点类型判断通配符，此时判断的是param
* 然后截取路由，根据“/” 截取到了 ”ssss“
* 他就会与当前子树的根节点“:leng”进行匹配,获得param的
  * key：“lang”
  * value：”ssss“
* 如果他还有子路径的话
* 将路径截取成“/intro”
* 在通配符路由子树下进行查询
* 工作其实上面发起的简单请求一致。

发起通配符路由，拿到的结果

![image-20210925215448262](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210925215448.png)

## 总结

以上就是gin路由的全过程了。

我的文章的全过程都是去Debug源码，跑执行的代码，至于跳过的，我都会删除，因为这样才能准确的看到全过程，不会看的很混乱。

>  建议自己Debug跑一边简单的路由请求全过程，然后拿自己曾经写的，再去Debug一遍，验证自己的猜想！！

