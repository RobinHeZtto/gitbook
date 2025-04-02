# Kotlin基础-委托属性



#### 委托属性的语法：

语法： `val/var <属性名>: <类型> by <表达式>`。

在 _by_ 后面的表达式是该 _委托_， 因为属性对应的 `get()`（与 `set()`）会被委托给它的 `getValue()` 与 `setValue()` 方法。 属性的委托不必实现任何的接口，但是需要提供一个 `getValue()` 函数（与 `setValue()`——对于 _var_ 属性）







```
by viewModels()
```
