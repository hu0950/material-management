### 前言

> 本文先会从基础概念、实现方法入手，在此基础上，分析underscore的源码，探究其实现细节以及达到的效果，以便在今后的应用中，可以根据业务需求，来设计与实现适合需求场景的防抖和节流方案。最后，总结了一些防抖与节流常用的场景。

在日常开发中，会常遇到用户高频触发事件的操作，然而，这些高频率的触发，会带来一系列显而易见的问题，例如：
1. 事件触发的回调函数中，包含着比较复杂的处理逻辑，需要较多的运算时间和资源，会导致浏览器响应速度跟不上触发频率，页面会出现卡顿或假死的现象。
2. 高频事件的触发，往往存在着大量的DOM操作，会导致重排和重绘的触发频率过高，从而造成页面性能过度损耗以及 `CPU` 使用率过高，严重的会导致页面崩溃。
3. 每一次的事件触发，往往都代表需要和服务器建立一次http请求，而高频的触发，会在短时间内发起了数十次甚至上百次的请求，假设服务器没有设置限流的策略，必会给服务端带来很大的压力，造成服务器资源的浪费。

针对以上的问题，很多请求是不需要实时响应的，为了节省不必要的请求资源，可以采用防抖和节流两种方案来进行优化，从而达到在可高频触发场景中，控制事件回调函数执行的频次。

### 什么是防抖和节流？
防抖：降低连续执行调用的次数，将频繁的事件合成一次执行。

节流：设定一个单位时间，在这个单位时间内只能触发一次函数执行，如果在这个单位时间内，多次触发函数，只能有一次生效。

### 基本实现原理与实现

#### 1）防抖

> 实现思路：利用定时器计时的原理，在设定的等待时间内，若再次有触发，则计时器清零，开始重新计时，直到上一次结束触发后的等待时间内，没有新的触发，才会执行调用。

1. 非立即执行：第一次事件触发，并不会立即执行，包含第一次触发在内的所有触发，都需要等到delay时间之后再执行
```
function debounce(fn, delay) {
  return function (...args) {
    let _this = this
    // 如果存在定时器，清空上一次的定时器，保证在设置的延迟时间内，再次触发，计时器会重新开始计时
    if (fn.id) clearTimeout(fn.id)
    fn.id = setTimeout(() => {
      // delay时间后调用该函数，包括参数也是在delay时间结束后获取
      fn.apply(_this, args)
    }, delay)
  }
}
```
2. 立即执行：指第一次事件触发，会立即执行，此后的触发，都会延迟delay时间执行
```
function debounce(fn, delay) {
  return function(...args) {
    let _this = this
    // callNow用于标志是否是第一次执行
    let callNow = !fn.id
    if (fn.id) clearTimeout(fn.id)
    // 第一次执行时触发
    if (callNow) {
      fn.apply(_this, args)
    }
    // 每次触发事件的时候都会清除上次的延时器，并同时记录一个新的延时器，当事件停止触发后最后一次记录的延时器不会被清除可以延时执行
    fn.id = setTimeout(function() {
      fn.apply(_this, args)
    }, delay)
  }
}
```

当然，防抖会存在一定的问题，比如：对于持续的触发事件，就会存在永远无法执行事件处理函数的情况。但在特定的场景中，会更希望在每固定的时间段，必须执行一次事件处理函数，这时可以通过节流来实现这样的效果。

#### 2）节流

> 实现思路：有定时器和时间戳两种实现方式。定时器方式是利用定时器计时，事件触发会延迟到设定时间后再执行。时间戳方式是通过比较，本次触发距上一次执行调用的时间差不小于所设定的时间，从而判断是否可执行。

1. 定时器
```
function throttle(fn, delay) {
  return function(...args) {
    const _this = this
    if (!fn.id) {
      fn.id = setTimeout(() => {
        fn.id = null
        fn.apply(_this, args)
      }, delay)
    }
  }
}
```
TODO: 图

2. 时间戳
```
// prev的初始值设置为0，第一次触发会立即执行；设置为当前时间，第一次触发不会立即执行
let prev = Date.now()
function throttle(fn, delay) {
  return function(...args) {
    let _this = this
    let now = Date.now()
    if (now - prev >= delay) {
      fn.apply(_this, args)
      prev = now
    }
  }
}
```
TODO: 图

### `underscore`实现防抖和节流的源码分析
(备注：以下源码的分析，依赖的是underscore 1.9.1)

