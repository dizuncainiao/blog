---
title: Event Bus
date: 2021-12-23 15:47:51
---

## 前言

我们都知道在 Vue 中父子组件通信使用 `props`、`$emit`，兄弟组件或者跨组件之间可以通过 `Vuex` 或 `EventBus` ，包括 `$parent` 、`$root` 和 `$refs` 都是可以达到通信的目的，总之 Vue 组件之间的通信方式非常丰富。

今天的主角是`EventBus` ，那么先谈谈我对它的理解。

`EventBus` 之所以能实现全局跨组件通信首先因为它是一个全局对象，可以通过挂在 `window` 或 `Vue.prototype` 上，并且能通过触发事件传参来实现数据的传递。

在小型项目（二三十个页面）中使用起来还是比较友好的，比起 `Vuex` 要来的简单粗暴一点。因为 `Vuex` 需要建立 `mutations` 、`actions` ，某些逻辑交互甚至需要借助 `computed` + `watch` 来实现，这在小项目中显得比较臃肿。

在大型项目（几百个页面）中使用 `EventBus` 不是不行，就是这里面水很深，很难把握的住……

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021050719014288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


`EventBus` 的缺点就是 `Vuex` 的优点，缺乏 **状态管理** 。试想一下我们在看一段代码时，看到 `$emit('someEvent')` 或  `$on('someEvent')`后想要知道它分别在哪里被监听，在哪里被触发，然后全局一搜竟然有几十处，这……😵

但这并不意味着 `EventBus` 在大型项目中就不能用，二者结合起来用才是最香的。比如这样的场景：**根据父组件中导航菜单的开合状态在子组件中处理一些逻辑** ，类似于这样的场景使用 `EventBus` 是不是要简单粗暴一点，触发个事件就解决了。`EventBus` 比较适合在那种 **定向的**、**耦合度较低** 的功能场景中使用。

