**debouce: 防抖**，触发事件后，在n秒内函数只能执行一次，如果在n秒内又触发了事件，则重新计算函数的执行时机。
例，web input输入框，接收用户输入，可能要根据用户输入实时做实时查询。假设一个用户疯狂输入，比如在键盘上一直歇斯底里的输入ssssssssssssssssssssssss，事件在不断触发。显然，我们不需要在事件疯狂触发时，就去请求后台接口（这也是不必要的，用户的输入不稳定，请求也是无效的）。我们可以做的，就是设置一个时间的阈值n，在n秒内，如果事件疯狂触发，且两次事件的间隔均小于n，则事件不会触发函数执行，只有当下次触发距离上次触发大于等于n时，会触发执行一次函数。
```javascript

    const debounce = (func, delay) => {
      let inDebounce
      return function() {
        const context = this
        const args = arguments
        clearTimeout(inDebounce)
        inDebounce = setTimeout(() => func.apply(context, args), delay)
      }
    }
```

**throttle，节流**，连续触发事件，但是在一段时间内只执行一次函数。
```javascript
const throttle = (func, limit) => {
  let inThrottle
  return function() {
    const args = arguments
    const context = this
    if (!inThrottle) {
      func.apply(context, args)
      inThrottle = true
      setTimeout(() => inThrottle = false, limit)
    }
  }
}
```

参考：
1. [Throttling and Debouncing in JavaScript](https://codeburst.io/throttling-and-debouncing-in-javascript-b01cad5c8edf "Throttling and Debouncing in JavaScript")
2. [在线实时编辑md](http://www.mdeditor.com/ "在线实时编辑md")
