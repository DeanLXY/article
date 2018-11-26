## 需求背景
主要是有业务需求的前提下，尽量不要做可以避免的硬编码，以防后续更改时挖下巨坑。
经常会在需求上遇到有对一个view(ViewGroup)做状态变更，ui状态改变
如果是单个View,是可以考虑直接使用系统控件去实现效果

```
例如 
RadioButton
RadioGroup
CheckedTextView
...
```
在满足比较常见的效果外，我们还会考虑如果是更为复杂的组合控件，甚至可能要考虑多种状态
CheckedTextView 能不能再添加一种新的状态呢，这确实是一个非常有意思的事情呢
## 添加新状态
### 支持新状态的属性（类型？）
先看下有哪些类型的属性是支持添加新状态的。
哈哈，皮了一下。
其实是绝大多数类型的属性都是支持的

```
    android:background
    android:text
    android:textColor
    ...    
```
以上对应系统定义的属性

```
     android.R.attr.background
     ...
```




