# Vue 3.0 设计与实现

## 第一章 设计



## 第二章 响应式系统

不同于 Vue 2.x，Vue 3.x 放弃了 `Object.defineProperty()` 的方式，选择了 `ES6 Proxy` 来实现响应式系统，并且将响应式系统更多的暴露给用户。但是其中的设计思路并没有变化：对数据劫持，将其变为响应式数据，更具体一些就是，在 getter 中收集依赖，在 setter 中触发 trigger 来重新执行函数。

### 副作用函数

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

#### cleanUp

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

目前我们的逻辑通畅，但是执行会产生死循环，因为我们的 `trigger` 会遍历 `set`，执行所有函数，但执行函数会调用 `cleanUp`，将函数本身从 `set` 移除，随后又执行 `fn`，导致函数又被添加到 `set` 中，这就相当于：

```js
const set = new Set([1]);

set.forEach(item => {
  set.delete(1);
  set.add(1);
});
```

这会导致无限循环。解决方法就是，我们遍历一个 `set` 副本，而不是直接修改正在遍历的 `set`：

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

### 响应式数据

#### 引用类型

之前我们都是硬编码的方式，对数据进行代理拦截，接下来我们要优化响应式数据的封装：

```js
function reactive(obj) {
  return new Proxy(obj, {
    // 各种拦截器
  });
}
```

值得注意的是，`reactive` 是深层响应的，因为其中的 `get` 拦截器如果发现返回的属性是一个对象，会递归的调用 `reactive` 返回响应的对象。

但是有时我们可能不希望深响应，这就催生了 `shallowReactive` ，即浅响应。浅响应不会递归的返回响应对象，而是只有一层响应。

同样，Vue 3.0 在完成各种代理时也完成了只读属性和浅只读属性，只需要在拦截器中拦截掉所有修改即可。

#### 原始类型

对于非原始值（即 JS 引用类型）的拦截，Proxy 是做不到的，因为他们并不按照引用的方式传递，当原始值赋值时，两个变量时完全没有关系的。所以 Vue 3.0 对于原始值的响应式处理是：**包裹一层对象，让它变成引用类型**。

```js
function ref(val) {
  return reactive({
    value: val
  });
}
```

所以从实现来看，`ref(1)` 与 `reactive({ value: 1 })` 并没有区别。但是我们需要知道它是一个 ref 包裹的原始值，还是真的就是一个响应的对象，因为这涉及到 Vue 自动脱 ref（Vue 为了避免用户反复通过 `.value` 语法来编码，会自动取 ref 对象的 `value` 属性）。所以 Vue 给 ref 对象添加了一个不可枚举的属性来标记：

```js
function ref(val) {
  const wrapper = {
    value: val
  };
  // 添加一个不可枚举的属性标记
  Object.defineProperty(wrapper, '__v_isRef', {
    value: true
  });
  return reactive(wrapper);
}
```

#### toRef 和 toRefs

我们可能会希望在 `setup` 返回的对象中直接展开一个包裹好的响应式对象，这样代码会看起来很简洁，而且不需要反复的用 `.` 来取属性。但是实际上，如果我们直接用展开运算符来展开一个响应式对象，那会导致丢失响应。

```js
const obj = reactive({
  foo: 1,
  bar: 2
});

return {
  ...obj
};
// 这么做等价于
return {
  foo: 1,
  bar: 2
};
```

所以需求是：我们能不能得到一个普通对象，其中的每一个属性都映射到了响应式的对象对应的属性上，这样我们就可以对这个普通对象展开，所有的属性依旧指向响应式对象的属性？

答案是可以的，我们需要使用 `getter` 和 `setter` 来让普通对象的属性和响应式对象的属性之间建立联系：

```js
const obj = reactive({ foo: 1, bar: 2 });

const newObj = {
  foo: {
    get value() {
      return obj.foo;
    },
    set value(newVal) {
      obj.foo = newVal;
    }
  },
  bar: {
    get value() {
			return obj.bar;
    },
    set value(newVal) {
      obj.bar = newVal;
    }
  }
}
```

观察上面的 `newObj`，我们其实是建立了一个新的普通对象，这个对象的每一个属性都类似于一个 `ref` 包裹的对象，引用着原始的响应对象。为了概念上的统一，我们将这些属性也视作 `ref` 对象。接下来将这个逻辑改为函数：

```js
function toRef(obj, key) {
  const wrapper = {
    get value() {
      return obj[key];
    },
    set value(newVal) {
      obj[key] = newVal;
    }
  }
  // 视作真正的 ref 对象
  Object.defineProperty(wrapper, '__v_isRef', {
    value: true
  })
  
  return wrapper;
}

const obj = reactive({ foo: 1, bar: 2 });
// 我们希望得到的，为展开运算符消费的普通对象
const newObj = {
  // 其中的属性保持了对响应式对象的链接，可以看做是真正的 ref 对象
  foo: toRef(obj, 'foo'),
  bar: toRef(obj, 'bar')
};
```

那么我们可以再封装一次，让我们不需要自己一次次调用 `toRef` 函数：

```js
function toRefs(obj) {
  const res = {};
  for (const key in obj) {
    // 调用toRef来批量转换
    res[key] = toRef(obj, key);
  }
  return res;
}
```

这样，当我们读取 `toRef` 产生的对象时，其实是读取了对应响应式数据，设置其值时，也是对响应式数据进行设置。

#### 自动脱 ref

为了不给用户增加更多的心智负担，我们希望在模板中能自动脱去 ref 的能力，即会自动读取 `ref` 对象的 `value` 属性。

```html
<template>
	<!-- 我们希望这么使用，而不是foo.value -->
	<h2>{{foo}}</h2>
</template>

<script>
	export default {
    setup() {
      const obj = reactive({
        foo: 1,
        bar: 2
      });
      
      return {
        ...toRefs(obj)
      }
    }
  }
</script>
```

Vue 给出的解决方案是：再通过 Proxy 代理一次。

```js
function proxyRefs(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      const value = Reflect.get(target, key, receiver);
      // 如果是 ref 对象，就读取 value 属性
      return value.__v_isRef ? value.value : value;
    },
    set(target, key, newValue, receiver) {
      const value = target[key];
      if (value.__v_isRef) {
        value.value = newValue;
        return;
      }
      return Reflect.set(target, key, newValue, receiver);
    }
  })
}
```

`setup` 函数返回的对象，会被这个函数处理一次，这就是为什么我们可以直接在模板中使用 `ref` 值，而无需通过 `value` 属性。

## 第三章 渲染器

### 整体概述

渲染器是框架性能的核心。渲染器负责将虚拟 DOM 渲染到特定平台上（是跨平台的）。

