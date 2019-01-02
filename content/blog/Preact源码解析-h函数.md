---
title: Preact源码解析-h函数
date: '2018-01-02T20:16:11.284Z'
---

从基本的preact.js看起。

Preact使用`h()`来把JSX转换为虚拟DOM elements。

```javascript
import { VNode } from  './vnode'; // VNode构造，是一个空构造函数
import  options  from  './options'; // options对dom，vdom转换的一些配置项，初始为空

const stack = [];

const EMPTY_CHILDREN = [];

export function h(nodeName, attributes) {

	// nodeName可以认为是type，对应的是span，div等
	// attributes对应dom的attributes，例如id，className(class)

	let children=EMPTY_CHILDREN, lastSimple, child, simple, i;

	// 构建JSX的参数为(type, attributes, ...children),这里把children都push到stack中
	for (i=arguments.length; i-- > 2; ) {
		stack.push(arguments[i]);
	}

	// 判定attribute是否存在，同样将children push到stack中，完毕后删除
	if (attributes && attributes.children!=null) {
		if (!stack.length) stack.push(attributes.children);
		delete attributes.children;
	}

	// 执行迭代
	while (stack.length) {

		// 如果是数组，把child从children数组中单独push，相当于对嵌套子节点做了一次flatten
		if ((child = stack.pop()) && child.pop!==undefined) {
			for (i=child.length; i--; ) stack.push(child[i]);
		}

		// 对数组类型做区分
		else {

			// boolean型转为null
			if (typeof child==='boolean') child = null;

			// 如果不是函数，可能是其他类型
			// 把null，undefined转为''
			// 把number转为String(number)
			// Symbol, Object则将simple置为false
			if ((simple = typeof nodeName!=='function')) {
				if (child==null) child = '';
				else if (typeof child==='number') child = String(child);
				else if (typeof child!=='string') simple = false;
				// 注意这里，在对这些类型判断完毕以后，如果simple === false，则有两种可能
				// 1.simple是function
				// 2.simple是Symbol或者Object
			}

			// 分情况判断
			// 1. 首次执行，则lastSimple肯定为undefined，且children===EMPTY_CHILDREN成立
			// 2. 第二次执行，主要区别是下面的判定，可以总结为simple和lastSimple都为true
			// 满足这个条件的，就是上一次和这一次的child为string、null、boolean、number、undefined即可
			if (simple && lastSimple) {
				// 上面会把这些类型转为字符串，这里使用简单的+=就将字符串拼接到一起
				// 减少了children的数量，有助于提高性能
				children[children.length-1] += child;
			}
			else if (children===EMPTY_CHILDREN) {
				// 首次执行后，这里切换了children指向，之后这个条件将不会成立
				children = [child];
			}
			else {
				children.push(child);
			}

			lastSimple = simple;
		}
	}

	// VNode暂时还没有做什么事情，单纯记录一些信息
	let p = new VNode();
	p.nodeName = nodeName;
	p.children = children;
	p.attributes = attributes==null ? undefined : attributes;
	p.key = attributes==null ? undefined : attributes.key;

	// if a "vnode hook" is defined, pass every created VNode to it
	// 如果在options里定义了vnode hook，就执行
	if (options.vnode!==undefined) options.vnode(p);

	return p;
}

```

需要明确，`h()` 不负责JSX语法转为JavaScript对象，通常这件事是由babel去做，`h()`负责把JSX生成的JavaScript对象处理为VNode。

也有一个[htm库](https://github.com/developit/htm)，可以在浏览器端将JSX转为JavaScript对象，然后再render为html。

---
