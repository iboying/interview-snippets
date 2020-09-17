# 前端常见的功能实现

## 1. 防抖

```javascript
function debounce(func, ms = 500) {
  let timer;
  return function (...args) {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      func.apply(this, args);
    }, ms);
  };
}
```

## 2. 节流

```javascript
function throttle(func, ms) {
  let canRun = true;
  return function (...args) {
    if (!canRun) return;
    canRun = false;
    setTimeout(() => {
      func.apply(this, args);
      canRun = true;
    }, ms);
  };
}
```

## 3. new

```javascript
function myNew(Func) {
  const instance = {};
  if (Func.prototype) {
    Object.setPrototypeOf(instance, Func.prototype);
  }
  const res = Func.apply(instance, [].slice.call(arguments, 1));
  if (typeof res === "function" || (typeof res === "object" && res !== null)) {
    return res;
  }
  return instance;
}
```

## 4. bind

```javascript
Function.prototype.myBind = function (context = globalThis) {
  const fn = this;
  const args = Array.from(arguments).slice(1);
  const newFunc = function () {
    if (this instanceof newFunc) {
      // 通过 new 调用，绑定 this 为实例对象
      fn.apply(this, args);
    } else {
      // 通过普通函数形式调用，绑定 context
      fn.apply(context, args);
    }
  };
  // 支持 new 调用方式
  newFunc.prototype = fn.prototype;
  return newFunc;
};
```

## 5. call

```javascript
Function.prototype.myCall = function (context = globalThis) {
  // 关键步骤，在 context 上调用方法，触发 this 绑定为 context
  context.fn = this;
  let args = [].slice.call(arguments, 1);
  let res = context.fn(...args);
  delete context.fn;
  return res;
};
```

## 6. apply

```javascript
Function.prototype.myApply = function (context = globalThis) {
  // 关键步骤，在 context 上调用方法，触发 this 绑定为 context
  context.fn = this;
  let res;
  if (arguments[1]) {
    res = context.fn(...arguments[1]);
  } else {
    res = context.fn();
  }
  delete context.fn;
  return res;
};
```

## 7. deepCopy

```javascript
function deepCopy(obj, cache = new WeakMap()) {
  if (!obj instanceof Object) return obj;
  // 防止循环引用
  if (cache.get(obj)) return cache.get(obj);
  // 支持函数
  if (obj instanceof Function) {
    return function () {
      obj.apply(this, arguments);
    };
  }
  // 支持日期
  if (obj instanceof Date) return new Date(obj);
  // 支持正则对象
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  // 还可以增加其他对象，比如：Map, Set等，根据情况判断增加即可，面试点到为止就可以了

  // 数组是 key 为数字素银的特殊对象
  const res = Array.isArray(obj) ? [] : {};
  // 缓存 copy 的对象，用于出来循环引用的情况
  cache.set(obj, res);

  Object.keys(obj).forEach((key) => {
    if (obj[key] instanceof Object) {
      res[key] = deepCopy(obj[key], cache);
    } else {
      res[key] = obj[key];
    }
  });
  return res;
}
```

## 8. 事件总线 | 发布订阅模式

```javascript
class EventEmitter {
  constructor() {
    this.cache = {};
  }

  on(name, fn) {
    if (this.cache[name]) {
      this.cache[name].push(fn);
    } else {
      this.cache[name] = [fn];
    }
  }

  off(name, fn) {
    const tasks = this.cache[name];
    if (tasks) {
      const index = tasks.findIndex((f) => f === fn || f.callback === fn);
      if (index >= 0) {
        tasks.splice(index, 1);
      }
    }
  }

  emit(name) {
    if (this.cache[name]) {
      for (let fn of this.cache[name]) {
        fn();
      }
    }
  }

  emit(name, once = false) {
    if (this.cache[name]) {
      // 创建副本，如果回调函数内继续注册相同事件，会造成死循环
      const tasks = this.cache[name].slice()
      for (let fn of tasks) {
        fn();
      }
      if (once) {
        delete this.cache[name]
      }
    }
  }
}
```

## 9. 柯里化：只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数

```javascript
function curry(func) {
  return function curried(...args) {
    if (args.length >= func.length) {
      func.apply(this, args);
    } else {
      return function (...args2) {
        curried.apply(this, args.concat(args2));
      };
    }
  };
}
```

## 10. es5 实现继承

```javascript
function create(proto) {
  function F() {}
  F.prototype = proto;
  return new F();
}

// Parent
function Parent(name) {
  this.name = name;
}

Parent.prototype.say = function () {
  console.log(this.name);
};

// Child
function Child(age, name) {
  Parent.call(this, name);
  this.age = age;
}
Child.prototype = create(Parent.prototype);
Child.prototype.constructor = Child;

Child.prototype.say = function () {
  console.log(this.age);
};
```

## 11. instanceof

```javascript
function instanceOf(instance, klass) {
  let proto = instance.__proto__;
  let prototype = klass.prototype;
  while (true) {
    if (proto === null) return false;
    if (proto === prototype) return true;
    proto = proto.__proto__;
  }
}
```

## 12. 异步并发数限制

