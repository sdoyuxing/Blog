## 背景
继续 **你不知道的 Popperjs 上篇** 来解析 Popperjs。
## 版本
**0.5.2**
## 实现原理

### 保存原有的 DOM 上下文

popper 元素是 absolute（绝对定位）：绝对定位的元素的位置相对于最近的已定位父元素，如果元素没有已定位的父元素，那么它的位置相对于根元素。

Popperjs 并没有改变 popper（定位元素）的 html 结构位置，所以 popper 元素的 top 和 left 是相对已定位父元素的位置。当父元素不是根元素的时候之前计算 popper 的 top 和 left 就会有偏差，多出 offsetParent（相对已定位父元素）的 top 和 left。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/197ab39366ce4f89adfe9ca006f53847~tplv-k3u1fbpfcp-watermark.image?)

计算 popper 位置的时候要减去它的已定位的父元素 offsetParent 的偏移量。

```js
  /**
     * Given an element and one of its parents, return the offset
     * 给定一个元素和他们的一个父元素，返回偏移量
     * @function
     * @ignore
     * @param {HTMLElement} element
     * @param {HTMLElement} parent
     * @return {Object} rect
     */
    function getOffsetRectRelativeToCustomParent(element, parent, fixed) {
        var elementRect = getBoundingClientRect(element);
        var parentRect = getBoundingClientRect(parent);
      
        var rect = {
            top: elementRect.top - parentRect.top,
            left: elementRect.left - parentRect.left,
            bottom: (elementRect.top - parentRect.top) + elementRect.height,
            right: (elementRect.left - parentRect.left) + elementRect.width,
            width: elementRect.width,
            height: elementRect.height
        };
        return rect;
    }
```
获取 popper 最近的已定位父元素

```js
/**
 * Returns the offset parent of the given element
 * 返回给定元素的定位父元素
 * @function
 * @ignore
 * @argument {Element} element
 * @returns {Element} offset parent
 */
function getOffsetParent(element) {
  // NOTE: 1 DOM access here
  var offsetParent = element.offsetParent;
  //判断父元素是否为body，如果是返回dom中的root 节点，如果不是返回父元素
  return offsetParent === window.document.body || !offsetParent
    ? window.document.documentElement
    : offsetParent;
}
```

> **`HTMLElement.offsetParent`** 是一个只读属性，返回一个指向最近的（指包含层级上的最近）包含该元素的定位元素或者最近的 `table`, `td`, `th`, `body` 元素。当元素的 `style.display` 设置为 "none" 时，`offsetParent` 返回 `null`。`offsetParent` 很有用，因为 [`offsetTop`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetTop "offsetTop") 和 [`offsetLeft`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetLeft "offsetLeft") 都是相对于其内边距边界的。
> 
> 元素的 `offsetParent` 可能值：`null`，`body` 元素，该元素的某个父级定位元素。
> 
> 为 `null` 的情况：`body` 元素、元素的 `display` 为 `none` 、元素尚未添加到 `DOM`、元素的 `position` 为 `fixed`。
> 
> 为 `body` 元素的情况：该元素不是任何一个定位元素的后代，也不是 `null`。

popper 元素 display 为 none，那么 `popper.offsetPatent` 值为null，`getOffsetParent` 返回的是`window.document.documentElement` ，也就是相对根元素计算偏移量，那么计算的偏移量似乎会有偏差。
写个例子验证下：

```html
      <div  id="scroll" style="position: relative; top: 100px; height: 200px;" >
        <button  id="reference"  style="position: relative; top: 100px; left: 100px" >
          reference
        </button>
        <div id="popper" style="background-color: red; width: 300px; height: 200px; display: none;" >
          popper
        </div>
      </div>
```

```js
     document.getElementById("reference").onclick = function () {
        new Popper(document.getElementById("reference"), document.getElementById("popper"),
          { placement: "left-start", gpuAcceleration: false }
        );
        document.getElementById("popper").style.display="block"
      };
```
第一次点击按钮效果如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b54b4ef53ae1440c904ee8c8f8c9a467~tplv-k3u1fbpfcp-watermark.image?)

第二次点击按钮才能准确定位：

