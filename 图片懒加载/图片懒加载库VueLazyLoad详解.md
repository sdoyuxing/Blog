# 背景

上篇[《图片懒加载原理方案详解》](https://juejin.cn/post/7196970992576397367#heading-14)中详细解析了图片懒加载的原理和方案。主流方案有两个：

-   监听页面的滚动事件判断图片元素的 top 是否在页面可视区内。
-   通过 IntersectionObserver 监听元素是否进去可视区域内。

实现过程中注意优化的点：

-   避免重复设置 src。
-   滚动事件监听添加节流。
-   加载图片的时间点可以适当提前。
-   避免出现布局抖动。
-   响应式图片要额外处理。
-   seo 不友好的问题。

本文将通过vue-lazyload总结和学习图片懒加载其他优化细节。

# 说明

[vue-lazyload](https://github.com/hilongjw/vue-lazyload)是基于 vue 下的图片懒加载库，使用方法如下

```html
<ul>
    <li v-for="img in list">
        <img v-lazy="img.src" />
    </li>
</ul>
```

# 实现原理

vue-lazyload 两个方案都采用了，通过 observer 配置属性来切换方案，`getBoundingClientRect` 方式的实现很细节而 `IntersectionObserver` 的实现相对比较简单。

## 1. placeholder 的实现很细致和灵活

placeholder 是图片加载之前的为了保证用户体验而设置的一个默认图片，提示用户图片正在加载。vue-lazyload 将 placeholder 拆分为加载中显示的图片 loading 和加载图片出错显示的图片 error，并且现在JavaScript 中创建 Image 对象把要加载的图片下载下来后再修改 img 标签的 src 属性，这样能避免直接修改 src 下载图片的过程中页面 img 标签显示空白的情况。

```javascript
this.renderLoading(() => {
    this.attempt++;

    this.options.adapter["beforeLoad"] &&
        this.options.adapter["beforeLoad"](this, this.options);
    this.record("loadStart");

    loadImageAsync(
        {
            src: this.src,
            cors: this.cors,
        },
        (data) => {
            this.naturalHeight = data.naturalHeight;
            this.naturalWidth = data.naturalWidth;
            this.state.loaded = true;
            this.state.error = false;
            this.record("loadEnd");
            this.render("loaded", false);
            this.state.rendered = true;
            this._imageCache.add(this.src);
            onFinish();
        },
        (err) => {
            !this.options.silent && console.error(err);
            this.state.error = true;
            this.state.loaded = false;
            this.render("error", false);
        }
    );
});
```

这是加载的主要代码，renderLoading 函数是加载 loading 图片。

```javascript
  /*
   * render loading first
   * @params cb:Function
   * @return
   */
  renderLoading (cb) {
    this.state.loading = true
    loadImageAsync({
      src: this.loading,
      cors: this.cors
    }, data => {
      this.render('loading', false)
      this.state.loading = false
      cb()
    }, () => {
      // handler `loading image` load failed
      cb()
      this.state.loading = false
      if (!this.options.silent) console.warn(`VueLazyload log: load failed with loading image(${this.loading})`)
    })
  }
```

`loadImageAsync` 函数是创建 `Image` 对象加载对应的图片，`render` 方法是将 img 标签设置 src。

```javascript
const loadImageAsync = (item, resolve, reject) => {
    let image = new Image();
    if (!item || !item.src) {
        const err = new Error("image src is required");
        return reject(err);
    }

    image.src = item.src;
    if (item.cors) {
        image.crossOrigin = item.cors;
    }

    image.onload = function () {
        resolve({
            naturalHeight: image.naturalHeight,
            naturalWidth: image.naturalWidth,
            src: image.src,
        });
    };

    image.onerror = function (e) {
        reject(e);
    };
};
```

```javascript
   /**
         * set element attribute with image'url and state
         * @param  {object} lazyload listener object
         * @param  {string} state will be rendered
         * @param  {bool} inCache  is rendered from cache
         * @return
         */
    _elRenderer (listener, state, cache) {
      if (!listener.el) return
      const { el, bindType } = listener

      let src
      switch (state) {
        case 'loading':
          src = listener.loading
          break
        case 'error':
          src = listener.error
          break
        default:
          src = listener.src
          break
      }

      if (bindType) {
        el.style[bindType] = 'url("' + src + '")'
      } else if (el.getAttribute('src') !== src) {
        el.setAttribute('src', src)
      }

      el.setAttribute('lazy', state)

      this.$emit(state, listener, cache)
      this.options.adapter[state] &&
                this.options.adapter[state](listener, this.options)

      if (this.options.dispatchEvent) {
        const event = new CustomEvent(state, {
          detail: listener
        })
        el.dispatchEvent(event)
      }
    }
```

## 2. 添加图片缓存

声明 `_imageCache` 记录加载过的图片，图片已经缓存到了浏览器很快就能显示出来，就不用创建 Image 对象来加载图片。

```javascript
if (this._imageCache.has(this.src)) {
    this.state.loaded = true;
    this.render("loaded", true);
    this.state.rendered = true;
    return onFinish();
}
```

但如果使用过程中用户清空浏览器图片缓存，`_imageCache` 仍有记录就会出现 img 元素加载图片时显示空白。

## 3. 事件监听使用节流

vue-lazyload 使用的节流不是平时常用的节流，而是防抖和节流相结合，节流频率变化的方式。

```javascript
function throttle(action, delay) {
    let timeout = null;
    let movement = null;
    let lastRun = 0;
    let needRun = false;
    return function () {
        needRun = true;
        if (timeout) {
            return;
        }
        let elapsed = Date.now() - lastRun;
        let context = this;
        let args = arguments;
        let runCallback = function () {
            lastRun = Date.now();
            timeout = false;
            action.apply(context, args);
        };
        if (elapsed >= delay) {
            runCallback();
        } else {
            timeout = setTimeout(runCallback, delay);
        }
        if (needRun) {
            clearTimeout(movement);
            movement = setTimeout(runCallback, 2 * delay);
        }
    };
}
```

delay 默认是 200ms，throttle 返回的函数第一次执行会马上执行 action，第二次执行如果在 200ms 内就会推迟 200ms 执行中途不再执行 action，而如果在 200ms 间隔以外就马上执行 action。最后超过 400ms 后再最后执行一次，这种写法很少见。

## 4. 监听事件不止滚动事件

vue-lazyload 除了监听 scroll 事件还默认监听了，还监听了 wheel、mousewheel、resize、animationend、transitionend、touchmove。

- 监听 wheel、mousewheel 事件是因为在自定义滚动条下需要监听滚轮事件。

- 监听 resize、animationend、transitionend 事件是因为页面大小可能会改变导致可视区域的范围改变而需要重新计算判断元素是否在可视区域内。

- touchmove 是在手机端下需要监听的事件。

监听事件考虑的很细，但默认监听这么多事件会导致性能的消耗，根据条件判断来加对应的监听事件会更好些。

## 5. 事件列队的方式来处理懒加载

每个需要懒加载的元素都会生成一个 ReactiveListener 对象放到 ListenerQueue 队列中，ReactiveListener 对象包含判断元素是否在可视区域，加载图片等系列操作。触发滚动事件时会遍历 ListenerQueue 队列中每一个 ReactiveListener 对象是否需要加载图片。

这种方式代码结构清晰职责分明，扩展性好。添加一个懒加载元素只需在队列中添加一个对象。缺点是元素的 dom 对象一直存在队列中没有释放，只有组件销毁才能会释放。在懒加载图片很多的情况下性能不是很好。

## 6. 支持 data-srcset

> HTMLImageElement 的 srcset 的值是一个字符串，用来定义一个或多个图像候选地址，以 , 分割，每个候选地址将在特定条件下得以使用。候选地址包含图片 URL 和一个可选的宽度描述符和像素密度描述符，该候选地址用来在特定条件下替代原始地址成为 src(en-US) 的属性。 ——MDN

srcset 的作用是可以根据页面的宽度来加载不同的图片。
vue-lazyload 不是简单的把 data-srcset 赋值给 img 的 srcset 属性，而是用 JavaScript 代码来实现 srcset 的效果。之所以用 JavaScript 代码来实现是为了图片加载中能更精细的控制 loading 和 error 的图片显示。

```javascript
function getBestSelectionFromSrcset(el, scale) {
    if (el.tagName !== "IMG" || !el.getAttribute("data-srcset")) return;

    let options = el.getAttribute("data-srcset");
    const result = [];
    const container = el.parentNode;
    const containerWidth = container.offsetWidth * scale;

    let spaceIndex;
    let tmpSrc;
    let tmpWidth;

    options = options.trim().split(",");

    options.map((item) => {
        item = item.trim();
        spaceIndex = item.lastIndexOf(" ");
        if (spaceIndex === -1) {
            tmpSrc = item;
            tmpWidth = 999998;
        } else {
            tmpSrc = item.substr(0, spaceIndex);
            tmpWidth = parseInt(
                item.substr(spaceIndex + 1, item.length - spaceIndex - 2),
                10
            );
        }
        result.push([tmpWidth, tmpSrc]);
    });

    result.sort(function (a, b) {
        if (a[0] < b[0]) {
            return 1;
        }
        if (a[0] > b[0]) {
            return -1;
        }
        if (a[0] === b[0]) {
            if (b[1].indexOf(".webp", b[1].length - 5) !== -1) {
                return 1;
            }
            if (a[1].indexOf(".webp", a[1].length - 5) !== -1) {
                return -1;
            }
        }
        return 0;
    });
    let bestSelectedSrc = "";
    let tmpOption;

    for (let i = 0; i < result.length; i++) {
        tmpOption = result[i];
        bestSelectedSrc = tmpOption[1];
        const next = result[i + 1];
        if (next && next[0] < containerWidth) {
            bestSelectedSrc = tmpOption[1];
            break;
        } else if (!next) {
            bestSelectedSrc = tmpOption[1];
            break;
        }
    }

    return bestSelectedSrc;
}
```

## 7. 自定义控制可视区的判定范围

在计算 img 元素是否在可视区域内时，通过 preLoad 来设置判定区的大小。

```javascript
  /*
   *  check el is in view
   * @return {Boolean} el is in view
   */
  checkInView () {
    this.getRect()
    return (this.rect.top < window.innerHeight * this.options.preLoad && this.rect.bottom > this.options.preLoadTop) &&
            (this.rect.left < window.innerWidth * this.options.preLoad && this.rect.right > 0)
  }
```

# 待完善

## 1. 没有解决布局抖动

库中没有 css 的封装，可能作者的开发初衷就是不想封装 css，让使用者自己处理 css，相关布局抖动的描述可以去看[《图片懒加载原理方案详解》](https://juejin.cn/post/7196970992576397367#heading-14)。

## 2. 跳过已经加载图片的判断方式

源码中是通过 rendered 和 loaded 来判断图片是否已经加载过，并没有在 ListenerQueue 队列中销毁已经加载过的事件对象。

## 3. 局部懒加载

没有考虑页面局部滚动条内图片的懒加载情况。

## 4. 性能不是很好

性能方面消耗包括绑定比较多频繁触发的事件，ListenerQueue 队列中的事件没有做对应的销毁，图片比较多情况下 ListenerQueue 队列中的事件堆积比较多。

## 5. observer 模式配置简单

observer 模式模式下 IntersectionObserver 初始化代码。

```javascript
   /**
    * init IntersectionObserver
    * set mode to observer
    * @return
    */
    _initIntersectionObserver () {
      if (!hasIntersectionObserver) return
      this._observer = new IntersectionObserver(this._observerHandler.bind(this), this.options.observerOptions)
      if (this.ListenerQueue.length) {
        this.ListenerQueue.forEach(listener => {
          this._observer.observe(listener.el)
        })
      }
    }
```

IntersectionObserver 的配置通过 options 中的 observerOptions 直接配置，相比 event 模式的配置过于简单，没有对 IntersectionObserver api 一些常用的功能进行封装，比如自定义可视区域的判断范围，需要查找 IntersectionObserver api 中的配置项 rootMargin 才能实现。如果 observer 模式下的 也加个一个 preLoad 配置，就不用额外再去找 IntersectionObserver api 文档。

## 6. SEO 不友好

开始 img 的 src 是值是空的，后面加载的时候才设置 src，搜索引擎就抓取不到图片。

# 总结

vue-lazyload 在用户体验上处理的很细节，从配置项中有个 attempt 图片加载失败次数上看出考虑的很细腻，但在大量图片懒加载时性能会差点，IntersectionObserver 的封装有点匆忙。