对于 `Vuex` 和 `EventBus` 之间我觉得基本上能够相互替代，但是不能完美替代。这两者在我的日常开发中都是缺一不可的，但是就是这么好用的 `EventBus`  在 [Vue3](https://v3.cn.vuejs.org/guide/migration/events-api.html#%E4%BA%8B%E4%BB%B6-api) 中被移除了，怎么办🤔，嗯。。。要不写一个？

## 步骤

做任何事情都是分步骤的，把大象装进冰箱里还分成 3 步呢，那么写一个 `EventBus` 可以分为以下步骤：

- `on` 用来监听当前实例上的自定义事件。事件可以由 `emit` 触发。回调函数会接收所有传入事件触发函数的额外参数。
- `once` 监听一个自定义事件，但是只触发一次。一旦触发之后，监听器就会被移除。
- `off` 移除自定义事件监听器。
    - 如果没有提供参数，则移除所有的事件监听器；
    - 如果只提供了事件，则移除该事件所有的监听器；
    - 如果同时提供了事件与回调，则只移除这个回调的监听器。

- `emit` 触发当前实例上的事件。附加参数都会传给监听器回调。

为了尽量 1:1 模拟出 Vue2 中的用法，这里就直接照搬了 Vue2 中的 [#实例方法-事件](https://cn.vuejs.org/v2/api/#%E5%AE%9E%E4%BE%8B%E6%96%B9%E6%B3%95-%E4%BA%8B%E4%BB%B6) 。有了步骤以后实现一个 `EventBus` 就已经完成了一大半了，下面待我慢慢道来。

## 事件监听-on

`on` 方法传入两个参数，第一个是 **事件** *string* ，第二个是 **回调函数** 。`on` 与其说是事件监听，不如将它理解为 **注册** 、**登记** ，因为 `on` 的核心就是将 **事件** 以及对应的 **回调** 收集供 `emit` 触发。实现代码如下：

```javascript
class DzEmitter {
    _events = {}

    static addEvent(e, callback, isOnce = false) {
        const {_events} = this
        const keys = Object.keys(_events)
        if (keys.includes(e)) {
            _events[e].isOnce = isOnce
            _events[e].callbacks.push(callback)
        } else {
            _events[e] = {
                isOnce,
                callbacks: [callback]
            }
        }
    }

    // 监听当前实例上的自定义事件
    on(e, callback) {
        DzEmitter.addEvent.call(this, e, callback)
    }
}
```

这里使用 `_events` 来收集 **事件** 与 **回调** 。需要注意的是，一个 **事件** 可以对应多个 **回调** ，因此要加以处理下。一个注册了多个相同事件的 `_events` 数据如下：

```javascript
const _events = {
    selfClick: {
        isOnce: true,
        callbacks: [fn, fn2]
    }
}
```

`isOnce` 是用来判断注册来源是 `once` 还是 `on` 。

## 事件触发-emit

`emit` 传入两个参数，第一个是 **事件** *string* ，第二个是 **传给回调的参数** 。其核心就是根据 **事件名称** ，把参数传给 **回调** ，并调用 **回调函数** 就行了。根据上面的 `_events` 数据，写出以下代码：

```javascript
// 触发当前实例上的事件。附加参数都会传给监听器回调。
emit(e, data) {
    try {
        const {callbacks, isOnce} = this._events[e]
        callbacks.forEach(fn => fn(data))
        isOnce && this.off(e)
    } catch (e) {
        console.error(e)
    }
}
```

因为一个 **事件** 对应多个 **回调** ，所以触发事件时它对应的数个回调要一同调用。由 `once` 注册的事件需要在第一次调用后销毁掉。

## 完整代码

关于 `once` 和 `off` 就不单独讲了，直接上全部代码：

```javascript
class DzEmitter {
    _events = {}

    static addEvent(e, callback, isOnce = false) {
        const {_events} = this
        const keys = Object.keys(_events)
        if (keys.includes(e)) {
            _events[e].isOnce = isOnce
            _events[e].callbacks.push(callback)
        } else {
            _events[e] = {
                isOnce,
                callbacks: [callback]
            }
        }
    }

    // 触发当前实例上的事件。附加参数都会传给监听器回调。
    emit(e, data) {
        try {
            const {callbacks, isOnce} = this._events[e]
            callbacks.forEach(fn => fn(data))
            isOnce && this.off(e)
        } catch (e) {
            console.error(e)
        }
    }

    // 监听当前实例上的自定义事件
    on(e, callback) {
        DzEmitter.addEvent.call(this, e, callback)
    }

    // 监听一个自定义事件，但是只触发一次。一旦触发之后，监听器就会被移除
    once(e, callback) {
        DzEmitter.addEvent.call(this, e, callback, true)
    }

    // 移除自定义事件监听器
    off(e, callback) {
        // 如果没有提供参数，则移除所有的事件监听器
        if (!arguments.length) {
            this._events = {}
        }

        // 如果只提供了事件，则移除该事件所有的监听器
        if (e && !callback) {
            delete this._events[e]
        }

        // 如果同时提供了事件与回调，则只移除这个回调的监听器
        if (e && callback) {
            const {callbacks} = this._events[e]
            const index = callbacks.findIndex(fn => fn === callback)
            index >= 0 && callbacks.splice(index, 1)
        }
    }
}
```

以上基本 1:1 模拟了 vue2 中的 `EventBus`  （ps：移除了数组传参的方式），将它设为一个全局对象后就能实现事件的监听和触发，从而实现全局通信。基本用法如下：

```javascript
const emitter = new DzEmitter()

const onClickFn = data => {
	console.log('on-1', data);
}
const onClickFn2 = data => {
	console.log('on-2', data);
}
const onceClickFn = data => {
	console.log('once-1', data);
}

emitter.on('onClick', onClickFn)
emitter.on('onClick', onClickFn2)
emitter.once('onceClick', onceClickFn)

// emitter.off('onClick', onClickFn2)
// emitter.off('onClick')

setTimeout(() => {
    emitter.emit('onClick', `触发事件：onClick`)
    emitter.emit('onceClick', `触发事件：onceClick`)
    console.log(emitter);
}, 1000)
```

## 总结

以上便是我尝试写一个 `EventBus` 的全部了，整体代码量还是非常少的。猜测一下尤大在 Vue3 中移除 `EventBus` 的原因，可能是出于更好的性能考虑吧，也有可能是为了更贴切我们日常的编程直觉吧。毕竟实现一个简单的 `EventBus` 需要生成一个复杂的 Vue 实例感觉不是太好（ps：不纯粹，会有太多不相干的属性方法），借助于一些专注的库来实现这个功能应该会更好。
以上代码仅用于交流分享，请不要用在实际开发中，出了问题本人概不负责🤣🤣。 想要在 Vue3 中使用 `EventBus` 功能的，请看它的 [官方推荐](https://v3.cn.vuejs.org/guide/migration/events-api.html#%E8%BF%81%E7%A7%BB%E7%AD%96%E7%95%A5) 。

