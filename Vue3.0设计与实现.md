# Vue 3.0 设计与实现

## 第一章 设计



## 第二章 响应式系统

不同于 Vue 2.x，Vue 3.x 放弃了 `Object.defineProperty()` 的方式，选择了 `ES6 Proxy` 来实现响应式系统，并且将响应式系统更多的暴露给用户。但是其中的设计思路并没有变化：对数据劫持，将其变为响应式数据，更具体一些就是，在 getter 中收集依赖，在 setter 中触发 trigger 来重新执行函数。

### 副作用函数与响应式数据

副作用函数，这是个与纯函数相对的概念。副作用函数执行会产生副作用（废话）。举个例子，Vue 模板中读取了一个数据（响应式数据），这里副作用函数就是 render 函数，render 函数读取响应式数据，并将 UI 渲染到页面上（其实只是生成了虚拟节点）。那么，当 render 函数读取过的数据变化时，render 函数应该重新执行，生成新的虚拟节点。我们抽取一下其中的思想：**如果一个副作用函数读取了响应式数据，那么响应式数据变化时，这个函数应该重新执行**。为什么一定是副作用函数呢？因为不产生我们需要的副作用的函数我们不 care 啊。简单的总结就是：**副作用函数，就是我们需要他跟着响应式数据变化的函数**。

所以说，响应式系统设计的重点有两个：响应式数据 和 副作用函数。**副作用函数读取了响应式数据，触发了响应式数据的 getter，就将自己记录到对应响应式数据的一种数据结构中；当响应式数据的 setter 被触发，说明副作用函数应该被执行，我们就可以找到对应的数据结构，将其中的函数全部执行一遍**。

```js
const obj = {
  foo: 1,
}

effect(() => {
  console.log(obj.foo);
})

// 当我们修改了obj.foo时，我们希望重新打印obj.foo
obj.foo++;

```

那么我们来设计一下 effect 函数：

```js
// 暂时存储被注册的副作用函数
let activeEffect;

function effect(fn) {
  // 把当前函数放入全局，方便 getter 能找到被注册的函数
  activeEffect = fn;
  // 执行 fn，这会触发响应式数据的 getter
  fn();
}
```

这时，我们还需要将 obj 变为响应式数据，主要思路就是之前说的 在 getter 收集依赖，在 setter 触发依赖，但是这时我们需要考虑将副作用函数收集在哪里，这里我就不卖关子了，收集进一个树型的数据结构：

```js
// 最外层是一个 weakMap，key 为响应式对象，value 为另一个 map
{
  // 第二层的 map，保存了对象具体的 key，以及对应的副作用函数们（保存在 Set 中，这样就不会重复）
  [obj1]: {
    // 每一个 key 对应一个 set，保存所有副作用函数
    'foo': [fn, fn, ...],
    'bar': [fn, ...]
  },
  [obj2]: {
    // ...
  }
}
```

第一层使用 `WeakMap` 是因为不希望产生引用导致垃圾回收机制无法回收响应式数据。到了第二层，就没有必要使用弱引用了，因为如果源对象被垃圾回收，那么对应的值，也就是 map 也会被回收掉。

设计完数据结构，我们就可以正式开始写响应式数据了：

```js
// 存储副作用函数
const bucket = new WeakMap();

const obj = new Proxy({ foo: 1 }, {
  // 对[[get]]操作进行拦截
  get(target, key) {
    // 如果找不到正在执行的副作用函数，直接返回
    if(!activeEffect) return;
    // 试图拿对应的map
    let depMap = bucket.get(target);
    // 如果没有就添加一个对应的map
    if (!depMap) {
      depMap = new Map();
      bucket.set(target, depMap);
    }
    // 试图拿对应的set，保存了所有副作用函数
    let depSet = depMap.get(key);
    if (!depSet) {
      depSet = new Set();
      depMap.set(key, depSet);
    }
    // 向set中添加这次读取数据的副作用函数，set本身是会去重的
    depSet.add(activeEffect);
    // 返回对应值
    return target[key];
  },
  // 对[[set]]操作进行拦截
  set(target, key, newVal) {
    // 设置属性值
  	target[key] = newVal;
    // 取对应的map
    const depMap = bucket.get(target);
    if (!depMap) return;
    const depSet = depMap.get(key);
    // 执行所有的副作用函数
    depSet && depSet.forEach(fn => fn())
	}
})
```

当然，我们可以把上面追踪依赖的逻辑（getter）和出发依赖的逻辑（setter）单独封装为函数，即 `track` 和 `trigger`。

