---
theme: channing-cyan
highlight: a11y-light
---

## 背景
ElementUI 中`Tooltip`、`Select`、`Cascader`、`TimePicker`等组件中怎么把提示框定位到目标元素的，是用 Popperjs 来实现，本系列文章将从以下几个角度带你了解 Popperjs：

1. **Popperjs 的特点、实现原理、使用场景。**
2. **Popperjs 0.5.2版本源码。**
4. **elementUI 对 Popperjs 做了哪些修改？**
5. **Popperjs 1.x 版本有哪些改动？**
6. **floating-ui ( Popperjs 2.x ) 版本有哪些改动？**
7. **产品功能角度 Popperjs 是如何迭代的？**
8. **ant desgin 中的 Tooltip 是怎么实现的？和 Popperjs 有什么区别？**

本篇是系列的第一篇，很多同学可能和我一样是在 elementUI 源码中第一次知道 Popperjs 的，同时产生疑问 ElementUI 采用的是 Popperjs 的哪个版本？ GitHub 上 elementUI 源码中的 popper.js 文件是在2016 年 7 月第一次提交的，而 Popperjs 在那个时间点的版本是 0.5.2。于是把两个代码进行对比，99% 是一样的，因此判定 elementUI 用的是 Popperjs 0.5.2 版本，那么我们就从 0.5.2 版本开始来了解 Popperjs 吧。

## 版本
**0.5.2**

## 介绍
### 功能
将元素定位到指定目标元素附近，例如定位提示和弹窗位置：指定一个按钮或一个提示元素描述按钮，popper 会自动把提示放在按钮附件正确的地方，应用的组件：Tooltip 文字提示、Popover 弹出框、Popconfirm 气泡确认、Dropdown 下拉菜单、Select 选择器等。
### 特点
-  **滚动容器：在多个滚动容器中提示元素能够正确的定位到目标元素附近**
-  **dom 上下文：保持提示在原有的上下文关系中**
-  **兼容性：不同浏览器、不同环境下获取边界差异**
-  **可配置：使用不同场景的一些差异配置需求**
-  **裁剪和溢出：在超出边界时会导致溢出或裁剪时，自动处理定位到其他不会超出边界的位置上**
-  **快速翻转：视图变化时，快速调整到正确的位置适应视图的变化**
### 使用方法

```html
<div class="my-button">reference</div>
<div class="my-popper">popper</div>
```

```js
var reference = document.querySelector(".my-button");
var popper = document.querySelector(".my-popper");
var popperInstance = new Popper(reference, popper, {
  // popper options here
});
```
## 实现原理
一个简单前端工具库可以分为：主要功能实现、扩展使用环境、扩展使用场景，三个方面考虑。下面将通过这三个方面来解析源码。
### 主要功能实现
Popperjs 的主要功能就是把一个 DOM 元素定位到目标 DOM 元素附近，那么我们先简单实现定位到目标 DOM 的右边。

**实现思路是：通过`getBoundingClientRect`方法来获取目标元素的位置，获取到目标元素 reference 的位置之后将 popper 设置固定定位并且`left`为 reference 的`right`值，`top`值为 reference 的`top`值，就可以把 popper 元素定位到 reference 的右边。**

`getBoundingClientRect`返回的`right`是元素右边到视窗左边的距离，`bottom`是元素下边到视窗上边的距离。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aba62612eb6a46d78525b09f3186c6b0~tplv-k3u1fbpfcp-watermark.image?)
`getBoundingClientRect`有个兼容性问题，ie9以上支持`getBoundingClientRect`方法返回`width/height`属性。要支持 ie9 以下可以这么写。

```js
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
```
计算完之后就是把样式作用到元素上了，因为这里只操作 width、height、left、top、right、bottom 所以只给这些属性加上单位。

```js
// 应用样式
function setStyle(element, styles) {
  for (let prop in styles) {
    var style = styles[prop];
    if (["width", "height", "top", "right", "bottom", "left"].indexOf(prop) !== -1
      && typeof styles[prop] === "number") {
      style += "px";
    }
    element.style[prop] = style;
  }
}
```
在这基础上还判断了数字是否是无穷大。

```js
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
            return (n !== '' && !isNaN(parseFloat(n)) && isFinite(n));
        }
        Object.keys(styles).forEach(function (prop) {
            var unit = '';
            // add unit if the value is numeric and is one of the following
            if (['width', 'height', 'top', 'right', 'bottom', 'left'].indexOf(prop) !== -1 && is_numeric(styles[prop])) {
                unit = 'px';
            }
            element.style[prop] = styles[prop] + unit;
        });
    }
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
    if (["width", "height", "top", "right", "bottom", "left"].indexOf(prop) !==-1 &&is_numeric(styles[prop])) {
      unit = "px";
    }
    element.style[prop] = styles[prop] + unit;
  });
}

function Popper(reference, popper) {
  var referenceRect = getBoundingClientRect(reference);
  var popperStyles = {
    position: "absolute",
    left: referenceRect.right,
    top: referenceRect.top,
  };
  setStyle(popper, popperStyles);
}
```
### 扩展使用场景
主要功能实现之后，就是扩展使用场景，工具库要让更多人使用就要满足业务开发中的不同的场景需求。

