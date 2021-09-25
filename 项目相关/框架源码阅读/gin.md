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

动态路由

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