# 支持GIF的自定义实现

[TOC]

## 背景

> 随着时间的推移，app开发已经不仅仅是从功能业务上去吸引用户，也需要增加一些夺人眼球的效果。这其中就包括一些设计做好的gif图

## 实现

> 核心就是创建Movie 对象，不断更新Movie对象播放的位置

```java
1. 创建Movie对象
  decodeStream(InputStream is)
  Note: 可以将gif图放在res/raw 目录下
2. 设置Movie当前播放时间
    setTime(int relativeMilliseconds)
3. 渲染到界面上
    draw(Canvas canvas, float x, float y)
```

## 具体实现

1. 测量

> Note: 当前gif view大小，可以根据Movie对象大小设定，当然也可以根据自身控件设置大小决定

2. 绘制

> ​Note:heavy_exclamation_mark: 不断设置movie播放的时间，绘制movie到界面上

## 注意事项

1. 如果是api>=11

`需要关闭硬件加速`

2. 如果api level >=`JELLY_BEAN`

可以使用`postInvalidateOnAnimation`请求绘制

3. 计算当前Movie时间方式

`mCurrentAnimationTime = (int) ((now - mMovieStart) % dur);`

Note:dur为需要在多长时间内播放完成gif图