![1671705517215.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0001d180ac544087830caa09bd776d66~tplv-k3u1fbpfcp-watermark.image?)

这里配置了gpuAcceleration为false，在试下启用css3 硬件加速试试。

```html
      <div  id="scroll" style="position: relative; top: 100px; height: 200px;" >
        <button  id="reference"  style="position: relative; top: 100px; left: 100px" >
          reference
        </button>
        <div id="popper" style="background-color: red; width: 300px; height: 200px; display: none;" >
          popper
        </div>
      </div>
```

```js
     document.getElementById("reference").onclick = function () {
        new Popper(document.getElementById("reference"), document.getElementById("popper"),
          { placement: "left-start", gpuAcceleration: true }
        );
        document.getElementById("popper").style.display="block"
      };
```
也是有同样的问题。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b54b4ef53ae1440c904ee8c8f8c9a467~tplv-k3u1fbpfcp-watermark.image?)

这个问题如果不注意的话会经常出现，而为什么 elementUI 的 Tooltip 组件没有这个问题呢？
因为 Tooltip 组件把提示元素（也就是 popper 元素）放置在 body 下面，所以提示元素一直是相对根元素定位的。

因此 Popperjs 在保持原有 DOM 上下文的特性上是有 bug 的，解决也不难，就是在获取 popper 尺寸的地方一起获取 offsetParent 就可以了，这个问题先记下来，看看后面迭代有没有修复这个问题。

判断已定位父元素是否是 body，是的话返回文档根元素。之所以这么处理是因为 `getBoundingClientRect` 用在根元素会有一些兼容性问题。

- `document.documentElement.getBoundingClientRect()` 是获取窗口位置，但在IE8以下返回的 `top`、`left` 值为 -2px，因为 IE8 以下浏览器是以窗口的 (2, 2) 坐标为原点坐标的。这里坐标原点的位置不影响算相对距离。
- 浏览器默认窗口和 body 有 8px 的间隙，所以使用 `document.body.getBoundingClientRect()` 得到的`top`、`left` 值为 8px，因此如果 `offsetParent` 是 `document.body` 要换成`document.documentElement` 来获取位置。

调用 `getOffsetRectRelativeToCustomParent` 获取 popper 已定位父元素的偏移量。

```js
 var referenceOffsets = getOffsetRectRelativeToCustomParent(reference, getOffsetParent(popper));
```
完整代码：