#### 不同方位需要
通过配置属性 placement 来配置定位方位，方位分为：`top-start`（上左）、`top-end`（上右）、`top`（上中）、`bottom-start`（下左）、`bottom-end`（下右）、`bottom`（下中）、`right-start`（右上）、`right-end`（右下）、`right`（右中）、`left-start`（左上）、`left-end`（左下）、`left`（左中）。不同方位 top 和 left 计算方式不同：

- `left-start`（左上）：定位 popper 的 left 值为`reference.left`，popper 的 top 值为`reference.top - popper.height`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ec433817075457b8bee3e6b3ad51ee7~tplv-k3u1fbpfcp-zoom-1.image)

图中 popper 为定位元素，reference 为目标元素。

- `left`（左中）：定位 popper 的 left 值为`reference.left+reference.width/2-popper.width/2`，popper 的 top 值为`reference.top-popper.height`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a2160a89c6f4e2ab1883143251d4b0e~tplv-k3u1fbpfcp-zoom-1.image)

- `left-end`（左下）：定位 popper 的 left 值为`reference.left+reference.width-popper.width`，popper 的 top 值为`reference.top-popper.height`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/217c827d685e4e19a9f4e4cc95067978~tplv-k3u1fbpfcp-zoom-1.image)

简单实现计算的思路：

```js
// 方位配置
function Popper(reference, popper, options) {
  var referenceRect = getBoundingClientRect(reference);
  var popperRect = getBoundingClientRect(popper);
  if (popperRect.width === 0 || popperRect.height === 0) {
    setStyle(popper, { position: "absolute", visible: "hidden", display: "block" });
    popperRect = getBoundingClientRect(popper);
  }

  var popperOffsets = {
    left: 0,
    top: 0,
  };
  var placement = options.placement || "right-start";
  var placementArray = placement.split("-");
  var basePlacement = placementArray[0];
  var shiftPlacement = placementArray[1];
  var baseOffsets = {
    top: {
      top: referenceRect.top - popperRect.height,
      left: referenceRect.left + referenceRect.width / 2 - popperRect.width / 2,
    },
    bottom: {
      top: referenceRect.bottom,
      left: referenceRect.left + referenceRect.width / 2 - popperRect.width / 2,
    },
    left: {
      top: referenceRect.top + referenceRect.height / 2 - popperRect.height / 2,
      left: referenceRect.left - popperRect.width,
    },
    right: {
      top: referenceRect.top + referenceRect.height / 2 - popperRect.height / 2,
      left: referenceRect.right,
    },
  };
  var shiftOffsets = {
    x: {
      start: {
        left: referenceRect.left,
      },
      end: {
        left: referenceRect.right - popperRect.width,
      },
    },
    y: {
      start: { top: referenceRect.top },
      end: { top: referenceRect.bottom - popperRect.height },
    },
  };
  var axis = ["bottom", "top"].indexOf(basePlacement) !== -1 ? "x" : "y";

  popperOffsets = Object.assign(
    popperOffsets,
    baseOffsets[basePlacement],
    shiftPlacement ? shiftOffsets[axis][shiftPlacement] : {}
  );

  setStyle(popper, popperOffsets);
```
`baseOffsets`声明了上、下、左、右四个方位默认位置，然后再判断`start`、`end`进行定位，最后再将几个定位对象组合在一起。这里先把 popper 元素放到页面中计算宽高，popper 元素可能初始样式`display:none`而获取不到宽高。在获取 popper 元素的大小时要考虑`margin`的值也要计算到里面，把 popper 元素尺寸的计算封装为`getOuterSizes`:

