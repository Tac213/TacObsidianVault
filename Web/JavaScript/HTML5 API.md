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

## AudioElement

AudioElement用于处理音频。需要像下面这样定义一个audio:

```html
<audio src="music/Gunjou.mp3" id="audio"></audio>
```

audio标签的element的类型为`HTMLAudioElement`。

可监听事件：

- timeupdate: 视频播放过程中时间更新，可用于更新进度条、显示时间戳之类的
- play: 播放事件
- pause: 暂停事件

### 音频相关操作

比较简单，直接看代码即可：

```js
    playCurrentSong() {
        this.musicContainer.classList.add('play');
        this.playButton.querySelector('i.fas').classList.remove('fa-play');
        this.playButton.querySelector('i.fas').classList.add('fa-pause');

        this.audioElement.play();
    }

    pauseCurrentSong() {
        this.musicContainer.classList.remove('play');
        this.playButton.querySelector('i.fas').classList.add('fa-play');
        this.playButton.querySelector('i.fas').classList.remove('fa-pause');

        this.audioElement.pause();
    }

    /**
     *
     * @param {PointerEvent} event
     */
    setProgress(event) {
        const clickX = event.offsetX;
        this.audioElement.currentTime = (clickX / this.progressContainer.clientWidth) * this.audioElement.duration;
    }
```

## 动态创建Element

```js
/** @type {HTMLDivElement} */
const divElement = document.createElement('div');
/** add element to parent */
parentElement.appendChild(divElement);

/** write html code here */
divElement.innerHTML = 'some html code';
```

## innerHTML

Element上都有innerHTML属性，使用innerHTML属性，可以在运行时动态改变dom，比如：

```js
postElement.innerHTML = `
    <div class="number">${this.postCount}</div>
    <div class="post-info">
        <h2 class="post-title">${post.title}</h2>
        <p class="post-body">${post.body}</p>
    </div>
`;
```

## 发起http请求

```js
async function fetchRandomUser()
{
    /**@type {Response} */
    let res = await fetch('https://randomuser.me/api');
    // for more infomation, visit: https://randomuser.me
    let userData = await res.json();
}
```

如果要给协议报文增加Header信息，可以填充第二个参数，比如：

```js
async function getMoreSongs(url) {
    const res = await fetch(`https://cors-anywhere.herokuapp.com/${url}`, {
        headers: {
            'X-Requested-With': 'XMLHttpRequest',
        },
    });
    const songDatas = res.json();
}
```

其他Headers定义方法[参考](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#headers)

## Window.localStorage

[mdn web docs](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)

是一个Storage对象，可以保存窗口的本地信息，用下面的方法可以写入数据：

```js
localStorage.setItem('transactions', JSON.stringify(this.transactions));
```

用下面的方法可以读取数据：

```js
let localStorageTransaction = localStorage.getItem('transactions');
let transactions = localStorageTransaction !== null ? JSON.parse(localStorageTransaction) : [];
```

用下面的方法可以清除数据：
```js
localStorage.clear()
```

## Element.querySelector / Element.querySelectorAll

以css选择器的语法，选择一个子Element。

querySelector返回首个满足selector语法的Element:

```js
/** @type {HTMLElement} */
let subelement = element.querySelector('div');
```

querySelectorAll返回所有满足selector语法的Element array:

```js
/** @type {Array<HTMLElement>} */
let subelements = element.querySelector('.custom-class');
```

## 监听页面滚动到底部

```js
window.addEventListener('scroll', () => {
    const { scrollTop, scrollHeight, clientHeight } = document.documentElement;

    if (scrollHeight - scrollTop === clientHeight) {
        // do something
    }
});
```

## HTMLElement.tagName

通过这个属性可以判断element的类型，比如在点击事件回调里，判断被点击的是按钮：

```js
/**
 * @param {PointerEvent} event
 */
function onClickResult(event) {
    /** @type {HTMLElement} */
    // @ts-ignore
    const eventTarget = event.target;
    if (eventTarget.tagName === 'BUTTON') {
        // do something
    }
}
```

HTMLElement.getAttribute

通过这个函数可以获得element的一些自定义属性，比如element是这样定义的：

```html
<button class="btn" data-artist="A" data-songtitle="a">Get Lyrics</button>
```

那么可以通过下面的方法拿到html标签上的属性：

```js
const artistName = element.getAttribute('data-artist');
const songTitle = element.getAttribute('data-songtitle');
```

## event.composedPath

这个函数可以拿到事件发生时，从上到下与事件相关的所有element。比如点击事件，调这个函数可以拿到鼠标点从上到下的所有element，接着就可以对这个Array做操作，比如调用Array的find拿到目标element:

```js
/**
 * @param {PointerEvent} event
 */