```js
/**
 * Get bounding client rect of given element
 * 获取给定元素绑定的窗口矩形
 * @function
 * @ignore
 * @param {HTMLElement} element
 * @return {Object} client rect
 */
function getBoundingClientRect(element) {
  var rect = element.getBoundingClientRect();
  return {
    left: rect.left,
    top: rect.top,
    right: rect.right,
    bottom: rect.bottom,
    width: rect.right - rect.left,
    height: rect.bottom - rect.top,
  };
}

/**
 * Given an element and one of its parents, return the offset
 * 给定一个元素和他们的一个父元素，返回偏移量
 * @function
 * @ignore
 * @param {HTMLElement} element
 * @param {HTMLElement} parent
 * @return {Object} rect
 */
function getOffsetRectRelativeToCustomParent(
  element,
  parent,
  fixed,
  transformed
) {
  var elementRect = getBoundingClientRect(element);
  var parentRect = getBoundingClientRect(parent);

  var rect = {
    top: elementRect.top - parentRect.top,
    left: elementRect.left - parentRect.left,
    bottom: elementRect.top - parentRect.top + elementRect.height,
    right: elementRect.left - parentRect.left + elementRect.width,
    width: elementRect.width,
    height: elementRect.height,
  };
  return rect;
}

/**
 * Set the style to the given popper
 * 给指定的popper设置样式
 * @function
 * @ignore
 * @argument {Element} element - Element to apply the style to
 * @argument {Object} styles - Object with a list of properties and values which will be applied to the element
 */
function setStyle(element, styles) {
  function is_numeric(n) {
    return n !== "" && !isNaN(parseFloat(n)) && isFinite(n);
  }
  Object.keys(styles).forEach(function (prop) {
    var unit = "";
    // add unit if the value is numeric and is one of the following
    if (
      ["width", "height", "top", "right", "bottom", "left"].indexOf(prop) !==
        -1 &&
      is_numeric(styles[prop])
    ) {
      unit = "px";
    }
    element.style[prop] = styles[prop] + unit;
  });
}
/**
 * Returns the offset parent of the given element
 * 返回给定元素的定位父元素
 * @function
 * @ignore
 * @argument {Element} element
 * @returns {Element} offset parent
 */
function getOffsetParent(element) {
  // NOTE: 1 DOM access here
  var offsetParent = element.offsetParent;
  //判断父元素是否为body，如果是返回html，如果不是返回父元素
  return offsetParent === window.document.body || !offsetParent
    ? window.document.documentElement
    : offsetParent;
}

/**
 * Get the prefixed supported property name
 * 获取支持属性名称的前缀
 * @function
 * @ignore
 * @argument {String} property (camelCase)
 * @returns {String} prefixed property (camelCase)
 */
function getSupportedPropertyName(property) {
  var prefixes = ["", "ms", "webkit", "moz", "o"];

  for (var i = 0; i < prefixes.length; i++) {
    var toCheck = prefixes[i]
      ? prefixes[i] + property.charAt(0).toUpperCase() + property.slice(1)
      : property;
    if (typeof window.document.body.style[toCheck] !== "undefined") {
      return toCheck;
    }
  }
  return null;
}

Popper.prototype.modifiers = {};
/**
 * Apply the computed styles to the popper element
 * 将计算属性应用于popper元素
 * @method
 * @memberof Popper.modifiers
 * @argument {Object} data - The data object generated by `update` method
 * @returns {Object} The same data object
 */
Popper.prototype.modifiers.applyStyle = function (data) {
  // apply the final offsets to the popper
  // 将最后的偏移量应用于popper元素
  // NOTE: 1 DOM access here
  // 注意:这里有1个DOM访问
  var styles = {
    position: data.offsets.popper.position,
  };

  // round top and left to avoid blurry text
  // 四舍五入top和left以避免模糊文本
  var left = Math.round(data.offsets.popper.left);
  var top = Math.round(data.offsets.popper.top);

  // if gpuAcceleration is set to true and transform is supported, we use `translate3d` to apply the position to the popper
  // 如果gpuAcceleration设置为true并且transform支持，我们用translate3d应用于popper位置上
  // we automatically use the supported prefixed version if needed
  // 如果有需要，我们自动使用支持的前缀版本
  var prefixedProperty;
  if (
    this._options.gpuAcceleration &&
    (prefixedProperty = getSupportedPropertyName("transform"))
  ) {
    styles[prefixedProperty] = "translate3d(" + left + "px, " + top + "px, 0)";
    styles.top = 0;
    styles.left = 0;
  }
  // othwerise, we use the standard `left` and `top` properties
  //否则我们使用标准的left和top属性
  else {
    styles.left = left;
    styles.top = top;
  }

  // any property present in `data.styles` will be applied to the popper,
  // data.styles 中的每一个实时属性都将被应用到popper元素上
  // in this way we can make the 3rd party modifiers add custom styles to it
  // 用这种方法我们可以用第三方修饰器添加自定义的样式
  // Be aware, modifiers could override the properties defined in the previous
  // 了解到，修饰符可以覆盖调之前定义的属性
  // lines of this modifier!
  Object.assign(styles, data.styles);

  setStyle(this._popper, styles);

  // set an attribute which will be useful to style the tooltip (use it to properly position its arrow)
  // 是在一个属性。将有助于设置工具提示的样式
  // NOTE: 1 DOM access here
  // 注意：一个访问DOM的地方
  this._popper.setAttribute("x-placement", data.placement);

  return data;
};

/**
 * Given the popper offsets, generate an output similar to getBoundingClientRect
 * @function
 * @ignore
 * @argument {Object} popperOffsets
 * @returns {Object} ClientRect like output
 */
function getPopperClientRect(popperOffsets) {
  var offsets = Object.assign({}, popperOffsets);
  offsets.right = offsets.left + offsets.width;
  offsets.bottom = offsets.top + offsets.height;
  return offsets;
}

/**
 * Modifier used to shift the popper on the start or end of its reference element side
 * 被用于设置popper放在reference的方位的修饰符
 * @method
 * @memberof Popper.modifiers
 * @argument {Object} data - The data object generated by `update` method
 * @returns {Object} The data object, properly modified
 */
Popper.prototype.modifiers.shift = function (data) {
  var placement = data.placement;
  var basePlacement = placement.split("-")[0];
  var shiftVariation = placement.split("-")[1];

  // if shift shiftVariation is specified, run the modifier
  if (shiftVariation) {
    var reference = data.offsets.reference;
    var popper = getPopperClientRect(data.offsets.popper);

    var shiftOffsets = {
      y: {
        start: { top: reference.top },
        end: { top: reference.top + reference.height - popper.height },
      },
      x: {
        start: { left: reference.left },
        end: { left: reference.left + reference.width - popper.width },
      },
    };

    var axis = ["bottom", "top"].indexOf(basePlacement) !== -1 ? "x" : "y";

    data.offsets.popper = Object.assign(
      popper,
      shiftOffsets[axis][shiftVariation]
    );
  }

  return data;
};

/**
 * Get the outer sizes of the given element (offset size + margins)
 * 获取给定元素的外部尺寸（偏移量+边距）
 * @function
 * @ignore
 * @argument {Element} element
 * @returns {Object} object containing width and height properties
 */
function getOuterSizes(element) {
  // NOTE: 1 DOM access here
  // 注意:这里有1个DOM访问
  var _display = element.style.display,
    _visibility = element.style.visibility;
  element.style.display = "block";
  element.style.visibility = "hidden";
  var calcWidthToForceRepaint = element.offsetWidth;

  // original method
  // 原始的方法
  var styles = window.getComputedStyle(element);
  var x = parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
  var y = parseFloat(styles.marginLeft) + parseFloat(styles.marginRight);
  var result = {
    width: element.offsetWidth + y,
    height: element.offsetHeight + x,
  };

  // reset element styles
  // 重置元素样式
  element.style.display = _display;
  element.style.visibility = _visibility;
  return result;
}
/**
 * Get offsets to the popper
 * 获取popper的偏移量
 * @method
 * @memberof Popper
 * @access private
 * @param {Element} popper - the popper element
 * @param {Element} reference - the reference element (the popper will be relative to this)
 * @returns {Object} An object containing the offsets which will be applied to the popper
 */
Popper.prototype._getOffsets = function (popper, reference, placement) {
  placement = placement.split("-")[0];
  var popperOffsets = { position: "absolute" };

  //
  // Get reference element position
  //
  var referenceOffsets = getOffsetRectRelativeToCustomParent(reference, getOffsetParent(popper));

  //
  // Get popper sizes
  //
  var popperRect = getOuterSizes(popper);

  //
  // Compute offsets of popper
  //

  // depending by the popper placement we have to compute its offsets slightly differently
  if (["right", "left"].indexOf(placement) !== -1) {
    popperOffsets.top = referenceOffsets.top + referenceOffsets.height / 2 - popperRect.height / 2;
    if (placement === "left") {
      popperOffsets.left = referenceOffsets.left - popperRect.width;
    } else {
      popperOffsets.left = referenceOffsets.right;
    }
  } else {
    popperOffsets.left =
      referenceOffsets.left + referenceOffsets.width / 2 - popperRect.width / 2;
    if (placement === "top") {
      popperOffsets.top = referenceOffsets.top - popperRect.height;
    } else {
      popperOffsets.top = referenceOffsets.bottom;
    }
  }

  // Add width and height to our offsets object
  popperOffsets.width = popperRect.width;
  popperOffsets.height = popperRect.height;

  return {
    popper: popperOffsets,
    reference: referenceOffsets,
  };
};

function Popper(popper, reference, options) {
  options.gpuAcceleration = options.gpuAcceleration || true;
  this.modifiers._options = options;
  this.modifiers._popper = popper;
  var data = { instance: this, styles: {} };
  data.placement = options.placement || "right-start";
  data.offsets = this._getOffsets(popper, reference, data.placement);
  data = this.modifiers.shift(data);
  data = this.modifiers.applyStyle(data);
}

```

