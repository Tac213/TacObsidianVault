[mdn doc](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements)

## Autonomous custom elements

Autonomous custom elements必须继承`HTMLElement`

```js
class CustomElement extends HTMLElement {
    constructor() {
        super();
    }
}

window.customElements.define('custom-element', CustomElement);
```

html创建方法：

```html
<custom-element></custom-element>
```

js创建方法：

```js
document.createElement('custom-element');
```

## Customized built-in elements

Customized built-in elements需要继承其他继承自`HTMLElement`的类，比如：

```js
export class DragDropLIElement extends HTMLLIElement {
    constructor() {
        super();

        this.index = -1;
    }

}

window.customElements.define('drag-drop-li', DragDropLIElement, { extends: 'li' })
```

注意`define`函数要多传一个参数。

html创建方法：

```html
<li is="drag-drop-li"></li>
```

js创建方法：

```js
document.createElement('li', { is: 'drag-drop-li' });
```