#### `underscore`防抖(debounce)实现
```
/**
 * underscore debounce 实现
 * @param func 要防抖动的函数
 * @param wait 需要delay执行的时间
 * @param immediate 是否需要立即执行
 * @returns {*}
 */
_.debounce = function(func, wait, immediate) {
    var timeout, result;
    var later = function(context, args) {
      timeout = null;
       // if (args)的判断，是为了过滤立即执行函数（设置immediate为true）的情况
       // 立即函数执行的情况，setTimeout(later, wait)，没有给later函数传入args
      if (args) result = func.apply(context, args);
    };

    // _.debounce的调用最终会返回debounced
    var debounced = restArguments(function(args) {
      // 若timeout有值，代表目前有计时器在计时，此时清空计时器，以保证每次执行_.debounce时，都会重新计时
      // 注意: 执行clearTimeout(timeout), 并不会使timeout为null
      if (timeout) clearTimeout(timeout);
      // 判断是否设置了immediate，即是否需要立即执行
      if (immediate) {
        var callNow = !timeout;
        // timeout记录是否有计时器，目的是为了控制在wait时间内不执行func。在wait之后，执行later，目的是将timeout设置为null
        timeout = setTimeout(later, wait);
        // 当满足第一次触发或是在持续触发的最后一次发生的wait时间的条件，timeout设为null，此后再触发时，立即执行回调函数
        if (callNow) result = func.apply(this, args);
      } else {
        // 触发后需要等到达了设定的时间后，才会执行回调函数，在这个过程中如果发生持续触发，定时器的计时会重新开始计时，直到最后一次触发wait时间后，才会执行回调函数func
        timeout = _.delay(later, wait, this, args);
      }
      return result;
    });

    // 增加手动清除定时器功能
    debounced.cancel = function() {
      clearTimeout(timeout);
      timeout = null;
    };

    return debounced;
};

// restArguments函数会将超出函数参数长度的所有剩余参数累积到一个成为最后一个参数的数组中。该方法与ES6提供的rest类似。
var restArguments = function(func, startIndex) {
    startIndex = startIndex == null ? func.length - 1 : +startIndex;
    return function() {
      var length = Math.max(arguments.length - startIndex, 0),
          rest = Array(length),
          index = 0;
      for (; index < length; index++) {
        rest[index] = arguments[index + startIndex];
      }
      switch (startIndex) {
        case 0: return func.call(this, rest);
        case 1: return func.call(this, arguments[0], rest);
        case 2: return func.call(this, arguments[0], arguments[1], rest);
      }
      var args = Array(startIndex + 1);
      for (index = 0; index < startIndex; index++) {
        args[index] = arguments[index];
      }
      args[startIndex] = rest;
      return func.apply(this, args);
    };
  };

// 将函数func延迟给定的毫秒数，然后调用函数与提供的参数
_.delay = restArguments(function(func, wait, args) {
    return setTimeout(function() {
      return func.apply(null, args);
    }, wait);
  });
```
通过一个流程图，表示debounce的实现效果：（以下图表示一次持续过程的触发，每次持续触发，重复此流程）
![image](https://raw.githubusercontent.com/hu0950/material-management/master/debounce/underscore-flow-chart.png)

#### immediate为true
    1. 在触发_.debounce后，会立即执行函数func
    2. 在设定的wait时间内，持续触发，都不会执行函数func，并且会重新开始计时，直到停止触发_.debounce的wait时间之后，才会将计时器timeout设置为null（timeout为null具备了下一次触发时可执行回调函数的条件），再次触发才会立即执行函数func

#### immediate为false
    1. 第一次触发_.debounce后，若在wait时间内，再无重复触发，到达了设定的时间wait后，才会执行回调函数func
    2. 若在这个过程中发生持续触发，定时器的计时会重新开始计时，在持续发生的最后一次触发wait时间后，才会执行回调函数func

**总结以上两种方式的区别，在于前者立即执行的是持续触发的第一次，而后者是延迟执行持续触发的最后一次。**

为了更清晰地解释实现效果，用一个例子看一下_.debounce的执行：

```
let debounceInputEl = document.getElementById('debounce')
debounceInputEl.addEventListener('keyup', (e) => {
 // 调用防抖方法
 // 设置immediate为true
 _.debounce(ajax, 1000, true)(e.target.value, '额外的参数')
 // 设置immediate默认为false
 _.debounce(ajax, 1000)(e.target.value, '额外的参数')
})
function ajax(...params) {
console.log('实际执行传入的函数func----', `参数：${params[0]}、${params[1]}`, format(+new Date()))
}

```
在例子中，所设定的wait时间间隔是1s。
设置immediate为true，执行结果如下：
![image](https://raw.githubusercontent.com/hu0950/material-management/master/debounce/result1.png)

在3s时，第一次触发，看到传入_.debounce的ajax函数被立即调用；在3s-4s的时间内，又重新触发，计时器会重新开始计算，原本在4s可以清空计时器，现在需要在第5s才会清空计时器，以此类推，在设定的wait时间内，只要持续触发，函数ajax都再不会执行。5s时，结束了上一次的连续触发，在wait时间后，即6s时，timeout已经被清空为null，此时，触发_.debounce又会立即调用ajax。

设置immediate为false，执行结果如下：
![image](https://raw.githubusercontent.com/hu0950/material-management/master/debounce/result2.png)
由运行结果可以看到，连续触发，计时器会重新开始计算，总是在连续触发的最后一次触发的1s之后，才会延迟调用传入_.debounce的ajax函数。

#### `underscore`节流(throttle)实现

```
/**
 * underscore throttle 实现
 * @param func 要节流的函数
 * @param wait 需要delay执行的时间
 * @param options 配置项：options.leading设置"节流前缘(leading edge)"，默认第一次尝试调用的 func 会被立即执行，若设置为false，第一次调用也必须等待 wait 时间后才会执行func。
 *                       options.trailing设置"节流后缘(trailing edge)"，默认会在持续触发结束后，再调用一次 func , 若设置为false，在上一次执行func和下一次即将执行func时间内发生的触发，都不会再执行func。
 * @returns {*}
 */
_.throttle = function(func, wait, options) {
  var timeout, context, args, result;
  var previous = 0;
  if (!options) options = {};

  var later = function() {
    // 根据options.leading的值重设previous，以保证本次持续触发结束后的下一次触发，是否需要立即执行
    previous = options.leading === false ? 0 : _.now();
    timeout = null;
    result = func.apply(context, args);
    if (!timeout) context = args = null;
  };

  var throttled = function() {
    console.log('input触发：', arguments, format(+new Date()))
    // 保存当前的时间戳
    var now = _.now();
    // 第一次执行时，previous为0，!previous为true
    // 如果设置了{ options.leading: false }, 将previous设置为当前时间
    if (!previous && options.leading === false) previous = now;
    // 为了计算持续触发函数的时间间隔与设定时间间隔
    var remaining = wait - (now - previous);
    context = this;
    args = arguments;
    if (remaining <= 0 || remaining > wait) {
      // 若有 timeout 存在，则取消计时
      if (timeout) {
        clearTimeout(timeout);
        timeout = null;
      }
      // 记录已触发过的时间戳
      previous = now;
      result = func.apply(context, args);
      // 由于变量都存在闭包中，垃圾回收机制无法回收，这里是为了清空数据，防止内存泄露
      // 再次检查 timeout，因为 func 执行期间可能有新的 timeout 被设置，如果 timeout 被清空了，代表不再有等待执行的 func，也清空 context 和 args
      if (!timeout) context = args = null;
    } else if (!timeout && options.trailing !== false) {
      // 在开启options.trailing的模式下，在remaining时间后延迟执行func，可以理解为：执行该次func的时间点是上一次执行func时间 + wait
      // 可以进入该分支的触发，都不满足remaining <= 0 || remaining > wait，即当前的触发并不满足可以直接立即执行 func 的条件，需要延迟执行。
      timeout = setTimeout(later, remaining);
    }
    return result;
  };

  throttled.cancel = function() {
    clearTimeout(timeout);
    previous = 0;
    timeout = context = args = null;
  };

  return throttled;
};

通过一个流程图，表示throttle的实现效果：
```
如果觉得源码在没有上下文的情况下晦涩难懂，可以结合以下例子理解。

采用如下方式调用_.throttle方法：
```
let throttleInput = document.getElementById('throttle')
let options = {
  // leading: false
  // trailing: false
}
throttleInput.addEventListener('keyup', (e) => {
  // 调用节流方法
  _.throttle(ajax, 4000, options)(e.target.value)
})
function ajax(...params) {
  console.log('实际执行传入的函数func----', `参数：${params[0]}`, format(+new Date()))
}

```
1. 未设置leading和trailing
效果：第一次触发，立即执行。此后，若触发时间与上一次触发时间的差值大于wait，则立即执行，否则，延后执行，执行时间是：上一次执行func的时间+wait。
![image](https://raw.githubusercontent.com/hu0950/material-management/master/throttle/throttle_result1.png)
2. trailing:false
效果：第一次触发，立即执行。此后，若触发时间与上一次触发时间的差值大于wait，则立即执行，否则，不执行。
![image](https://github.com/hu0950/material-management/blob/master/throttle/throttle-result2.png)
3. 3. 两个为false(关闭立即和延时执行)
效果：第一次触发，不立即执行，并将该次触发时间赋值给previous，标记为已执行。此后每次的触发时间与previous的差值，若大于wait，则执行func，否则，不执行。当执行func时，会更新previous，再次通过以上规则进行比较，推测何时可执行func，以此类推。
![image](https://github.com/hu0950/material-management/blob/master/throttle/throttle-result3.png)
4. leading为false
效果：第一次触发，不立即执行。此后触发，延迟执行func，执行时间是：上一次执行结束后的第一轮触发时间+wait。
![image](https://github.com/hu0950/material-management/blob/master/throttle/throttle-result4.png)

是否设置options.leading = false ？
// 若是: 第一次触发不执行func。设置previous = now，是为了标志此轮已执行过，并使remaining = wait，因不满足执行func的条件。

// 若否: 第一次触发执行func。该次触发时，previous = 0, 满足remaining（等于 wait - (now - previous)） <= 0 的条件，符合执行func的条件。

// 包括第二次在内之后且不是最后一次的触发，若持续触发函数的时间间隔大于设定时间间隔wait（满足remaining > wait），则执行func；反之，则该次触发不执行func。

// 是否设置options.trailing = false ?

// 是：在上一次执行func和下一次即将执行func之间所发生的触发结束后，再调用一次 func

// 否：在上一次执行func和下一次即将执行func之间所发生的触发结束后，都不会再执行func。

### 适用场景

这两种方法都是为了限制执行调用的频率，那如何区别在某种场景中，应该要使用何种方法？

> 首先，需要明确的是：debounce的作用是为了降低连续执行调用的次数。

例如：
1. 表单输入校验场景：当用户输入第一次输入的一段时间内，如果还有字符输入的话，可以暂时不去请求校验，以免用户还未输入完成，就向用户提示校验的错误信息，体验未必友好。这时候我们并不希望在每次输入都发请求校验，而是在其完全停止输入时，再对输入内容进行校验。
2. 滚动场景：要实现滚动到底部自动加载更多的效果，对监听浏览器scroll事件使用debounce，只有当用户停止滚动后，才会判断是否滚动到页面底部，这样大大可以减少在滚动中，一些不必要的执行。

> 相比debounce， throttle的目的在于：使函数在固定的频率执行调用。

例如：
1. 类似window的resize和mousemove事件，只要一变化，就会一下会带来大量事件的触发，引发多次重绘与重排，需要通过节流控制回调的发生频率。
2. 图片懒加载场景：可对用户下拉的操作触发进行节流，在每固定的时间判断是否应该展示图片即可。
3. 表单提交场景：控制提交按钮点击事件的频率，防止用户在短时间内多次重复提交
4. 滚动场景：判断是否到页面底部自动加载更多，对监听浏览器scroll事件使用throttle，页面滚动每间隔一段时间会判断一次。
5. 搜索联想场景：用户在短时间内不停的输入，但并不要实时响应搜索结果，这种场景可以通过对输入的触发进行节流，在不影响用户体验的情况下，节约请求资源。
6. DOM 元素的拖拽，控制单位时间拖拽触发的事件频率。
7. 射击游戏的 mousedown/keydown 事件，控制单位时间只能发射子弹的频率。

在实际的开发中，并没有严格意义规定，哪种特定的场景应该使用防抖或是节流才是正确的，而是应该根据具体的需求以及想要实现的效果，来选择较为合理的方案。

### 总结
防抖与节流都是为了限制执行调用的频率，区别在于：防抖的作用是要将连续触发的频率降低至一次执行，那么在某段较长的时间内，触发事件是连续发生的，就会出现函数将一直推迟执行，会造成不会被执行的效果；而节流则避免了即使是触发事件是连续发生的，也会保证在固定的频率真正执行一次调用。



