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

我们使用新节点（`newChildren1`）依次对比旧节点，可以发现新的元素在旧节点中的位置为：0, 1, 2。是一个递增序列，说明 **「相对顺序」** 没有改变，所以不需要移动。

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