### 滚动条场景

如果最近已定位的父元素有滚动条的话也会影响 popper 位置的计算，出现滚动条有以下几种情况

-   popper 和 reference 有同一个父元素并且已定位，父元素出现滚动条
    -   popper 开始显示在文档中，设置为绝对定位后父元素的滚动计算重新计算。
    -   popper 开始 display 为 none，设置为绝对定位后父元素的滚动不影响
-   popper 和 reference 不同级。即 popper 最近已定位元素有滚动条不是 reference 的父元素。

Popperjs 并没有处理 absolute 下最近已定位父元素有滚动条下 scrollTop 导致定位出错的情况。
写个 demo 测试一下

```html
 <div  id="scroll" style="position: relative; top: 100px; height: 500px;overflow: auto;" >
    <button  id="reference"  style="position: relative; top: 100px; left: 200px" >
    reference
    </button>
    <div id="popper" style="background-color: red; width: 300px; height: 200px;" >
      popper
    </div>
    <div  style="background-color: blue; width: 300px; height: 700px;" >
    </div>
  </div>
```

```js
document.getElementById("scroll").scrollTop=50
  document.getElementById("reference").onclick = function () {
    new Popper(document.getElementById("reference"), document.getElementById("popper"),
      { placement: "top-start", gpuAcceleration: false }
    );
  };
```
效果如下：
![1673340936733.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25296ad27c1143c1a04a6d8862c4223f~tplv-k3u1fbpfcp-watermark.image?)
点击按钮后：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0f213f71f904ce3afe94c697fda86af~tplv-k3u1fbpfcp-watermark.image?)