```javascript
/**
 * 关键点
 * 1. new promise 一经创建，立即执行
 * 2. 使用 Promise.resolve().then 可以把任务加到微任务队列，防止立即执行迭代方法
 * 3. 微任务处理过程中，产生的新的微任务，会在同一事件循环内，追加到微任务队列里
 * 4. 使用 race 在某个任务完成时，继续添加任务，保持任务按照最大并发数进行执行
 * 5. 任务完成后，需要从 doingTasks 中移出
 */
function limit(count, array, iterateFunc) {
  const tasks = [];
  const doingTasks = [];
  let i = 0;
  const enqueue = () => {
    if (i === array.length) {
      return Promise.resolve();
    }
    const task = Promise.resolve().then(() => iterateFunc(array[i++]));
    tasks.push(task);
    const doing = task.then(() => doingTasks.splice(doingTasks.indexOf(doing), 1));
    doingTasks.push(doing);
    const res = doingTasks.length >= count ? Promise.race(doingTasks) : Promise.resolve();
    return res.then(enqueue);
  };
  return enqueue().then(() => Promise.all(tasks));
}

// test
const timeout = i => new Promise(resolve => setTimeout(() => resolve(i), i));
limit(4, [1000, 1000, 1000, 1000], timeout).then((res) => {
  console.log(res);
})
```

## 13. 异步串行 | 异步并行

```javascript
// 字节面试题，实现一个异步加法
function asyncAdd(a, b, callback) {
  setTimeout(function () {
    callback(null, a + b);
  }, 1000);
}

// 0. promisify
const promiseAdd = (a, b) => new Promise((resolve, reject) => {
  asyncAdd(a, b, (err, res) => {
    if (err) {
      reject(err)
    } else {
      resolve(res)
    }
  })
})

// 1. 串行处理
async function serialSum(...args) {
  return args.reduce((task, now) => task.then(res => promiseAdd(res, now)), Promise.resolve(0))
}

// 2. 并行处理
async function parallelSum(...args) {
  if (args.length === 1) return args[0]
  const tasks = []
  for (let i = 0; i < args.length; i += 2) {
    tasks.push(promiseAdd(args[i], args[i + 1] || 0))
  }
  const results = await Promise.all(tasks)
  return parallelSum(...results)
}

// 测试
(async () => {
  const res1 = await serialSum(1, 2, 3, 4, 5, 8, 9, 10, 11, 12)
  console.log(res1)
  const res2 = await parallelSum(1, 2, 3, 4, 5, 8, 9, 10, 11, 12)
  console.log(res2)
})()
```

## 14. vue reactive

```javascript
// Dep module
class Dep {
  static stack = []
  static target = null
  deps = null
  
  constructor() {
    this.deps = new Set()
  }

  depend() {
    if (Dep.target) {
      this.deps.add(Dep.target)
    }
  }

  notify() {
    this.deps.forEach(w => w.update())
  }

  static pushTarget(t) {
    if (this.target) {
      this.stack.push(this.target)
    }
    this.target = t
  }

  static popTarget() {
    this.target = this.stack.pop()
  }
}

// reactive
function reactive(o) {
  if (o && typeof o === 'object') {
    Object.keys(o).forEach(k => {
      defineReactive(o, k, o[k])
    })
  }
  return o
}

function defineReactive(obj, k, val) {
  let dep = new Dep()
  Object.defineProperty(obj, k, {
    get() {
      dep.depend()
      return val
    },
    set(newVal) {
      val = newVal
      dep.notify()
    }
  })
  if (isObj(val)) {
    reactive(val)
  }
}

// watcher
class Watcher {
  constructor(effect) {
    this.effect = effect
    this.update()
  }

  update() {
    Dep.pushTarget(this)
    this.value = this.effect()
    Dep.popTarget()
    return this.value
  }
}

// 测试代码
const data = reactive({
  msg: 'aaa'
})

new Watcher(() => {
  console.log('===> effect', data.msg);
})

setTimeout(() => {
  data.msg = 'hello'
}, 1000)
```

## 15. promise

```javascript
// 建议阅读 [Promises/A+ 标准](https://promisesaplus.com/)
class MyPromise {
  constructor(func) {
    this.status = 'pending'
    this.value = null
    this.resolvedTasks = []
    this.rejectedTasks = []
    this._resolve = this._resolve.bind(this)
    this._reject = this._reject.bind(this)
    try {
      func(this._resolve, this._reject)
    } catch (error) {
      this._reject(error)
    }
  }

  _resolve(value) {
    setTimeout(() => {
      this.status = 'fulfilled'
      this.value = value
      this.resolvedTasks.forEach(t => t(value))
    })
  }

  _reject(reason) {
    setTimeout(() => {
      this.status = 'reject'
      this.value = reason
      this.rejectedTasks.forEach(t => t(reason))
    })
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      this.resolvedTasks.push((value) => {
        try {
          const res = onFulfilled(value)
          if (res instanceof MyPromise) {
            res.then(resolve, reject)
          } else {
            resolve(res)
          }
        } catch (error) {
          reject(error)
        }
      })
      this.rejectedTasks.push((value) => {
        try {
          const res = onRejected(value)
          if (res instanceof MyPromise) {
            res.then(resolve, reject)
          } else {
            reject(res)
          }
        } catch (error) {
          reject(error)
        }
      })
    })
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }
}

// 测试
new MyPromise((resolve) => {
  setTimeout(() => {
    resolve(1);
  }, 500);
})
  .then((res) => {
    console.log(res);
    return new MyPromise((resolve) => {
      setTimeout(() => {
        resolve(2);
      }, 500);
    });
  })
  .then((res) => {
    console.log(res);
  }, err => {
    console.log('==>', err);
  });
```