```js
function createRenderer() {
  function render(vnode, container) {
    // ...
  }
  
  function hydrate(vnode, container) {
    // ...
  }
  
  return {
    render,
    hydrate
  }
}

const renderer = createRenderer();
// 首次渲染
renderer.render(oldVnode, document.querySelector('#app'));
// 第二次渲染
renderer.render(newVnode, document.querySelector('#app'));
// 第三次渲染
renderer.render(null, document.querySelector('#app'));
```

这里我们的 `createRenderer` 函数创建了一个渲染器，渲染器（renderer）和渲染（render）是有区别的，渲染器中包含了更多的功能（比如 hydrate 函数），而 `render` 只负责将虚拟 DOM 渲染到容器中。

我们首次渲染时，是没有 oldVnode 的，只有我们传入的新的 Vnode，这时我们需要执行「挂载」操作，将传入的 Vnode 挂载进容器中。

当我们第二次调用 render 函数时，这时，容器中会保存下来上一次的 Vnode，与本次传入的新的 Vnode 进行对比，力求最小量对 DOM 进行更新，所以这时我们需要执行「更新」操作。

当我们第三次调用 render 函数，传入的新的 Vnode 是 null，表示我们本次要执行的是一个「卸载」操作，需要对容器中的 DOM 进行卸载。

当然，我们其实是可以将「挂载」和「更新」这两个操作看做是一个操作，「挂载」是特殊的「更新」。

```js
function render(vnode, container) {
  // 传入了vnode，表示要进行更新操作
  if(vnode) {
    patch(container._vnode, vnode, container);
  }
  // 没传入vnode，表示需要卸载
  else {
    // 卸载逻辑
  }
  // 这次的vnode保存下来，下次再调用时使用
  container._vnode = vnode
}
```

### 挂载与更新

那么我们开始考虑 `patch` 函数：

```js
function patch(oldVnode, newVnode, container) {
  // 没有oldVnode，说明要挂载
  if(!oldVnode) {
    mountElement(newVnode, container);
  }
  // 要更新
  else {
    // 更新逻辑
  }
}
```

这里我们将所有对平台具体的操作抽离出去，这样我们可以针对不同的平台修改这些函数。

#### 挂载

我们可以考虑 `mountElement` 函数：

```js
function createRenderer(options) {
  // 将跨平台实现的函数传进来
  const {
    createElement,
    insert,
    setElementText
  } = options;
  
  function mountElement(vnode, container) {
    // 将真实的DOM元素保存下来，这样方便其他的操作
    const el = vnode.el = createElement(vnode.type);
    
    // 处理children
    // children属性为字符串，设置内部文本
    if(typeof vnode.children === 'string') {
      setElementText(el, vnode.children);
    }
    // 有子节点
    else if(Array.isArray(vnode.children)) {
      // 遍历子节点，对每一个子节点进行挂载，所以第一个参数是null
      vnode.children.forEach(child => patch(null, child, el));
    }
    
    // 处理props
    for(const key in vnode.props) {
      // 设置属性，这里我们可以选择直接设置，也可以通过setAttribute，这要看具体的情况（不多展开细节）
      el[key] = vnode.props[key];
    }
    
    // 将定制好的节点插入容器
    insert(el, container);
  }
  
  function patch(oldVnode, newVnode, container) {
    // ...
  }
  
  function render(vnode, container) {
    // ...
  }
  
  return {
    render
  }
}
```

#### 卸载

接着我们讨论「卸载」操作：

```js
function render(vnode, container) {
  if(vnode) {
    patch(contain._vnode, vnode, container);
  }
  // 没传入vnode，表示需要卸载
  else {
  	if(container._vnode) {
      // 封装进unmount函数
      unmount(container._vnode)
    }
  }
  container._vnode = vnode
}
```

```js
function unmount(vnode) {
  // 我们调用原生的卸载方法
  const parent = vnode.el.parentNode;
  if(parent) parent.removeChild(vnode.el);
}
```

由于我们想要卸载的元素，可能是组件，或者包含自定义指令，这时我们需要在执行 `unmount` 函数时，调用这些钩子函数。

#### 事件的处理

关于事件，我们需要考虑：如何在虚拟节点上描述事件、如何将事件添加在 DOM 上 以及 如何更新事件。

对于描述事件，我们可以把事件看做是特殊的，以 `on` 开头的属性。

将事件添加在 DOM 上，我们只需要通过 `addEventListener` 来绑定即可。

但是对于事件更新，如果我们需要更换事件，则需要 `removeEventListener`，再绑定新的事件。所以 Vue 采用了比较取巧的方式：绑定一个假的事件处理函数，在这个函数中调用真实的处理函数，这样我们就可以直接换掉真实的函数了。

```js
function patchProps(el, key, prevValue, nextValue) {
  if(/^on/.test(key)) {
    // 这个invoker就是我们的假的处理函数，保存在el._vei中
    const invokers = el._vei || (el._vei = {});
    let invoker = invokers[key];
    const name = key.slice(2).toLowerCase();
    if(nextValue) {
      // 之前没有绑定过这个事件
      if(!invoker) {
        invoker = el._vei[key] = (e) => {
          // 调用真正的处理函数
          invoker.value(e)
        }
        invoker.value = nextValue;
        el.addEventListener(name, invoker);
      } else {
        // 之前绑定过，直接换值就行
        invoker.value = newxtValue;
      }
    } else if (invoker) {
      // nextValue 没有值，但是invoker有值，说明需要移除事件
      el.removeEventListener(name, invoker)
    }
  }
  else if (key === 'class') {
    // ...
  }
  else {
    // ...
  }
}
```

#### 更新子节点

更新子节点时，我们首先区分子节点是不是只有文本：

- 如果 `vnode.children` 是字符串，那么说明元素有文本子节点。
- 如果 `vnode.children` 是数组，那么说明元素有多个子节点。

之所以要区分文本节点和子节点，是因为这样我们可以让更新子节点的逻辑更清晰。

那么一个节点的子节点有可能是：

- 没有子节点
- 文本子节点
- 一组子节点

由于有新旧节点之分，那么就是 3 * 3 = 9 种情况。

```js
function patchElement(n1, n2) {
  const el = n2.el = n1.el;
  const oldProps = n1.props;
  const newProps = n2.props;
  // 更新props
  for (const key in newProps) {
    // 更新属性
    if (newProps[key] !== oldProps[key]) {
      patchProps(el, key, oldProps[key], newProps[key]);
    }
  }
  for (const key in oldProps) {
    // 删除属性
    if (!(key in newProps)) {
      patchProps(el, key, oldProps[key], null);
    }
  }
  // 更新子节点的函数
  patchChildren(n1, n2, el);
}
```