```js
const bucket = new WeakMap();

const obj = new Proxy({ foo: 1 }, {
  get(target, key) {
    track(target, key);
    return target[key];
  },
  set(target, key, newVal) {
    trigger(target, key);
    target[key] = newVal;
  }
})

// 封装track逻辑
function track(target, key) {
	if (!activeEffect) return;
  let depMap = bucket.get(target);
  if (!depMap) {
    depMap = new Map();
    bucket.set(target, depMap);
  }
  let depSet = depMap.get(key);
  if (!depSet) {
    depSet = new Set();
    depMap.set(key, depSet);
  }
  depSet.add(activeEffect);
  return target[key];
}

// 封装trigger逻辑
function trigger(target, key) {
	const depMap = bucket.get(target);
  if (!depMap) return;
  const depSet = depMap.get(key);
  depSet && depSet.forEach(fn => fn());
}
```

现在我们只是大致实现了基本原理，但还是有很多细节没有考虑，比如当用户的副作用函数读取数据时，是通过分支语句读取的，那么当分支语句切换时，会产生遗留的副作用函数：

```js
const data = {
  ok: true,
  text: 'Hello World!'
}
const obj = new Proxy(data, {/* ... */})

effect(() => {
  // 这里使用三元表达式产生分支。
  document.body.innerHTML = data.ok ? data.text : 'not';
})
```

如果 `data.ok` 为 `true` ，那么副作用会被收集到 `data.ok` 以及 `data.text` 对应的 `set` 中。那么如果收集完后，`data.ok` 变为 `false`，那么 `data.text` 收集的依赖就是不需要的了，因为这里 `data.text` 的变化不会再影响页面上的数据了。那么我们怎么解决这个遗留副作用函数的问题呢？我们只需要**在每次执行副作用函数时，把该函数从所有收集了它的 Set 中移除**就好了，这样再次执行副作用函数时，会产生新的、必须的依赖，无用的依赖就不会被收集到。

想法很简单，但是我们需要能够从副作用函数找到所有相关联的响应式数据，所以我们在 `effect` 函数中，给所有的副作用函数添加一个 `dep` 数组，能保存下来这个函数被哪些 `Set` 收集了：

```js
function effect(fn) {
  // 再封装一层，这样就不会直接添加deps数组到用户的函数上了，而且每次执行副作用函数都会将它添加在全局的activeEffect中
  const effectFn = () => {
    activeEffect = effectFn;
    fn();
  }
  // 新添加的数组，记录Set
  effectFn.deps = [];
  // 执行函数
  effectFn();
}
```

那么 `effect` 这边的处理我们暂时就改好了，还有 `track` 我们需要处理。我们需要能够在一个函数被追踪时，将对应 `Set` 收集到函数中：

```js
function track(target, key) {
  if (!activeEffect) return;
  let depMap = bucket.get(target);
  if (!depMap) {
    depMap = new Map();
    bucket.set(target, depMap);
  }
  let depSet = depMap.get(key);
  if (!depSet) {
    depSet = new Set();
    depMap.set(key, depSet);
  }
  // 收集set到副作用函数中
  activeEffect.deps.push(depSet);
}
```

我们已经能够从副作用函数找到对应的 `Set` 了，现在在回头看 `effect`，我们的目的是每次执行之前先把自己从 `Set` 中清理掉，所以我们再封装一个 `cleanUp` 函数：

```js
function effect(fn) {
  const effectFn = () => {
    // 调用cleanUp，清理set
  	cleanUp(effectFn);
    activeEffect = effectFn;
    fn();
  }
  effectFn.deps = [];
  effectFn();
}

// 实现cleanUp函数
function cleanUp(effectFn) {
  // 遍历deps，删除掉所有set中的副作用函数
  effectFn.deps.forEach((depSet) => {
    depSet.delete(effectFn);
  });
  // 清空deps数组
  effectFn.deps.length = 0;
}
```

目前我们的逻辑通畅，但是执行会产生死循环，因为我们的 trigger 会遍历 set，执行所有函数，但执行函数会调用 cleanUp，将函数本身从 set 移除，随后又执行 fn，导致函数又被添加到 set 中，这就相当于：

```js
const set = new Set([1]);

set.forEach(item => {
  set.delete(1);
  set.add(1);
})
```

这会导致无限循环。解决方法就是，我们遍历一个 set 副本，而不是直接修改正在遍历的 set：

```js
function trigger(target, key) {
  const depMap = bucket.get(target);
  if (!depMap) return;
  const depSet = depMap.get(key);
  // 创建一个仅仅用来遍历的副本
  const depSetToRun = new Set(depSet);
  depSetToRun.forEach(effectFn => effectFn());
}
```

现在我们就完成了对遗留的副作用函数的清理。
