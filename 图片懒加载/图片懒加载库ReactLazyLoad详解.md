# 背景

上篇[《图片懒加载库 VueLazyLoad 详解》](https://juejin.cn/post/7201831345416159292)中详细解析了 VueLazyLoad 实现细节和侧重点。

- getBoundingClientRect 和 IntersectionObserver 两种方式都实现了，侧重 getBoundingClientRect 方式，IntersectionObserver 简单封装了下。
- 在图片懒加载的用户体验上考虑的很细腻。
- 性能处理上不是很好。

本文将通过 react-lazyload 解析来看看另一个侧重思路的封装。

# 说明

react-lazyload 用于延迟加载组件、图像或任何影响性能的东西。

```javascript
import React from "react";
import ReactDOM from "react-dom";
import LazyLoad from "react-lazyload";
import MyComponent from "./MyComponent";

const App = () => {
  return (
    <div className="list">
      <LazyLoad height={200}>
        <img src="tiger.jpg" /> /*
      </LazyLoad>
      <LazyLoad height={200} once>
        <MyComponent />
      </LazyLoad>
      <LazyLoad height={200} offset={100}>
        <MyComponent />
      </LazyLoad>
      <LazyLoad>
        <MyComponent />
      </LazyLoad>
    </div>
  );
};

ReactDOM.render(<App />, document.body);
```

# 特点

react-lazyload 懒加载只采用了 getBoundingClientRect 方式来实现，并没有使用 IntersectionObserver 方式，具体详情请看[《图片懒加载原理方案详解》](https://juejin.cn/post/7196970992576397367)。

## 1. 组件懒加载

react-lazyload 定位为组件的懒加载，图片只是其中一种，可以是任何影响性能的组件。

```javascript
  render() {
    const {
      height,
      children,
      placeholder,
      className,
      classNamePrefix,
      style
    } = this.props;

    return (
      <div className={`${classNamePrefix}-wrapper ${className}`} ref={this.setRef} style={style}>
        {this.visible ? (
          children
        ) : placeholder ? (
          placeholder
        ) : (
          <div
            style={{ height: height }}
            className={`${classNamePrefix}-placeholder`}
          />
        )}
      </div>
    );
  }
```

## 2. 绑定的监听事件

react-lazyload 默认绑定 scroll 事件不绑定 resize 事件，可以通过配置项 scroll 和 resize 配置是否绑定对应事件。这里和 vue-lazyload 尽量考虑全各种情况不同，默认只绑定 scroll 事件，连 resize 事件都是默认不绑定。比 vue-lazyload 绑定很多事件性能会好些，从这可以看出 react-lazyload 考虑常规环境使用，不想花性能去处理小概率的不常规环境。

## 3. 局部懒加载

通过配置项 overflow 设置是否局部懒加载。

```javascript
/**
 * Check if `component` is visible in overflow container `parent`
 * @param  {node} component React component
 * @param  {node} parent    component's scroll parent
 * @return {bool}
 */
const checkOverflowVisible = function checkOverflowVisible(component, parent) {
  const node = component.ref;

  let parentTop;
  let parentLeft;
  let parentHeight;
  let parentWidth;

  try {
    ({
      top: parentTop,
      left: parentLeft,
      height: parentHeight,
      width: parentWidth,
    } = parent.getBoundingClientRect());
  } catch (e) {
    ({
      top: parentTop,
      left: parentLeft,
      height: parentHeight,
      width: parentWidth,
    } = defaultBoundingClientRect);
  }

  const windowInnerHeight =
    window.innerHeight || document.documentElement.clientHeight;
  const windowInnerWidth =
    window.innerWidth || document.documentElement.clientWidth;

  // calculate top and height of the intersection of the element's scrollParent and viewport
  const intersectionTop = Math.max(parentTop, 0); // intersection's top relative to viewport
  const intersectionLeft = Math.max(parentLeft, 0); // intersection's left relative to viewport
  const intersectionHeight =
    Math.min(windowInnerHeight, parentTop + parentHeight) - intersectionTop; // height
  const intersectionWidth =
    Math.min(windowInnerWidth, parentLeft + parentWidth) - intersectionLeft; // width

  // check whether the element is visible in the intersection
  let top;
  let left;
  let height;
  let width;

  try {
    ({ top, left, height, width } = node.getBoundingClientRect());
  } catch (e) {
    ({ top, left, height, width } = defaultBoundingClientRect);
  }

  const offsetTop = top - intersectionTop; // element's top relative to intersection
  const offsetLeft = left - intersectionLeft; // element's left relative to intersection

  const offsets = Array.isArray(component.props.offset)
    ? component.props.offset
    : [component.props.offset, component.props.offset]; // Be compatible with previous API

  return (
    offsetTop - offsets[0] <= intersectionHeight &&
    offsetTop + height + offsets[1] >= 0 &&
    offsetLeft - offsets[0] <= intersectionWidth &&
    offsetLeft + width + offsets[1] >= 0
  );
};
```

## 4. 事件监听采用 passive event

react-lazyload 事件绑定还用了 passive event 来性能优化。
passive 是 addEventListener 第三个参数对象的一个属性：

> options 可选
> 
> 一个指定有关 listener 属性的可选参数对象。可用的选项如下：
> 
> passive 可选
> 
> 一个布尔值，设置为 true 时，表示 listener 永远不会调用 preventDefault()。如果 listener 仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告。查看使用 passive 改善滚屏性能以了解更多。
> 
> 根据规范，addEventListener() 的 passive 默认值始终为 false。然而，这引入了触摸事件和滚轮事件的事件监听器在浏览器尝试滚动页面时阻塞浏览器主线程的可能性——这可能会大大降低浏览器处理页面滚动时的性能。
> 
> 为了避免这一问题，大部分浏览器（Safari 和 Internet Explorer 除外）将文档级节点 Window、Document 和 Document.body 上的 wheel、mousewheel、touchstart 和 touchmove 事件的 passive 默认值更改为 true。如此，事件监听器便不能取消事件，也不会在用户滚动页面时阻止页面呈现。
> ——MDN

简单说因为浏览器不知道事件监听函数中会不会有 preventDefault 阻止事件默认行为，所以浏览器会等到事件监听函数执行完了确定没有 preventDefault 才会执行事件的默认行为。passive:true 就是实现告诉浏览器事件监听函数中不会有 preventDefault 事件，事件的默认行为的执行就不会等到事件监听函数执行完再执行。从而避免卡顿的效果。
vue-lazyload 也使用了这种方式绑定事件监听。

## 5. 事件防抖和节流

通过配置 debounce/throttle 来使用防抖/节流。

```javascript
export default function debounce(func, wait, immediate) {
  let timeout;
  let args;
  let context;
  let timestamp;
  let result;

  const later = function later() {
    const last = +new Date() - timestamp;

    if (last < wait && last >= 0) {
      timeout = setTimeout(later, wait - last);
    } else {
      timeout = null;
      if (!immediate) {
        result = func.apply(context, args);
        if (!timeout) {
          context = null;
          args = null;
        }
      }
    }
  };

  return function debounced() {
    context = this;
    args = arguments;
    timestamp = +new Date();

    const callNow = immediate && !timeout;
    if (!timeout) {
      timeout = setTimeout(later, wait);
    }

    if (callNow) {
      result = func.apply(context, args);
      context = null;
      args = null;
    }

    return result;
  };
}
```

```javascript
/*eslint-disable */
export default function throttle(fn, threshhold, scope) {
  threshhold || (threshhold = 250);
  var last, deferTimer;
  return function () {
    var context = scope || this;

    var now = +new Date(),
      args = arguments;
    if (last && now < last + threshhold) {
      // hold on to it
      clearTimeout(deferTimer);
      deferTimer = setTimeout(function () {
        last = now;
        fn.apply(context, args);
      }, threshhold);
    } else {
      last = now;
      fn.apply(context, args);
    }
  };
}
```

## 6. 事件列队的方式来处理懒加载

懒加载组件对象会放入 listeners 数组中，事件监听触发循环遍历 listeners 数组来判断是否要加载。
这种方式代码结构清晰职责分明，扩展性好。添加一个懒加载元素只需在队列中添加一个对象。缺点是元素的 dom 对象一直存在队列中没有释放，只有组件销毁才能会释放。react-lazyload 添加一个 once 配置是否加载完后释放对象。

## 7. 自定义控制可视区的判定范围

在计算是否在可视区域内时，通过 offset 来设置判定区的大小。

```javascript
/**
 * Check if `component` is visible in document
 * @param  {node} component React component
 * @return {bool}
 */
const checkNormalVisible = function checkNormalVisible(component) {
  const node = component.ref;

  // If this element is hidden by css rules somehow, it's definitely invisible
  if (!(node.offsetWidth || node.offsetHeight || node.getClientRects().length))
    return false;

  let top;
  let elementHeight;

  try {
    ({ top, height: elementHeight } = node.getBoundingClientRect());
  } catch (e) {
    ({ top, height: elementHeight } = defaultBoundingClientRect);
  }

  const windowInnerHeight =
    window.innerHeight || document.documentElement.clientHeight;

  const offsets = Array.isArray(component.props.offset)
    ? component.props.offset
    : [component.props.offset, component.props.offset]; // Be compatible with previous API

  return (
    top - offsets[0] <= windowInnerHeight &&
    top + elementHeight + offsets[1] >= 0
  );
};
```

# 待完善

react-lazyload 定位组件懒加载，因此他在处理图片懒加载上是按照组件懒加载统一处理，在细节和用户体验上并没有单独处理，和 vue-lazyload 侧重点相反。

1. placeholder 简单的实现
2. 没有 IntersectionObserver 实现方式
3. 不支持图片响应式 srcset 属性的懒加载
4. 没有缓存处理
5. SEO 不友好

具体描述可以查看[《图片懒加载库 VueLazyLoad 详解》](https://juejin.cn/post/7201831345416159292)

# 总结

和 VueLazyLoad 对比两者侧重点刚好相反：

- ReactLazyLoad 侧重性能和功能的抽象扩展。
- VueLazyLoad 侧重图片懒加载的用户体验细节。