```js
function patchChild(n1, n2, container) {
  // 新子节点是字符串
	if (typeof n2.children === 'string') {
    // 如果旧子节点是一组子节点，依次卸载
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c));
    }
    // 旧子节点是其他两种情况，直接换成新的文本即可
    setElementText(container, n2.children);
  }
  // 新子节点是一组子节点
  else if (Array.isArray(n2.children)) {
    // 旧子节点也是一组子节点
    if (Array.isArray(n1)) {
      // 这里就是核心的 diff 算法
      
    } else {
      // 旧子节点不是一组子节点，我们只需要清空之前的内容，依次挂载新的子节点
      setElementText(container, '');
      n1.children.forEach(c => patch(null, c, container));
    }
  }
  // 新子节点不存在
  else {
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c));
    }
    else if (typeof n1.children === 'string') {
      setElementText(container, '');
    }
  }
}
```

### Diff 算法

接下来我们就来看看渲染器中最核心的 diff 算法。

当新旧子节点都是一组子节点时，为了最小的性能开销完成更新，需要通过 diff 比较出两组节点的区别，然后最小量更新 DOM。

#### 简单 Diff 算法

我们可以对两组节点依次对比（不考虑节点仅顺序改变），如果有标签可以复用，我们就可以少操作一次 DOM。

```js
function patchChildren(n1, n2, container) {
  // ...
  else if (Array.isArray(n2)) {
    if (Array.isArray(n1)) {
      const oldChildren = n1.children;
      const newChildren = n2.children;
      const oldLen = oldChildren.length;
      const newLen = newChildren.length;
      // 找到较短的长度，作为公共长度
      const commonLen = Math.min(oldLen, newLen);
      for (let i = 0; i < commonLen; i++) {
        patch(oldChildren[i], newChildren[i]);
      }
      if (newLen > oldLen) {
        // 如果新节点更长，那么把新的节点挂载上去
        for (let i = commonLen; i < newLen; i++) {
          patch(null, newChildren[i], container);
        }
      } else if (oldLen > newLen) {
        for (let i = commonLen; i < oldLen; i++) {
          // 如果旧节点更长，就把长的部分卸载掉
          unmount(oldChildren[i]);
        }
      }
    }
  }
  // ...
}
```

那么我们可以很轻易地发现这种方法还有很大的优化空间，首先就是我们并没有考虑到顺序改变后 DOM 的复用。

有些时候，我们很可能只是改变了几个 DOM 节点的排列顺序，这时我们只需要移动这些 DOM 节点就可以了。这时我们需要能鉴别出，哪些 DOM 节点是相同的，这只靠 vnode.type 并不可靠：

```js
// oldChildren
{
  { type: 'p', clildren: '1' },
  { type: 'p', clildren: '2' },
  { type: 'p', clildren: '3' },
}
// newChildren
{
  { type: 'p', clildren: '2' },
  { type: 'p', clildren: '3' },
  { type: 'p', clildren: '1' },
}
```

这种情况我们只需要移动节点顺序，但是我们无法判断哪些节点是相同的节点，这就是我们为什么需要 key 属性。我们可以通过 key 的值来对节点进行鉴别。

要注意的是，即使是 key 值相同，也不意味着不需要进行 patch，因为新旧节点的值可能会改变，只是说这个 DOM 可以复用。

##### 更新节点

```js
function patchChildren(n1, n2, container) {
	if (typeof n2.children === 'string') {
		// ...
  } else if (Array.isArray(n2.children)) {
    const oldChildren = n1.children;
    const newChildren = n2.children;
    
    // 遍历新节点，寻找相同节点（先遍历新节点是为了避免有节点被移除）
    for (const newVNode of newChildren) {
      for (const oldVNode of oldChildren) {
        if (oldVNode !== newVNode) {
          // 找到相同节点，进行patch
					patch(oldVNode, newVNode, container);
          break; // 找到了，不需要再找了
        }
      }
    };
  } else {
		// ...
  }
};
```

这样我们就可以保证，所有的新旧节点都已经更新了，但是还没有移动到正确的顺序。因为我们是使用新节点依次对比旧节点，所以我们已经解决了挂载和卸载的问题。

##### 移动元素

我们可以开始考虑移动元素。首先我们需要找到要移动的元素。我们可以先思考什么时候不需要移动元素？答案是在节点顺序没有发生变化时，我们不需要移动元素。

```js
// oldChildren
{
  { key: 1 },
  { key: 2 },
  { key: 3 },
}

// newChildren1 和之前一样，不需要移动
{
  { key: 1 },
  { key: 2 },
  { key: 3 },
}

// newChildren2 少了一个，但是顺序没变，也不需要移动
{
  { key: 1 },
  { key: 3 },
}

// newChildren3 打乱顺序，需要移动
{
  { key: 3 },
  { key: 1 },
  { key: 2 },
}
```

那么我们就可以发现，重点在于 **「相对顺序」**，而不是简单的比较。

我们使用新节点（`newChildren1`）依次对比旧节点，可以发现新的元素在旧节点中的位置为：0, 1, 2。是一个**递增序列**，说明 **「相对顺序」** 没有改变，所以不需要移动。

我们还可以以同样的方法来看 `newChildren2`，序列为：0, 2。所以也不需要移动。

那么我们来看需要移动的 `newChildren3`：

- 第一个新节点对应旧节点中的位置是 2；

- 第二个新节点是 0，说明他与前一个节点的顺序不对，这个节点需要被移动；

- 第三个节点是 1，说明与第一个节点的顺序不对，也需要被移动；

我们再简单的总结一下这个算法：找到一个**「相对位置」**最大的索引值（也可以是暂时最大的，因为即使后面有更大的也不影响，说明前面的都比那个更大的小），后面如果有更大的，就更新这个最大索引值，如果更小，则说明这个节点需要移动。

```js
function patchChildren(n1, n2, container) {
	// ...
  else if (Array.isArray(n2)) {
    const oldChildren = n1.children;
    const newChildren = n2.children;
    
    // 最大索引
    let lastIndex = 0;
    for (let i = 0; i < newChilren.length; i++) {
      const newVNode = newChildren[i];
      for (let j = 0; j < oldChlidren.length; j++) {
        const oldVNode = oldChildren[j];
        if (newVNode.key === oldVNode.key) {
					patch(oldVNode, newVNode, container);
          if (j < lastIndex) {
            // 当前找到的索引值小于最大索引值，需要移动
          } else {
            // 找到的索引值不小于最大索引值，需要更新
            lastIndex = j
          }
          break;
        }
      };
    }
  } else {
    // ...
  }
};
```

在我们找到要移动的元素后，我们就可以思考怎么移动元素了。想要移动元素，我们需要拿到元素的引用，这在 patch 时，被放进了新节点的 el 属性上。

```js
function patchElement(n1, n2) {
	const el = n2.el = n1.el;
  // ...
};
```

随后我们考虑如何移动节点，依旧是 `newChildren3`：