```js
/**
 * Get the outer sizes of the given element (offset size + margins)
 * 获取给定元素的外部尺寸（尺寸+外边距）
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
```
> `Window.getComputedStyle()`方法返回一个对象，该对象在应用活动样式表并解析这些值可能包含的任何基本计算后报告元素的所有 CSS 属性的值。


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
  
    var rect = {
      top: elementRect.top,
      left: elementRect.left,
      bottom: elementRect.top + elementRect.height,
      right: elementRect.left + elementRect.width,
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
   * Given the popper offsets, generate an output similar to getBoundingClientRect
   * 给定popper偏移量，生成类似于getBoundingClientRect的输出
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
  Popper.prototype.modifiers = {};
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
  
      data.offsets.popper = Object.assign(popper,  shiftOffsets[axis][shiftVariation]);
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
    var popperOffsets = {};
  
    //
    // Get reference element position
    //
    var referenceOffsets = getOffsetRectRelativeToCustomParent(reference);
  
    //
    // Get popper sizes
    //
    var popperRect = getOuterSizes(popper);
  
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
  
  function Popper(reference, popper, options) {
    var data = { instance: this, styles: {} };
    data.placement = options.placement || "right-start";
    data.offsets = this._getOffsets(popper, reference, data.placement);
    data = this.modifiers.shift(data);
    var styles = {};
    styles.position = "absolute";
    styles.left = data.offsets.popper.left;
    styles.top = data.offsets.popper.top;
    setStyle(popper, styles);
  }
```
这里主要`_getOffsets`、`shift`、`setStyle`函数
- `_getOffsets`函数是先获取获取 reference 元素的定位和 popper 元素的尺寸，然后再计算 placement 值为`left`（左中）、`right`（右中）、`top`（上中）、`bottom`（下中）的位置，返回一个包含 popper 和 reference 位置、尺寸的对象。

```js
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
  var popperOffsets = {};

  //
  // Get reference element position
  // 获取reference元素的定位
  var referenceOffsets = getOffsetRectRelativeToCustomParent(reference);

  //
  // Get popper sizes
  // 获取 
  var popperRect = getOuterSizes(popper);

  //
  // Compute offsets of popper
  // 根据placement的第一个定位来计算popper的偏移量

  // depending by the popper placement we have to compute its offsets slightly differently
  // 根据popper配置的方位我们计算偏移量的方式有所不同
  // 这里是先计算placement为['left','right','top','bottom']的left和top值
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
  // 在offsets对象中添加宽度和高度
  popperOffsets.width = popperRect.width;
  popperOffsets.height = popperRect.height;

  return {
    popper: popperOffsets,
    reference: referenceOffsets,
  };
};
```
- `shift`函数根据 placement 中 '-' 后面的值'start'或者'end'来计算定位，也就是计算`top-start`（上左）、`top-end`（上右）、`bottom-start`（下左）、`bottom-end`（下右）、`right-start`（右上）、`right-end`（右下）、`left-start`（左上）、`left-end`（左下）的位置。

```js
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
  // 如果指定了placement第二个定位，执行这个修饰符
  // 根据placement中'-'后面的值'start'或者'end'来计算定位
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
```
- `setStyle`将 data 中 popper 定位作用到 popper 元素上。

*细节：* 
- `getOffsetRectRelativeToCustomParent`函数是为了之后处理上下文和滚动条而封装的函数
- `getPopperClientRect`函数新建了一个 popper 元素的定位尺寸对象，防止对之前 popperOffsets 对象的操作而影响最后的定位。函数中设置 right 和 bottom，在`shift`函数计算定位中并没有更新这两个，而只是更新了 left 和 top 的值。最后定位只用到了 left 和 top 的值，如果用 right 或 bottom 就会出现问题。
#### 启用CSS3 硬件加速来定位
定位除了通过 left、top 还可以用 transform 提升合成层来启动 css3 硬件加速,避免页面触发重排和重绘从而提升页面性能，transform 中 translate3d 才能提升合成层，但如果合成层太多会占用大量内存，可以用 gpuAcceleration 属性来配置是否开启。

浏览器在支持 transform 上还有个问题，transform: translate() 或者 transform: scale() 后的计算值产生了非整数，元素里面的字体会模糊。因此要对计算值四舍五入取整处理。

```js
   popperStyles.left = Math.round(popperStyles.left);
   popperStyles.top = Math.round(popperStyles.top);
   popperStyles["transform"] = `translate3d(${popperStyles.left}px,${popperStyles.top}px,0)`;
   popperStyles = Object.assign(popperStyles, { left: 0, top: 0 });
```

transform 会有兼容性问题，早期不同浏览器在实现 css3 属性的时候会添加前缀，需要添加获取当前浏览器支持的前缀属性。


```js
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
```
整理代码封装在`applyStyle`函数中。

```js
Popper.prototype.modifiers.applyStyle = function (data) {
  // apply the final offsets to the popper
  // 将最后的偏移量应用于popper元素
  // NOTE: 1 DOM access here
  // 注意:这里有1个DOM访问
  var styles = {
    position: data.offsets.popper.position,
  };

  // round top and left to avoid blurry text
  // 四舍五入取整top和left以避免模糊文本
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
```
完整代码：

```js
// popper中的实现

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

  var rect = {
    top: elementRect.top,
    left: elementRect.left,
    bottom: elementRect.top + elementRect.height,
    right: elementRect.left + elementRect.width,
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
  var referenceOffsets = getOffsetRectRelativeToCustomParent(reference);

  //
  // Get popper sizes
  //
  var popperRect = getOuterSizes(popper);

  //
  // Compute offsets of popper
  //

  // depending by the popper placement we have to compute its offsets slightly differently
  if (["right", "left"].indexOf(placement) !== -1) {
    popperOffsets.top =
      referenceOffsets.top +
      referenceOffsets.height / 2 -
      popperRect.height / 2;
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

function Popper(reference, popper, options) {
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

## 总结

本篇主要介绍了 Popperjs 是将元素定位到指定目标元素附近，相比用 css 实现它具有滚动容器、保持 dom 上下文、兼容性、可配置、避免裁剪和溢出、快速翻转等优势。

通过主要功能实现、扩展使用场景、扩展使用环境来说明其实现原理。
## 待续
本篇只讲述扩展使用场景的不同方位、css3 硬件加速。后面还有保留 DOM 上下文、滚动条、裁剪和溢出、快速翻转、自定义扩展等场景。

待续**你不知道的Popperjs下篇**
