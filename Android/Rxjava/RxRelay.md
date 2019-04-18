### RxRelay

[rxrelay github 地址](https://github.com/JakeWharton/RxRelay)

看了下rxrelay的代码  好像只是对subject做了些修改

RxRelay 与Subject不同点

RxRelay 实现的是Consumer接口 Subject实现的是observer接口。

因为在Subject中除了onNext还有onError和onComplete方法 。Subject是有状态 一旦onError和onComplete被调用 Subject将会置为终止状态 这之后将会停止传递数据。 大神觉的Subject的作用是两个非响应式接口的桥梁 更多的作用是用了移动数据。而不是终止和错误事件 所以Rxrelay这个库 去除了onComplete和onError 让开发者只用来传递数据不用关心状态(没有onComplete所以也就没有AsyncRelay了)。而且关心状态的处理也会带来额外的性能损失



总结： 

RxRelay对于Subject去除了终止状态仅用于两个non-Rx api直接数据传递。