popper 定位忽略了 scrollTop 为 50，原因是 getBoundingClientRect 是相对可视窗口的定位，而 top 和 left 是相对最近以定位的父元素

### 避免裁剪和溢出

裁剪和溢出其实是当 popper 定位到 reference 附近位置的时候超出最近已定位父元素的尺寸导致裁剪或者溢出，
所以这里要计算 popper 定位的边界。


```js
/**
     * Computed the boundaries limits and return them
     * 计算边界限制并且返回
     * @method
     * @memberof Popper
     * @access private
     * @param {Object} data - Object containing the property "offsets" generated by `_getOffsets`
     * @param {Number} padding - Boundaries padding
     * @param {Element} boundariesElement - Element used to define the boundaries
     * @returns {Object} Coordinates of the boundaries
     */
    Popper.prototype._getBoundaries = function (data, padding, boundariesElement) {
        // NOTE: 1 DOM access here
        // 注意：一个dom读取的地方 
        var boundaries = {};
        var width, height;
        // 边界元素是window
        if (boundariesElement === 'window') {
            var body = root.document.body,
                html = root.document.documentElement;
            //为了兼容浏览器使用此方法获取文档高度和宽度
            height = Math.max(body.scrollHeight, body.offsetHeight, html.clientHeight, html.scrollHeight, html.offsetHeight);
            width = Math.max(body.scrollWidth, body.offsetWidth, html.clientWidth, html.scrollWidth, html.offsetWidth);

            boundaries = {
                top: 0,
                right: width,
                bottom: height,
                left: 0
            };
        } else if (boundariesElement === 'viewport') { //边界元素是最近定位的祖先元素视口
            var offsetParent = getOffsetParent(this._popper);
            var scrollParent = getScrollParent(this._popper);
            var offsetParentRect = getOffsetRect(offsetParent);

            // Thanks the fucking native API, `document.body.scrollTop` & `document.documentElement.scrollTop`
            // 感谢该死的原生API，`document.body.scrollTop` 和 `document.documentElement.scrollTop`
            var getScrollTopValue = function (element) {
                return element == document.body ? Math.max(document.documentElement.scrollTop, document.body.scrollTop) : element.scrollTop;
            }
            var getScrollLeftValue = function (element) {
                return element == document.body ? Math.max(document.documentElement.scrollLeft, document.body.scrollLeft) : element.scrollLeft;
            }

            // if the popper is fixed we don't have to substract scrolling from the boundaries
            // 如果popper是固定定位，我们不用从边界上减去滚动条的宽高
            var scrollTop = data.offsets.popper.position === 'fixed' ? 0 : getScrollTopValue(scrollParent);
            var scrollLeft = data.offsets.popper.position === 'fixed' ? 0 : getScrollLeftValue(scrollParent);

            boundaries = {
                top: 0 - (offsetParentRect.top - scrollTop),
                right: root.document.documentElement.clientWidth - (offsetParentRect.left - scrollLeft),
                bottom: root.document.documentElement.clientHeight - (offsetParentRect.top - scrollTop),
                left: 0 - (offsetParentRect.left - scrollLeft)
            };
        } else {
            if (getOffsetParent(this._popper) === boundariesElement) {
                boundaries = {
                    top: 0,
                    left: 0,
                    right: boundariesElement.clientWidth,
                    bottom: boundariesElement.clientHeight
                };
            } else {
                boundaries = getOffsetRect(boundariesElement);
            }
        }
        boundaries.left += padding;
        boundaries.right -= padding;
        boundaries.top = boundaries.top + padding;
        boundaries.bottom = boundaries.bottom - padding;
        return boundaries;
    };
```
代码中对 boundariesElement 进行判断，如果是 window 获取浏览器可视区域的边界；如果是 viewport 获取边界元素是最近定位的祖先元素视口；如果是一个 dom 对象获取这个对象的视口作为边界。
计算边界的代码是：