```js
// oldChildren
{
  { key: 1 },
  { key: 2 },
  { key: 3 },

// newChildren3 打乱顺序，需要移动
{
  { key: 3 },
  { key: 1 },
  { key: 2 },
}
```

1. 取新节点中第一个 key 为 3 的虚拟节点，去找到在旧节点中的索引为 2，更新 `lastIndex` 为 2。
2. 取下一个节点 key 为 1，找到所在索引为 0，需要移动。将它移动到 `key = 3` 节点的后面。
3. 再取下一个节点 key 为 2，找到所在索引为 1，需要移动。将它移动到 `key = 1` 节点的后面。

更新完成。我们可以发现，**新节点们的顺序，就是我们希望移动的顺序**。所以对于每个需要移动的节点来说，只需要**把它放在他在新节点中的前一个节点的后面**就行了（因为新节点顺序没错，所以在前一个节点的后一个节点，看似是废话）。

那么我们看下根据这个思路实现的代码：

```js
function patchChildren(n1, n2, container) {
  // ...
  else if (Array.isArray(n2)) {
    const oldChildren = n1.children;
    const newChildren = n2.children;
    
    // 最大索引
    let lastIndex = 0;
    for (let i = 0; i < newChilren.length; i++) {
      const newVNode = newChildren[i];
      for (let j = 0; j < oldChlidren.length; j++) {
        const oldVNode = oldChildren[j];
        if (newVNode.key === oldVNode.key) {
					patch(oldVNode, newVNode, container);
          if (j < lastIndex) {
            // 当前找到的索引值小于最大索引值，需要移动
            // 拿到 newVNode 的前一个节点，他对应 DOM 节点后面就是正确的位置
            const prevVNode = newVNode[i - 1];
            // 如果 prevVNode 不存在，说明是第一个，不需要移动
            if (prevVNode) {
              // 由于我们需要移动真实 DOM 节点，所以需要找到它的下一个兄弟节点，放在他下一个兄弟的前面
              const anchor = prevVNode.el.nextSibling;
              // 移动节点，这个函数跨平台实现，在浏览器是由 insertBefore 实现
              insert(newVNode.el, container, anchor);
            }
          } else {
            lastIndex = j;
          }
          break;
        }
      };
    } 
  } else {
    // ...
  }
};
```

##### 新增元素

如果 `newChildren` 中有新添加的节点，在 `oldChildren` 中找不到索引。这时我们需要挂载新的节点，并将它放在合适的位置。

```js
function patchChildren(n1, n2, container) {
  // ...
  else if (Array.isArray(n2)) {
    const oldChildren = n1.children;
    const newChildren = n2.children;
    
    let lastIndex = 0;
    for (let i = 0; i < newChilren.length; i++) {
      const newVNode = newChildren[i];
      // 设定一个变量指示有没有找到匹配的索引
      let find = false;
      for (let j = 0; j < oldChlidren.length; j++) {
        const oldVNode = oldChildren[j];
        if (newVNode.key === oldVNode.key) {
          find = true;
					patch(oldVNode, newVNode, container);
          if (j < lastIndex) {
            // 当前找到的索引值小于最大索引值，需要移动
            // 拿到 newVNode 的前一个节点，他对应 DOM 节点后面就是正确的位置
            const prevVNode = newVNode[i - 1];
            // 如果 prevVNode 不存在，说明是第一个，不需要移动
            if (prevVNode) {
              // 由于我们需要移动真实 DOM 节点，所以需要找到它的下一个兄弟节点，放在他下一个兄弟的前面
              const anchor = prevVNode.el.nextSibling;
              // 移动节点，这个函数跨平台实现，在浏览器是由 insertBefore 实现
              insert(newVNode.el, container, anchor);
            }
          } else {
            lastIndex = j;
          }
          break;
        }
      };
      // 找完一个新节点，来看看这个节点是不是新节点。find 为 false 表示这个节点需要挂载
      if (!find) {
        // 找到前一个节点的下一个兄弟节点作为锚点
				const prevVNode = newChildren[i - 1];
        let anchor = null;
        // 如果有前置节点，说明不是第一个
        if (prevVNode) {
          anchor = prevVNode.el.nextSibling;
        }
        // 说明是第一个节点，使用第一个元素作为锚点
        else {
          anchor = container.firstChild;
        }
        // 挂载节点，这里需要指定挂载位置
        patch(null, newVNode, container, anchor);
      }
    } 
  } else {
    // ...
  }
};
```

由于我们需要新的 `patch` 函数，所以我们再修改一下

```js
// patch 函数接受第四个参数，即锚点元素
function patch() {
  // ...
  if (typeof type === 'string') {
    if (!n1) {
      // 挂载，将锚点元素传过去
      mountElement(n2, container, anchor);
    } else {
      patchElement(n1, n2);
    }
  }
  // ...
}

function mountElement(vnode, container, anchor) {
	// ...
  insert(el, container, anchor);
}
```

##### 移除元素

如果新的节点被删除了，那么旧节点会遗留下来。我们只需要在新节点都移动处理完后，再遍历一遍旧节点，删除其中的不存在的节点即可。

```js
function patchChildren(n1, n2, container) {
  // ...
  else if (Array.isArray(n2)) {
    const oldChildren = n1.children;
    const newChildren = n2.children;
    
    let lastIndex = 0;
    for (let i = 0; i < newChilren.length; i++) {
      // ...
    } 
    
    // 处理完，再处理删除的节点
    for (let i = 0; i < oldChildren.length; i++) {
      const oldVNode = oldChildren[i];
			const has = newChildren.find(vnode => vnode.key === oldVNode);
      if (!has) {
        // 如果在newChildren中找不到，那么删除该节点
        unmount(oldVNode);
      }
    }
  } else {
    // ...
  }
};
```

#### 双端 Diff 算法

简单 Diff 算法已经解决了问题，但是仍然有一些缺陷，简单 Diff 算法对 DOM 的移动操作并不是最优的。比如：

```
新节点					旧节点
p--3          p--1
p--1          p--2
p--2          p--3
```

 在这种情况下，简单 Diff 算法会让 p--1 节点和 p--2 节点移动到 p--3 节点的后面，但其实我们只需要将 p--3 节点移动到最前就可以了。

##### 原理

双端 Diff 算法同时对新旧两组节点的两个端点同时进行比较，这里会提供四个指针，分别指向四个端点。

