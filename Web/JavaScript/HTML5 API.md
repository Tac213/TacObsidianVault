## document.getDocumentById

这个是一个最基础的API，用于获取DOM上指定id的元素，得到HTMLElement。

```js
let button = document.getDocumentById('my-button');
```

## document.addEventListener

该API可以监听事件，比如监听键盘的按下和松开事件：

```js
/**
 * will be called when key up
 * @param {KeyboardEvent} event
 * @returns {void}
 */
function onKeyUp(event) {}
function onKewDown(event) {}

document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);
```

## requestAnimationFrame

可以传入一个回调，让这个回调在下一帧调用，反复调用这个函数，即可让某个函数每帧都被调用，比如：

```js
function tick()
{
	console.log('I\'m called every frame');
	requestAnimationFrame(tick);
}
```

类似上面这样，tick就会每一帧都会被调用。

## CanvasElement

CanavsElement在TypeScript中的定义如下：

```ts
/** Provides properties and methods for manipulating the layout and presentation of <canvas> elements. The HTMLCanvasElement interface also inherits the properties and methods of the HTMLElement interface. */

interface HTMLCanvasElement extends HTMLElement {

    /** Gets or sets the height of a canvas element on a document. */

    height: number;

    /** Gets or sets the width of a canvas element on a document. */

    width: number;

    captureStream(frameRequestRate?: number): MediaStream;

    /**

     * Returns an object that provides methods and properties for drawing and manipulating images and graphics on a canvas element in a document. A context object includes information about colors, line widths, fonts, and other graphic parameters that can be drawn on a canvas.

     * @param contextId The identifier (ID) of the type of canvas to create. Internet Explorer 9 and Internet Explorer 10 support only a 2-D context using canvas.getContext("2d"); IE11 Preview also supports 3-D or WebGL context using canvas.getContext("experimental-webgl");

     */

    getContext(contextId: "2d", options?: CanvasRenderingContext2DSettings): CanvasRenderingContext2D | null;

    getContext(contextId: "bitmaprenderer", options?: ImageBitmapRenderingContextSettings): ImageBitmapRenderingContext | null;

    getContext(contextId: "webgl", options?: WebGLContextAttributes): WebGLRenderingContext | null;

    getContext(contextId: "webgl2", options?: WebGLContextAttributes): WebGL2RenderingContext | null;

    getContext(contextId: string, options?: any): RenderingContext | null;

    toBlob(callback: BlobCallback, type?: string, quality?: any): void;

    /**

     * Returns the content of the current canvas as an image that you can use as a source for another canvas or an HTML element.

     * @param type The standard MIME type for the image format to return. If you do not specify this parameter, the default value is a PNG format image.

     */

    toDataURL(type?: string, quality?: any): string;

    addEventListener<K extends keyof HTMLElementEventMap>(type: K, listener: (this: HTMLCanvasElement, ev: HTMLElementEventMap[K]) => any, options?: boolean | AddEventListenerOptions): void;

    addEventListener(type: string, listener: EventListenerOrEventListenerObject, options?: boolean | AddEventListenerOptions): void;

    removeEventListener<K extends keyof HTMLElementEventMap>(type: K, listener: (this: HTMLCanvasElement, ev: HTMLElementEventMap[K]) => any, options?: boolean | EventListenerOptions): void;

    removeEventListener(type: string, listener: EventListenerOrEventListenerObject, options?: boolean | EventListenerOptions): void;

}
```

我们可以通过下面的方法在DOM中定义一个Canvas:

```html
<canvas id="canvas" width="800px" height="600px"></canvas>
```

通过下面的方法给Canvas可视化的背景:

```css
#canvas {
	background: #f0f0f0;
	display: block;  /* use block layout */
    border-radius: 5px;  /* make the block a rounded rectangle */
}
```

然后通过下面的方法拿到一个2D的context:

```js
let canvas = document.getElementById('canvas');
let canvasRenderingContext2D = canvas.getContext('2d');
```

接着就可以使用context来画东西了:

```js
// draw text
renderContext.font = '20px Arial';
renderContext.fillText(`Score: ${this.score}`, this.x, this.y);

// draw rectangle
renderContext.beginPath();
renderContext.rect(this.x, this.y, this.width, this.height);
renderContext.fillStyle = this.visible ? 'lightblue' : 'transparent';
renderContext.fill();
renderContext.closePath();

// draw circle
renderContext.beginPath();
renderContext.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
renderContext.fillStyle = this.visible ? 'lightblue' : 'transparent';
renderContext.fill();
renderContext.closePath();
```

## VideoElement

VideoElement用于处理视频。需要像下面这样定义一个video:

```html
<video src="video/gone.mp4" id="video" poster="image/poster.png" class="screen"></video>
```

poster属性为视频初始化未播放时显示的图片。

video标签的element的类型为`HTMLVideoElement`。

### 可监听事件

- click: 视频点击
- timeupdate: 视频播放过程中时间更新，可用于更新进度条、显示时间戳之类的
- play: 播放事件
- pause: 暂停事件

```js
this.video.addEventListener('click', this.toggleVideoStatus.bind(this));
this.video.addEventListener('timeupdate', this.updateProgress.bind(this));
this.video.addEventListener('play', this.togglePlayIcon.bind(this));
this.video.addEventListener('pause', this.togglePlayIcon.bind(this));
```

### 视频相关操作

比较简单，直接看代码就行了：

```js
    /**
     * toggle video status
     * @param {PointerEvent} event
     * @returns {void}
     */
    toggleVideoStatus(event)
    {
        this.isStopped = false;
        if (this.video.paused)
        {
            this.video.play();
        }
        else
        {
            this.video.pause();
        }
    }

    /**
     * stop video
     * @param {PointerEvent} event 
     * @returns {void}
     */
    stopVideo(event)
    {
        this.video.currentTime = 0;
        this.video.pause();
        this.isStopped = true;
    }


    /**
     * setVideoProgress
     * @param {Event} event 
     */
    setVideoProgress(event)
    {
        if (this.isStopped)
        {
            this.progress.value = 0;
            return;
        }
        this.video.currentTime = this.progress.value * this.video.duration / 100;
    }
```