```js
     boundaries = {
                top: 0 - (offsetParentRect.top - scrollTop),
                right: root.document.documentElement.clientWidth - (offsetParentRect.left - scrollLeft),
                bottom: root.document.documentElement.clientHeight - (offsetParentRect.top - scrollTop),
                left: 0 - (offsetParentRect.left - scrollLeft)
            };
```



浏览器视口边界相对offsetParent的位置，`0 - offsetParentRect.top`再把滚动条的情况算进去就是`0 - offsetParentRect.top + scrollTop)`, 这里计算最近定位的祖先元素视口的定位用的是 getOffsetRect，而不是相对浏览器视口的距离。
```js
    /**
     * Get the position of the given element, relative to its offset parent
     * @function
     * @ignore
     * @param {Element} element
     * @return {Object} position - Coordinates of the element and its `scrollTop`
     */
    function getOffsetRect(element) {
        var elementRect = {
            width: element.offsetWidth,
            height: element.offsetHeight,
            left: element.offsetLeft,
            top: element.offsetTop
        };

        elementRect.right = elementRect.left + elementRect.width;
        elementRect.bottom = elementRect.top + elementRect.height;

        // position
        return elementRect;
    }
```
这里获取的是 offsetParent 元素的实际 left 和top 值，如果 offsetParent 还有一个父元素，offsetParent 的top 是相对父元素的定位，那这里计算边界就有误差。写个 demo 试试：

```html
      <div  id="scrollParent"  style="position: relative; top: 100px; height: 500px;overflow: auto;" >
        <div  id="scroll" style="position: relative; top: 100px; height: 500px;" >
          <button  id="reference"  style="position: relative; top: 100px; left: 200px" >
          reference
          </button>
          <div id="popper" style="background-color: red; width: 300px; height: 200px;" >
            popper
          </div>
          <div  style="background-color: blue; width: 300px; height: 700px;" >
          </div>
      </div>
    </div>
```

```js
   document.getElementById("reference").onclick = function () {
        new Popper(document.getElementById("reference"), document.getElementById("popper"),
          { placement: "top-start", gpuAcceleration: false }
        );
      };
```
demo 中算出来的边界 top 是 -100，也就是计算的边界是 scrollParent 元素而不是浏览器的可视区域。但是 bottom 的值是 `root.document.documentElement.clientHeight - (offsetParentRect.top - scrollTop)` 为1197，却又是浏览器的可视区域作为边界计算考虑了 scroll 的 top 但没有考虑scrollParent 的 top。计算边界 top 是按照 scrollParent 元素作为边界，计算 bottom 是按照浏览器的可视区域作为边界，边界元素有点混乱。

判断 popper 的位置是否超出边界