![新旧节点及真实 DOM 状态.jpg](https://s2.loli.net/2022/05/17/y3pbe4t8aldI5DJ.png)

每一轮都会进行四次比较（中间的灰色箭头），如果有某次发现节点可以复用（第四次比较，发现 p-4 节点可以复用），那么移动对应的节点（第四次比较将 oldEndIdx 对应的节点，移动到 oldStartIdx 节点之前），随后指针接着移动。

![第一次移动后状态.jpg](https://s2.loli.net/2022/05/17/QPHqYFCLw9mWlBc.png)

代码如下：

```js
function patchKeyChild(n1, n2, container) {
  const oldChildren = n1.children;
  const newChildren = n2.children;
  // 四个索引值
  let oldStartIdx = 0;
  let oldEndIdx = oldChildren.length - 1;
  let newStartIdx = 0;
  let newEndIdx = oldChildren.length - 1;
  // 四个索引值指向的 vnode 节点
  let oldStartVNode = oldChildren[oldStartIdx];
  let oldEndVNode = oldChildren[oldEndIdx];
  let newStartVNode = newChildren[newStartIdx];
  let newEndVNode = newChildren[newEndIdx];
  
  // 循环进行双端比较，直到有一组结束
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (oldStartVNode.key === newStartVNode.key) {
      // 第一次比较，新旧节点起始互相比较
    } else if (oldEndVNode.key === newEndVNode.key) {
      // 第二次比较，新旧节点末尾互相比较
    } else if (oldStartVNode.key === newEndVNode.key) {
      // 第三次比较，新节点末尾与旧节点起始互相比较
    } else if (oldEndVNode.key === newStartVNode.key) {
      // 第四次比较，旧节点末尾与新节点起始互相比较
      
      // 假设第四次找到了可复用的节点，需要将 oldEnd，移动到 oldStart 前面
      // 调用 patch 进行打补丁
      patch(oldEndVNode, newStartVNode, container);
      // 移动 DOM
      insert(oldEndVNode.el, container, oldStartVNode.el);
      // 更新索引
      oldEndVNode = oldChildren[--oldEndIdx];
      newStartVNode = newChildren[++newStartIdx];
    }
  }
}
```

剩余三种情况的操作也和第四种情况类似，只是移动的元素不同。

直到至少有一方的索引汇合，双端比较结束：

![双端比较结束后状态.jpg](https://s2.loli.net/2022/05/28/VFjmRqdLZwCc9fY.png)

##### 非理想情况

在理想情况下，比如：

```
新节点					旧节点
p--3          p--1
p--1          p--2
p--2          p--3
```

我们会发现，双端 Diff 只需要移动一次，而简单 Diff 需要移动两次。

但是这是在能找到可复用节点的情况，如果是这种情况：

```
新节点					旧节点
p--2          p--1
p--4          p--2
p--1          p--3
p--3					p--4
```

我们就会发现，第一轮比较中，一个可复用的节点都没有，这时我们就需要另外的处理方式：我们会拿新一组节点的头部节点去旧一组中寻找：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  if (oldStartVNode.key === newStartVNode.key) {
    // 第一次比较，新旧节点起始互相比较
  } else if (oldEndVNode.key === newEndVNode.key) {
    // 第二次比较，新旧节点末尾互相比较
  } else if (oldStartVNode.key === newEndVNode.key) {
    // 第三次比较，新节点末尾与旧节点起始互相比较
  } else if (oldEndVNode.key === newStartVNode.key) {
    // 第四次比较，旧节点末尾与新节点起始互相比较
  } else {
    // 四次比较都没有发现有可复用节点
    // 使用新节点头部去找对应的索引
    const idxInOld = oldChildren.findIndex(node => node.key === newStartVNode.key);
  }
}
```

我们找到了头部节点 `p--2` 对应的索引 1，这说明，原来索引 1 的节点，现在应该排在头部，我们可以将它移动到头部。

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  if (oldStartVNode.key === newStartVNode.key) {
    // ...
  } else if (oldEndVNode.key === newEndVNode.key) {
    // ...
  } else if (oldStartVNode.key === newEndVNode.key) {
    // ...
  } else if (oldEndVNode.key === newStartVNode.key) {
    // ...
  } else {
    // 使用新节点头部去找对应的索引
    const idxInOld = oldChildren.findIndex(node => node.key === newStartVNode.key);
    if (idxInOld >= 0) {
      // 要移动的节点
      const vnodeToMove = oldChildren[idxInOld];
      // 打补丁
      patch(vnodeToMove, newStartVNode, container);
      // 移动到最前面，即旧节点的最前面
      insert(vnodeToMove, container, oldStartVNode.el);
      // 因为这个节点我们已经处理过了，真实 DOM 已经移到了正确位置，设置为 undefined
      oldChildren[idxInOld] = undefined;
      // 更新 newStart
      newStartVNode = newChildren[++newStartIdx];
    }
  }
}
```

接着双端 Diff 继续比较，直到遇见我们之前设置过的 `undefined`，说明这个节点我们处理过了。我们还需要添加两个处理分支来判断一下这种情况：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  if (!oldStartIdx) { // 如果头部节点被处理过，跳过
    oldStartVNode = oldChildren[++oldStartIdx];
  } else if (!oldEndIdx) { // 如果尾部节点被处理过，跳过
    oldEndVNode = oldChildren[--oldEndIdx];
  } else if (oldStartVNode.key === newStartVNode.key) {
    // ...
  } else if (oldEndVNode.key === newEndVNode.key) {
    // ...
  } else if (oldStartVNode.key === newEndVNode.key) {
    // ...
  } else if (oldEndVNode.key === newStartVNode.key) {
    // ...
  } else {
    const idxInOld = oldChildren.findIndex(node => node.key === newStartVNode.key);
    if (idxInOld >= 0) {
      const vnodeToMove = oldChildren[idxInOld];
      patch(vnodeToMove, newStartVNode, container);
      insert(vnodeToMove, container, oldStartVNode.el);
      oldChildren[idxInOld] = undefined;
      newStartVNode = newChildren[++newStartIdx];
    }
  }
}
```

 这样，非理想情况下的问题也解决了。

##### 添加新元素

如果新节点中有在旧节点中找不到的，那么就是新节点，需要挂载到正确的位置去。

![新增节点的情况.jpg](https://s2.loli.net/2022/05/28/2EPLOVjYAwdkv5S.png)

在一轮比较后，我们发现没有可复用的节点，所以试图找到 newStart 对应的旧索引，然后我们发现找不到，说明这是一个新增的节点。

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  if (!oldStartIdx) {
    oldStartVNode = oldChildren[++oldStartIdx];
  } else if (!oldEndIdx) {
    oldEndVNode = oldChildren[--oldEndIdx];
  } else if (oldStartVNode.key === newStartVNode.key) {
    // ...
  } else if (oldEndVNode.key === newEndVNode.key) {
    // ...
  } else if (oldStartVNode.key === newEndVNode.key) {
    // ...
  } else if (oldEndVNode.key === newStartVNode.key) {
    // ...
  } else {
    const idxInOld = oldChildren.findIndex(node => node.key === newStartVNode.key);
    if (idxInOld >= 0) {
      const vnodeToMove = oldChildren[idxInOld];
      patch(vnodeToMove, newStartVNode, container);
      insert(vnodeToMove, container, oldStartVNode.el);
      oldChildren[idxInOld] = undefined;
    } else { // 说明节点是新增的节点
      // 挂载到头部
      patch (null, newStartVNode, container, oldStartVNode.el);
    }
    newStartVNode = newChildren[++newStartIdx];
  }
}
```

但是这样做其实还有遗漏，如果我们改变一下新节点顺序，让新节点能找到可复用节点：

![新增节点的情况-遗漏.jpg](https://s2.loli.net/2022/05/28/8kozcHCmXONqLDF.png)

最终比较结果会变成，旧节点已经结束，但是新节点的 `p--4` 被遗漏下来了。所以我们还需要再进行修补：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  // 进行比较
}
// 在比较结束后，检查一下剩余的节点
if (oldEndIdx < oldStartIdx && newStartIdx <= newEndIdx) {
  // 如果旧节点已经结束，但是新节点还有剩余，说明需要挂载新节点
  for (let i = newStartIdx; i <= newEndIdx; i++) {
    // 将剩余的新增节点按顺序挂载到头部
    patch(null, newChildren[i], container, oldStartVNode.el);
  }
}
```

##### 移除不存在的元素

解决了新增节点的问题，我们再看移除节点的问题：

![移除节点.jpg](https://s2.loli.net/2022/05/28/ld3ozsjYVremgwF.png)

当新节点结束时，旧节点还剩下一个 `p--2` 节点：

![移除节点-结束.jpg](https://s2.loli.net/2022/05/28/9h7ciyFOAKEajZp.png)

所以我们还需要新增一段逻辑来移除旧节点:

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  // 进行比较
}

if (oldEndIdx < oldStartIdx && newStartIdx <= newEndIdx) {
  // 添加新节点
} else if (newEndIdx < newStartIdx && oldStartIdx <= oldEndIdx) {
  // 移除旧节点
  for (let i = oldStartIdx; i <= oldEndIdx; i++) {
    unmount(oldChildren[i]);
  }
}
```

双端 Diff 算法，就已经完整了。

#### 快速 Diff 算法

Vue 2 采用的是双端 Diff 算法，而 Vue 3 则换成了快速 Diff 算法。快速 Diff 算法顾名思义，他比双端 Diff 算法更快。



## 第四章 组件化

### 组件的实现原理

上一章讲的是渲染器，渲染器负责将虚拟 DOM 渲染为真实 DOM。当我们编写 Vue 时，实际上就是在编写组件化的虚拟 DOM。这些组件构成了整个 Vue 应用。

#### 渲染组件

从用户的角度看，一个组件是一个对象：

```js
const MyComponent = {
	name: 'MyComponent',
  data() {
		return {
      foo: 1,
    }
  },
}
```

但是从渲染器的角度看，一个组件是一个虚拟 DOM 树。

```js
const vnode = {
  // 使用 type 存储组件的选项对象，即用户编写的对象
  type: MyComponent,
  // ...
}
```

但是这时，我们的 `patch` 函数还没有办法接受组件类型，我们需要增加一个分支判断：

```js
function patch(n1, n2, container, anchor) {
	if (n1 && n1.type !== n2.type) {
    unmount(n1);
    n1 = null;
  }
  const { type } = n2;
  if (typeof type === 'string') {
		// 作为普通元素处理
  } else if (type === Text) {
    // 作为文本节点处理
  } else if (type === Fragment) {
		// 作为片段处理
  } else if (typeof type === 'object') {
    // 新增，作为组件处理
    if (!n1) {
      // 挂载
      mountComponent(n2, container, anchor);
    } else {
      // 更新组件
      patchComponent(n1, n2, anchor);
    }
  }
}
```

当渲染器有能力处理组件后，我们要设计组件应该被如何编写。组件需要描述一段页面结构，所以组件必须有一个渲染函数，即 `render` 函数。这个 `render` 函数还应该返回虚拟 DOM，让渲染器可以将虚拟 DOM 渲染为真实 DOM。

```js
const MyComponent = {
  name: 'MyComponent',
  render() {
    // 返回虚拟节点
    return {
      type: 'div',
      children: '文本内容',
    }
  },
}
```

这样，我们就可以将组件传递给渲染器，让渲染器完成渲染：

```js
// 应用这个组件时，用来描述该组件的虚拟节点
const CompVNode = {
  type: MyComponent
}

renderer.render(CompVNode, document.querySelector('#app'))
```

`renderer.render` 函数会调用 `mountComponent` 来真正的完成渲染：

```js
function mountComponent(vnode, container, anchor) {
  // 拿到用户的组件配置项
  const componentOptions = vnode.type
  // 拿到其中的渲染函数
  const { render } = componentOptions
  // 调用渲染函数得到虚拟 DOM 树
  const subTree = render()
  // 渲染
  patch(null, subTree, container, anchor)
}
```

这样，我们就得到了基本的组件化方案。

#### 组件状态与自更新

实际上在 `render` 中，引用 `data` 中的数据是很常见的，我们必须让组件能够具备读取自身状态，而且能自动更新。

```js
const MyComponent = {
  data() {
    return {
      foo: 'hello world'
    }
  },
  render() {
    return {
      type: 'div',
      children: `foo 的值为: ${this.foo}`, // 在渲染函数中使用组件状态
    }
  },
}
```

这里的渲染函数，试图获取 data 中的数据，我们需要能让他通过 this 访问到：

```js
function mountComponent(vnode, container, anchor) {
  const componentOptions = vnode.type
  const { render, data } = componentOptions
  // 使用 reactive 函数初始化 data，将它变成响应式数据
  const state = reactive(data())
  // 通过 state 调用 render
  const subTree = render.call(state, state)
  patch(null, subTree, container, anchor)
}
```

我们再来分析组件自更新。我们需要在 `state` 变化时，重新调用 `render` 函数和 `patch` 函数。我们可以将这两个 副作用函数 用 `effect` 包裹执行。

```js
function mountComponent(vnode, container, anchor) {
  const componentOptions = vnode.type
  const { render, data } = componentOptions
  const state = reactive(data())
  // 通过副作用函数执行，将这两个函数添加到依赖中去
  effect(() => {
    const subTree = render.call(state, state)
  	patch(null, subTree, container, anchor)
  })
}
```

为了避免多次修改状态，导致组件不断更新，我们可以将组件更新放进异步队列里，这样可以对多次更新任务去重，避免反复更新组件。

```js
// 任务缓存队列，用一个 set 来表示，用来去重
const queue = new Set()
// 标志正在刷新任务队列
let isFlushing = false
const p = Promise.resolve()

// 调度器，让每一次事件循环都只有一次更新
function queueJob(job) {
	queue.add(job)
  // 如果还没有开始刷新队列，刷新它。如果已经开始刷新，则不需要再添加微任务
  if (!isFlushing) {
    isFlushing = true
    p.then(() => {
      try {
        queue.forEach(job => job())
      } finally {
        // 执行完全部的任务后，可以接受新的任务了
        isFlushing = false
        queue.length = 0
      }
    })
  }
}
```

然后我们可以在 `effect` 函数中使用调度器

```js
function mountComponent(vnode, container, anchor) {
  const componentOptions = vnode.type
  const { render, data } = componentOptions
  
  const state = reactive(data())
  
  effect(() => {
    const subTree = render.call(state, state)
  	patch(null, subTree, container, anchor)
  }, {
    // 指定调度器
    scheduler: queueJob
  })
}
```

但是，上面这段代码还有缺陷，就是每次 patch 时，传入的都是 null ，每次都是进行挂载，而不是打补丁。为了实现打补丁，我们需要实现组件实例，用它来维护组件的整个生命周期的状态。

#### 组件实例与生命周期

组件实例本质上就是一个对象，维护着组价运行过程中的所有信息，例如注册到组件的生命周期函数、组件渲染的子树（subTree）、组件是否已经被挂载、组件自身的状态（data），等等。

```js
function mountComponent(vnode, container, anchor) {
	const componentOptions = vnode.type
  const { render, data } = componentOptions
  
  const state = reactive(data())
  
  // 定义组件实例，一个组件实例就是一个对象，包含组件有关的状态信息
  const instance = {
    // 组件自身的状态数据，即 data
    state,
    // 组件是否被挂载，初始值为 false
    isMounted: false,
    // 组件所渲染的内容，即子树（subTree）
    subTree: null,
  }
  
  // 将组件实例放在 vnode 上，用于以后更新
  vnode.component = instance
  
  effect(() => {
    // 调用组件的渲染函数，获得子树
    const sbTree = render.call(state, state)
    // 检查组件是否已经被挂载
    if (!instance.isMounted) {
      // 初次挂载
      patch(null, subTree, container, anchor)
      // 将组件设置为已挂载，这样以后就会进行更新，而不是挂载
      instance.isMounted = true
    } else {
      // 更新组件
      patch(instance.subTree, subTree, container. anchor)
    }
    // 更新组件实例的子树
    instance.subTree = subTree
  }, { scheduler: queueJob })
}
```

这样，我们就可以在合适的时机调用组件的声明周期钩子：

```js
function mountComponent(vnode, container, anchor) {
	const componentOptions = vnode.type
  // 拿到对应的钩子
  const { render, data, beforeCreate, created, beforeMount, mounted, beforeUpdate, updated } = componentOptions
  
  beforeCreate && beforeCreate()
  const state = reactive(data())
  
  const instance = {
    state,
    isMounted: false,
    subTree: null,
  }
  
  vnode.component = instance
  
  // 调用时允许 this 访问组件状态
  created && created.call(state)
  
  effect(() => {
    const sbTree = render.call(state, state)
    if (!instance.isMounted) {
      beforeMount && beforeMount.call(state)
      patch(null, subTree, container, anchor)
      instance.isMounted = true
      mounted && mounted()
    } else {
      beforeUpdate && beforeUpdate.call(state)
      patch(instance.subTree, subTree, container. anchor)
      updated && updated.call(state)
    }
    instance.subTree = subTree
  }, { scheduler: queueJob })
}
```

大概的原理就是这样。

#### props 与组件的被动更新

在 Vue 3 中，被子组件指定接受的 `prop`，会被传递到 `props` 中，而没指定，但是父组件传递进来的，会被传递到 `attrs` 中。我们在构建组件时，还需要考虑这两部分的数据。

```js
function mountComponent(vnode, container, anchor) {
	const componentOptions = vnode.type
  // 取出用户配置的 props
  const { render, data, props: propsOption, /* 省略其他 */ } = componentOptions
  
  beforeCreate && beforeCreate()
  const state = reactive(data())
  // 解析 props 和 attrs
  const [props, attrs] = resolveProps(propsOption, vnode.props)
  
  const instance = {
    state,
    isMounted: false,
    subTree: null,
  }
  
  vnode.component = instance
  // ...
}

// 解析组件的 props
function resolveProps(options, propsData) {
  const props = []
  const attrs = []
  for (const key in propsData) {
    // 遍历传入的组件属性，如果被组件定义，放入 props
    if (key in options) {
      props[key] = propsData[key]
    } else {
      // 没被组件定义，放到 attrs
      attrs[key] = propsData[key]
    }
  }
  return [props, attrs]
}
```

具体实现上还有一些细节，比如默认值，类型校验...

处理完 `props`，还要考虑当 `props` 改变时，父组件会先重新渲染，随后触发 `patchComponent` 来更新子组件，我们将这种更新叫做子组件的被动更新。当子组件被动更新时，我们需要：

- 检测子组件是不是真的要更新，即 `props` 是不是真的改变了
- 如果要更新，则更新子组件的 `props` 和 `slot` 等内容

`patchComponent` 实现如下：

```js
function patchComponent(n1, n2, anchor) {
  // 获取组件实例，即 n1.component，同时让新的组件虚拟节点 n2.component 也指向组件实例
  const instance = (n2.component = n1.component)
  // 获取当前的 props 数据
  const { props } = instance
  if (hasPropsChanged(n1.props, n2.props)) {
    // 如果 props 改变了，拿到新的 props
    const [nextProps] = resolveProps(n2.type.props, n2.props)
    // 更新 props
    for (const k in nextProps) {
      props[k] = nextProps[k]
    }
    // 删除不存在的 props
    for (const k in props) {
      if (!(k in nextProps)) delete props[k]
    }
  }
}

function hasPropsChanged(prevProps, nextProps) {
	const nextKeys = Object.keys(nextProps)
  // 如果数量变了，说明有变化
  if (nextKeys.length !== Object.keys(prevProps).length) {
    return true
  }
  for (let i = 0; i < nextKeys.length; i++) {
    const key = nextKeys[i]
    // 有不相等的 props，说明有变化
    if (nextProps[key] !== prevProps[key]) return true
  }
  return false
}
```

这里的 `props` 对象是浅响应（`shallowReactive`）的，因此只需要设置 instance.props 下的属性即可触发子组件重新渲染。

#### setup 的作用与实现

setup 是 Vue 3 新增的组件选项，用来配合组合式 API。他可以有两种返回值：

- 返回一个函数，该函数将作为该组件的 render 函数：

  ```js
  const Comp = {
    setup() {
  		return () => {
        return { type: 'div', children: 'hello' }
      }
    }
  }
  ```

  一般用来描述不好用模板实现的内容，比如多层嵌套的重复内容

- 返回一个对象，这个对象中的数据将暴露给模板使用：

  ```js
  const Comp = {
    setup() {
      const count = ref(0)
      return {
        count,
      }
    },
    render() {
      return { type: 'div', children: `count is ${this.count}` }
    }
  }
  ```

  

另外，setup 接受两个参数：props、context。

```js
const Comp = {
  props: {
    foo: String,
  },
  setup(props, context) {
		props.foo // 访问 props 数据
    // context 保存了与组件接口相关的数据和方法
    const { slot, emit, attrs, expose } = context
  }
}
```

context 对象包括：

- `slots`：组件接受的插槽。
- `emit`：用来发射事件。
- `attrs`：没被定义，但是被传入的属性。
- `expose`：一个函数，用来显示的暴露组件数据。

在 Vue 3 中，更支持使用 组合式 API。

#### 组件事件与 emit 的实现

在 Vue 中，我们可以使用 `emit` 函数来发射组件自定义的事件。

```jsx
const MyComponent = {
  name: 'MyComponent',
  setup(props, { emit }) {
    // 发出自定义 change 事件，并传递两个参数
		emit('change', 1, 2)
    
    return () => {
			return // ...
    }
  },
}

// 使用
<MyComponent @change="handler"></MyComponent>
// 上面的模板被编译生成的 vnode 为：
const CompVNode = {
	type: MyComponent,
  props: {
		onChange: handler,
  }
}
```

Vue 3 将事件编译为 以 `on` 开头的属性，存储在 `props` 中。具体实现则是根据事件名称去寻找对应的事件处理函数并执行。

```js
function mountComponent(vnode, container, anchor) {
  //...
  const instance = {
    state,
    props: shallowReactive(props),
    isMounted: false,
    subTree: null,
  }
  // 定义 emit 函数
  function emit(event, ...payload) {
    // 将事件转换为 on 开头 change -> onChange
		const eventName = `on${event[0].toUpperCase() + event.slice(1)}`
    // 去寻找事件处理函数
    const handler = instance.props[eventName]
    if (handler) {
      handler(...payload)
    } else {
			console.log('事件不存在')
    }
  }
  // 将 emit 函数添加到 context 中，提供给 setup
  const setupContext = { attrs, emit }
  // ...
}
```

但是现在由于事件没有在 props 中声明过，会被 `resolveProps` 放入 `attrs` 中，导致我们找不到事件属性，所以我们还需要接着优化一下：

```js
function resolveProps(options, propsData) {
  const props = {}
  const attrs = {}
  for (const key of propsData) {
    // 以字符串 on 开头的 props，无论是否显示地声明，都将它添加到 props
    if (key in options || key.startsWith('on')) {
      props[key] = propsData[key]
    } else {
      attrs[key] = propsData[key]
    }
  }
  return [props, attrs]
}
```

#### 插槽的工作原理与实现

插槽 是组件预留的一个槽位，具体渲染的内容由调用者传入。

```vue
<template>
	<header><slot name="header" /></header>
	<div><slot name="body" /></div>
	<footer><slot name="footer" /></footer>
</template>
```

父组件可以传入对应的内容：

```vue
<template>
	<MyComponent>
  	<template #header>
			<h1>我是标题</h1>
		</template>
		<template #body>
			<section>我是内容</section>
		</template>
		<template #footer>
			<p>我是注脚</p>
		</template>
  </MyComponent>
</template>
```

这段父组件模板会被编译成如下的渲染函数：

```js
function render() {
  return {
    type: MyComponent,
    // 组件的 children 会被编译成一个对象
    children: {
			header() {
				return { type: 'h1', children: '我是标题' }
      },
      body() {
        return { type: 'section', children: '我是内容' }
      },
      footer() {
				return { type: 'p', children: '我是注脚' }
      }
    }
  }
}
```

而有插槽的子组件则会被编译为：

```js
// MyComponent 组件的编译结果
function render() {
  // 没有根元素，返回的是一个数组
  return [
    {
      type: 'header',
      children: [this.$slot.header()]
    },
    {
      type: 'body',
      children: [this.$slot.body()]
    },
    {
      type: 'footer',
      children: [this.$slot.footer()]
    }
  ]
}
```

我们可以发现，插槽的工作原理和 `render` 函数十分类似，就是调用插槽函数获得渲染内容。在运行时的实现上，插槽则依赖于 `setupContext` 中的 `slots` 对象，如下代码所示：

```js
function mountComponent(vnode, container, anchor) {
  // 省略部分代码
  
  // 直接使用编译好的 vnode.children 对象作为 slots 对象即可
  const slots = vnode.children || {}
  
  const setupContext = { attrs, emit, slots }
}
```

最简单的 `slot` 实现非常简单，就是将编译器编译好的 `children` 属性作为 `slot` 的值。

#### 注册生命周期

在 Vue 中，我们还可以自己注册生命周期钩子。

```js
import { onMounted } from 'vue'

const MyComponent = {
  setup() {
    onMounted(() => {
      console.log('mounted 1')
    })
    // 可以注册多个
    onMounted(() => {
      console.log('mounted 2')
    })
    // ...
  }
}
```

为了能让 `onMounted` 函数能够注册到对应组件上，我们需要维护一个全局变量 `currentInstance`，来保存当前的组件实例。每当初始化组件时，我们先将它设置为当前组价实例，再执行 `setup` 函数，这样我们就可以通过 `currentInstance` 来获取当前正在初始化的组件实例。

```js
let currentInstance = null

function setCurrentInstance(instance) {
	currentInstance = instance
}

function mountComponent(vnode, container, anchor) {
  // ...
  
  const instance = {
    state,
    props: shallowReactive(props),
    isMounted: false,
    subTree: null,
    slots,
    // 在组件实例中添加 mounted 数组，用来储存 onMounted 函数注册的生命周期钩子函数
    mounted: []
  }
  
  // ...
  
  const setupContext = { attrs, emit, slots }
  
  // 执行 setup 之前，设置当前组件实例
  setCurrentInstance(instance)
  // 执行 setup
  const setupResult = setup(shallowReadonly(instance.props), setupContext)
  // 重置当前组件实例
  setCurrentInstance(null)
}
```

以 mounted 钩子为例，我们只需要在注册时，将函数推入数组中，就可以实现多次注册了。

```js
function onMounted(fn) {
  if (currentInstance) {
    currentInstance.mounted.push(fn)
  } else {
    console.log('onMounted 函数只能在 setup 中调用')
  }
}
```

最后，我们需要在合适的时机来调用这些钩子：

```js
function mountComponent(vnode, container, anchor) {
  // ...
  effect(() => {
    const subTree = render.call(render(renderContext, renderContext))
    if (!instance.isMounted) {
      // ...
      
      // 遍历执行钩子函数
      instance.mounted && instance.mounted.forEach(hook => hook.call(renderContext))
    } else {
      // ...
    }
    instance.subTree = subTree
  }, {
    sheduler: queueJob
  })
}
```

其他钩子函数原理是一样的。
