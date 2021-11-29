# Debug

## 1.gin 框架存储数字类型都是 float64

在这个项目中，我将获取参数的值，绑定到了 `context` 上，在 `gin` 中存储的值类型 `float64`。所以当你使用 `GetInt` 时候获取到的值是 `0`。

所以你需要采用 `GetFloat64` 方法获取数字。