```js
    /**
     * Modifier used to make sure the popper does not overflows from it's boundaries
     * 被用于确定popper在边界内没有溢出的修饰器
     * @method
     * @memberof Popper.modifiers
     * @argument {Object} data - The data object generated by `update` method
     * @returns {Object} The data object, properly modified
     */
    Popper.prototype.modifiers.preventOverflow = function (data) {
        var order = this._options.preventOverflowOrder;
        var popper = getPopperClientRect(data.offsets.popper);

        var check = {
            left: function () {
                var left = popper.left;
                if (popper.left < data.boundaries.left) {
                    left = Math.max(popper.left, data.boundaries.left);
                }
                return { left: left };
            },
            right: function () {
                var left = popper.left;
                if (popper.right > data.boundaries.right) {
                    left = Math.min(popper.left, data.boundaries.right - popper.width);
                }
                return { left: left };
            },
            top: function () {
                var top = popper.top;
                if (popper.top < data.boundaries.top) {
                    top = Math.max(popper.top, data.boundaries.top);
                }
                return { top: top };
            },
            bottom: function () {
                var top = popper.top;
                if (popper.bottom > data.boundaries.bottom) {
                    top = Math.min(popper.top, data.boundaries.bottom - popper.height);
                }
                return { top: top };
            }
        };

        order.forEach(function (direction) {
            data.offsets.popper = Object.assign(popper, check[direction]());
        });

        return data;
    };
```
preventOverflowOrder 来配置遍历顺序，默认是 `['left', 'right', 'top', 'bottom']` 判断是否超出边界，超出的话设为边界值。这样 popper 和 reference 就会出现重叠，就要进行下一步翻转
### 翻转
popper 和 reference 就会出现重叠，popper 进行另一面翻转，比如 let 出现重叠就翻转为 right。

```js
    /**
     * Modifier used to flip the placement of the popper when the latter is starting overlapping its reference element.
     * 装饰器被用于翻转popper的位置当popper和他的reference元素重叠的时候
     * Requires the `preventOverflow` modifier before it in order to work.
     * 确保preventOverflow装饰器要在之前执行
     * **NOTE:** This modifier will run all its previous modifiers everytime it tries to flip the popper!
     * 每次装饰器翻转popper时它都会执行它之前的装饰器
     * @method
     * @memberof Popper.modifiers
     * @argument {Object} data - The data object generated by _update method
     * @returns {Object} The data object, properly modified
     */
    Popper.prototype.modifiers.flip = function (data) {
        // check if preventOverflow is in the list of modifiers before the flip modifier.
        // 检查preventOverflow是否在flip之前的列表中
        // otherwise flip would not work as expected.
        // 否则filp无法正常工作
        if (!this.isModifierRequired(this.modifiers.flip, this.modifiers.preventOverflow)) {
            console.warn('WARNING: preventOverflow modifier is required by flip modifier in order to work, be sure to include it before flip!');
            return data;
        }

        if (data.flipped && data.placement === data._originalPlacement) {
            // seems like flip is trying to loop, probably there's not enough space on any of the flippable sides
            return data;
        }

        var placement = data.placement.split('-')[0];
        var placementOpposite = getOppositePlacement(placement);
        var variation = data.placement.split('-')[1] || '';

        var flipOrder = [];
        if (this._options.flipBehavior === 'flip') {
            flipOrder = [
                placement,
                placementOpposite
            ];
        } else {
            flipOrder = this._options.flipBehavior;
        }

        flipOrder.forEach(function (step, index) {
            if (placement !== step || flipOrder.length === index + 1) {
                return;
            }

            placement = data.placement.split('-')[0];
            placementOpposite = getOppositePlacement(placement);

            var popperOffsets = getPopperClientRect(data.offsets.popper);

            // this boolean is used to distinguish right and bottom from top and left
            // they need different computations to get flipped
            var a = ['right', 'bottom'].indexOf(placement) !== -1;

            // using Math.floor because the reference offsets may contain decimals we are not going to consider here
            if (
                a && Math.floor(data.offsets.reference[placement]) > Math.floor(popperOffsets[placementOpposite]) ||
                !a && Math.floor(data.offsets.reference[placement]) < Math.floor(popperOffsets[placementOpposite])
            ) {
                // we'll use this boolean to detect any flip loop
                data.flipped = true;
                data.placement = flipOrder[index + 1];
                if (variation) {
                    data.placement += '-' + variation;
                }
                data.offsets.popper = this._getOffsets(this._popper, this._reference, data.placement).popper;

                data = this.runModifiers(data, this._options.modifiers, this._flip);
            }
        }.bind(this));
        return data;
    };
```
```js
    /**
     * Get the opposite placement of the given one/
     * @function
     * @ignore
     * @argument {String} placement
     * @returns {String} flipped placement
     */
    function getOppositePlacement(placement) {
        var hash = { left: 'right', right: 'left', bottom: 'top', top: 'bottom' };
        return placement.replace(/left|right|bottom|top/g, function (matched) {
            return hash[matched];
        });
    }
```
从 `getOppositePlacement` 代码中可以看到翻转只能在 x 轴或 y 轴翻转，不能从 x 轴翻到 y 轴。可以设置 `flipBehavior` 来配置翻转规则。翻转之后再执行 `_getOffsets` 和 `runModifiers` 来重新计算定位和判断是否超出边界。如果左右两边都超出边界，就要设置 `flipBehavior` 来自定义翻转规则。
### 监听滚动条和页面视图改变

