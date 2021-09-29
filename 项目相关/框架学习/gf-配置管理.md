# 配置管理

## 配置文件

默认的配置文件是`config.toml`

如果需要自定义文件格式，可以通过`SetFileName` 方式修改默认读取的配置文件名称。

举例：

```go
// 设置默认配置文件，默认的 config.toml 将会被覆盖
g.Cfg().SetFileName("config.json")
// 后续读取时将会读取到 config.json 配置文件内容，
g.Cfg().Get("database")
```

默认文件修改

三种方法：

1. 通过配置管理方法`SetFileName`修改。
2. 修改命令行启动参数 - `gf.gcfg.file`。
3. 修改指定的环境变量 - `GF_GCFG_FILE`。

举例：

1. **通过单例模式**

   ```
    g.Cfg().SetFileName("config.prod.toml")
   ```

2. **通过命令行启动参数**

   ```
    ./main --gf.gcfg.file=config.prod.toml
   ```

3. **通过环境变量（常用在容器中）**

   - 启动时修改环境变量：

     ```
       GF_GCFG_FILE=config.prod.toml; ./main
     ```

   - 使用`genv`模块来修改环境变量：

     ```
     genv.Set("GF_GCFG_FILE", "config.prod.toml")
     ```

## 配置读取

```go
// /home/www/templates/
g.Cfg().Get("viewpath")
// 127.0.0.1:6379,1
g.Cfg().Get("redis.cache")
// test2
g.Cfg().Get("database.default.1.name")
```

## 内容配置

```go
func SetContent(content string, file ...string)
func GetContent(file ...string) string
func RemoveContent(file ...string)
func ClearContent()
```

**需要注意的是该配置是全局生效的，并且优先级会高于读取配置文件。**

因此，假如我们通过`gcfg.SetContent("v = 1", "config.toml")`配置了`config.toml`的配置内容，并且也同时存在`config.toml`配置文件，那么只会使用到`SetContent`的配置内容，而配置文件内容将会被忽略。

## TOML格式

文档：https://github.com/LongTengDao/TOML/blob/%E9%BE%99%E8%85%BE%E9%81%93-%E8%AF%91/toml-v1.0.0.md



感觉没有什么好写的，别人文档写的太好了。。我只需要看懂就行了。