function onClickMeal(event) {
    /** @type {HTMLElement} */
    // @ts-ignore
    const mealInfo = event.composedPath().find((item) => {
        // @ts-ignore
        if (item.classList) return item.classList.contains('meal-info');
        return false;
    });
    if (!mealInfo) return;
}
```

## document.location.reload

可以直接reload当前document

```js
document.location.reload()
```

## element.classList / element.className

这些属性可以直接操作element的class属性

通过直接设置`className`，可以类似HTML那样直接改变class属性的值，比如：

```js
element.className = 'card active';
```

而通过classList则可以增减class:

```js
element.classList.remove('a');
element.classList.add('b');
```

还可以通过toggle来根据当前状态增加或者减少：

```js
element.classList.toggle('c');
```

## draggable element

通过html可以直接定义某个element是可以拖拽的：

```html
<div draggable="true"></div>
```

如果可以拿到element的实例，也可以通过：

```js
element.setAttribute('draggable', 'true');
```

或者：

```js
element.draggable = true;
```

来完成。

## drag drop event

和darg drop相关的事件有：

- dragstart
- dragend
- dragenter
- dragleave
- drop

事件参数类型都是DragEvent。

## 语音识别Speech recognition

[mdn doc](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Speech_API/Using_the_Web_Speech_API)

对于 Web Speech API speech recognition(语音识别)的支持，在各浏览器中还不成熟，还在发展，为了支持多个浏览器，其核心类的代码应当这样写：

```js
window.SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
window.SpeechGrammarList = window.SpeechGrammarList || window.webkitSpeechGrammarList
window.SpeechRecognitionEvent = window.SpeechRecognitionEvent || window.webkitSpeechRecognitionEvent
```

chrome上目前都是带webkit前缀的。

实例化语音识别器：

```js
let recognition = new window.SpeechRecognition();
//recognition.continuous = false;
recognition.lang = 'en-US';
recognition.interimResults = false;
recognition.maxAlternatives = 1;
```

识别语法：

```js
let colors = [ 'aqua' , 'azure' , 'beige', 'bisque', 'black', 'blue', 'brown', 'chocolate', 'coral' ... ];
let grammar = '#JSGF V1.0; grammar colors; public <color> = ' + colors.join(' | ') + ' ;';
let speechRecognitionList = new window.SpeechGrammarList();
speechRecognitionList.addFromString(grammar, 1);  // 第二个参数是权重，范围0 - 1
recognition.grammars = speechRecognitionList;
```

语法格式使用的是 [JSpeech Grammar Format](https://www.w3.org/TR/jsgf/) (JSGF)

开始语音识别：

```js
/**
 * Capture user speak
 * @param {SpeechRecognitionEvent} e
 */
function onSpeak(e) {
    /** @type {SpeechRecognitionResultList} */
    const resultList = e.results;
    /** @type {SpeechRecognitionResult} */
    const result = resultList[0];
    /** @type {SpeechRecognitionAlternative} */
    const alternative = result[0];
    const msg = alternative.transcript;  // 拿到语音对应的string
}

recognition.addEventListener('result', onSpeak);
recognition.start();
```

停止语音识别：

```js
recognition.stop();
```

recognition可使用的事件：

- result: 语音结果
- end: 语音结束

## 语音合成Speech synthesis

[mdn doc](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Speech_API/Using_the_Web_Speech_API#speech_synthesis)

语音合成即文字转语音。

可以通过下面的方式，拿到当前系统支持的声音列表：

```js
    constructor() {
        /** @type {Array<SpeechSynthesisVoice>} */
        this.voices = [];
        this.populateVoiceList();
        if (window.speechSynthesis.onvoiceschanged !== undefined) {
            window.speechSynthesis.addEventListener('voiceschanged', this.populateVoiceList.bind(this));
        }
    }

    populateVoiceList() {
        const voices = window.speechSynthesis.getVoices();
        for (const voice of voices) {
            if (this.voices.includes(voice)) {
                continue;
            }
            this.voices.push(voice);
            const optionElement = document.createElement('option');
            optionElement.value = voice.name;
            if (voice.default) {
                optionElement.defaultSelected = true;
            }
            optionElement.innerText = `${voice.name} (${voice.lang})`;
            this.voiceSelect.appendChild(optionElement);
        }
    }
```

然后创建一个`SpeechSynthesisUtterance`实例，传入必要参数，即可合成声音：

```js
    handleReadText() {
        const voiceName = this.voiceSelect.selectedOptions[0].value;
        for (const voice of this.voices) {
            if (voice.name !== voiceName) {
                continue;
            }
            const utterance = new SpeechSynthesisUtterance(this.textarea.value);
            utterance.voice = voice;
            utterance.pitch = 0.5;  // 0.0 - 1.0
            utterance.rate = 2.0;
            window.speechSynthesis.speak(utterance);
            return;
        }
    }
```