```js
/**
     * Setup needed event listeners used to update the popper position
     * 设置所需的事件监听器用于更新弹出器位置
     * @method
     * @memberof Popper
     * @access private
     */
    Popper.prototype._setupEventListeners = function () {
        // NOTE: 1 DOM access here
        // 注意:这里有1个DOM访问
        this.state.updateBound = this.update.bind(this);
        root.addEventListener('resize', this.state.updateBound);
        // if the boundariesElement is window we don't need to listen for the scroll event
        // 如果boundariesElement是window，我们不需要监听滚动事件
        if (this._options.boundariesElement !== 'window') {
            var target = getScrollParent(this._reference);
            // here it could be both `body` or `documentElement` thanks to Firefox, we then check both
            // 这里target可能是body或者documentElement，感谢Firefox，然后我们检查这两个
            if (target === root.document.body || target === root.document.documentElement) {
                target = root;
            }
            target.addEventListener('scroll', this.state.updateBound);
            this.state.scrollTarget = target;
        }
    };
```
这里没有对 resize 和 scroll 事件做节流处理，再看看 update 的代码。

```js
/**
     * Updates the position of the popper, computing the new offsets and applying the new style
     * 更新popper的定位，计算新的偏移量和应用新的样式
     * @method
     * @memberof Popper
     */
    Popper.prototype.update = function () {
        var data = { instance: this, styles: {} };


        // store placement inside the data object, modifiers will be able to edit `placement` if needed
        // 位置数据放在data对象中，如果有需要modifiers将会修改`placement`字段
        // and refer to _originalPlacement to know the original value
        // 并参考_originalPlacement了解原始值
        data.placement = this._options.placement;
        data._originalPlacement = this._options.placement;


        // compute the popper and reference offsets and put them inside data.offsets
        // 计算popper和refernce的偏移量并放到data.offsets中
        data.offsets = this._getOffsets(this._popper, this._reference, data.placement);


        // get boundaries
        // 获取边界
        data.boundaries = this._getBoundaries(data, this._options.boundariesPadding, this._options.boundariesElement);


        data = this.runModifiers(data, this._options.modifiers);


        if (typeof this.state.updateCallback === 'function') {
            this.state.updateCallback(data);
        }
    };
```
## 总结
Popperjs 在保持 DOM 上下文上没有考虑 popper 元素 display 为 none 的情况和滚动条的情况。elementUI 通过把 popper 元素放到 boy 里面来避免上述情况。
Popperjs 处理溢出情况是先定位在边界内，然后判断 popper 元素和 reference 是否重叠而进行翻转。在获取边界元素的方位值时出现边界元素混乱的情况。
由此看出 Popperjs 0.5 版本还有些问题，还不是很成熟，所以 elementUI 把代码放到本地来方便后期维护。

## 待续
本来打算只做上、下两篇，后面发现内容有点多，就决定分为上、中、下篇。下一篇的内容是 Popperjs 其他配置项的实现和 elementUI 在 Popperjs 的基础上做了哪些修改。
