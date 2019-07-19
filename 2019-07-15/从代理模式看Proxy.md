## 从代理模式看Proxy

编程在现实生活中都能找到原型。
这篇文章会从代理模式这个最常见的设计模式入手，进而来看下ES6的Proxy类是如何使用的。

学过HTTP协议相关知识,我们知道代理分为正向代理和反向代理。代理主要做权限控制,比如说科学上网;可以用来做身份验证,比如说局域网内的身份确认;可以用来做缓存,比如说缓存代理服务器,减小服务器压力;可以重定向请求,做内容分发,比如说反向代理服务器做网关层（网关用于转换各种上层HTTP协议的资源,隧道则是HTTP和其他协议的转换也是一种网关）;引入副作用,比如为每个请求加上日志。

归根结底,代理就是让客户端和服务端不直接进行通信,通过代理进行接下来的操作。

这种情形在互联网开发中太常见了。
Web开发中不允许用户直接修改数据库,必须发起Post或者Put请求通过Web服务器,让服务器调用RPC服务修改。
面向对象的语言不允许直接修改对象属性,而是通过set方法进行赋值（POJO）。
内部类不允许对外暴露，只能在框架内部使用（Vue框架Watcher等类）。

所以在代理这层能做的事情就比较多了。

首先是权限控制,在引入Proxy之前对对象的取值无法在底层进行校验和权限控制。

```javascript
// 引入前
let amount = {
  _max: 100,
  min: 10
}

console.log(amount._max, amount.min) // 100, 10

// 引入后
let amount = {
  _max: 100,
  min: 10
}
let amountProxy = new Proxy(amount, {
  get(target, key) {
    if (key.startsWith('_')) {
      console.log(`私有属性无法读取`)
      return
    }
    return target[key]
  },
  set(target, key, value) {
    if (key.startsWith('_')) {
      console.log(`无法修改私有属性`)
      return
    }
    if (!/\d+/.test(value)) {
      console.log(`${value}不是数字，不能设置`)
      return
    }
    target[key] = value
  }
})

console.log(amountProxy._max, amountProxy.min) // 私有属性无法读取, 10

amountProxy._max = '11' // 无法修改私有属性

```
上述amount变量被proxy对象引用,只有proxy对象才能访问到amount

再次，proxy对象可以引入副作用,一般用于测试用例。

```javascript
const a = {
  count: 0,
  times: 0
}

let count = 0

const aMock = new Proxy(a, {
  get(target, key, re) {
    return target[key]
  },
  set(target, key, value) {
    if (key === 'count') {
      count++
    }
    target[key] = value
    return true
  }
})

new Vue({
  data() {
    return aMock
  },
  methods: {
    click() {
      this.count++
    },
    update() {
      this.times = count
      // expect(this.times).to.equal(10)
    }
  },
  template: `<div><button @click="update">设置count的次数{{times}}</button><button @click="click">增加{{count}}</button></div>`
}).$mount('#app')

```
这里需要注意一点,set之后需要返回一个true,表示设置完成。

这里是用Proxy代理了根对象，在set和get中引入了副作用，实现了一个简单的单向绑定的todoMvc。
https://jack.ofspades.com/frameworkless-javascript-part-2-templates-and-rendering/
https://jack.ofspades.com/frameworkless-javascript-part-3-one-way-data-binding/

不过需要注意的是，一个Proxy对象，只能代理一层。这个跟Object.defineProperty是一样的。
我在这里用Proxy重构了一个简易版的Vue。

[代码库](https://github.com/LeeeeeeM/proxy)

[在线示例](https://leeeeeem.github.io/proxy/example/)