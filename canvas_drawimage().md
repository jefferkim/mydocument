#话题:关于canvas drawimage()函数的讨论HTML5中引入新的元素canvas，其drawImage 方法允许在 canvas 中插入其他图像( img 和 canvas 元素) 。
- drawImage函数有三种函数原型：- drawImage(image, dx, dy) - drawImage(image, dx, dy, dw, dh) - drawImage(image, sx, sy, sw, sh, dx, dy, dw, dh)第一个参数image可以用HTMLImageElement，HTMLCanvasElement或者HTMLVideoElement作为参数。dx和dy是image在canvas中定位的坐标值；dw和dh是image在canvas中即将绘制区域（相对dx和dy坐标的偏移量）的宽度和高度值；sx和sy是image所要绘制的起始位置，sw和sh是image所要绘制区域（相对image的sx和sy坐标的偏移量）的宽度和高度值。 ![image](http://img04.taobaocdn.com/tps/i4/T19UhZXAhXXXaH.prs-360-360.gif)##@王铁套
	<html> 	  <head>         <title>drawImage with an incorrect type for the image argument (part two)</title>  	   <style> 	     canvas { display:none } 	   </style> 	   <script>   	   window.onload = function(){   	   var r = document.getElementById('r');    	   ctx = document.getElementsByTagName('canvas')[0].getContext('2d');    	   var passed = false;   	   var message = "";              try{                   ctx.drawImage((new Image()), 0, 0, 150, 150);                  message = "No exception thrown"              }catch(e){                    passed = e.code === e.INDEX_SIZE_ERR;                    if (!passed) {                   message = "Got exception code " + e.code + " expected 1 (INDEX_SIZE_ERR)"}                  }                  r.textContent = passed ? "PASS" : "FAIL";                  if (message) {                   r.textContent += " (" + message + ")"}                 }            </script>        </head>        <body>           <p id="r">FAIL (Script did not run.)</p>            <canvas></canvas>        </body>	</html>
- 在Chrome(24.0.1312.57) 和 Firefox(18.0.1) 输出PASS- 但在Opera(12.12) 和 IE(10.0.9200.16439) 输出FAIL (不抛出异常)##@Kang-Hao (Kenny) Lu:
现有规范： 

- 如果drawImage函数中image参数是一个没有完全加载完成的HTMLImageElement对象，或者是一个readyState属性是HAVE_NOTHING或者HAVE_METADATA的HTMLVideoElementobject对象，那么中止。

一个没有src的<img>标签按理说应该没有完全加载，那也就不应该抛出异常。
但是，webkit可能不会迎合规范进行修改，毕竟这种表现已经延续了很久了，火狐是有bug提交（https://bugzilla.mozilla.org/show_bug.cgi?id=691186），chrome中未发现。


建议你在本地或者在一个稳定环境下，在canvas上绘画一个video，看下是否正常工作。
有些用户在试图绘制一个没有加载完全的video到drawImage后抛出异常并停止工作




##@Glenn Maynard

chrome 24 和 firefox 19 (https://zewt.org/~glenn/test-drawimage-exception.html) 测试结果如下：
####chrome中：
- drawImage(img, dx, dy) 图片下载完前不做任何处理
- drawImage(img, dx, dy, dw, dh) 图片下载完前不做任何处理
- drawImage(img, dx, dy) 在图片下载失败后不做任何处理
- drawImage(img, dx, dy, dw, dh) 在图片下载失败后抛出异常

####firefox中：
- drawImage(img, dx, dy) 图片下载完前不做任何处理
- drawImage(img, dx, dy, dw, dh) 图片下载完前不做任何处理
- drawImage(img, dx, dy) 图片下载失败抛出异常
- drawImage(img, dx, dy, dw, dh) 图片下载失败抛出异常

IE9 和 Firefox表现一致.


理论上在图片完成加载前不会抛出异常，但是会在获取完成但失败时会抛出异常。

认为chrome中有一项存在bug，三个参数和五个参数表现不同是不合逻辑的。

-----------------

>   - Moreover, Opera has lazy loading of images (only loading images
>   - that are rendered or have some event handlers or were created with
>   - new Image() etc), so we'd probably want to just load the image when
>   - the script tries  to draw it instead of throwing.

opera在draw时来加载已经太晚了，因为drawImage需要同步完成。当然如果你碰巧图片同步（比如在内存缓存中或者其他）



##@Rik Cabanier
to @Glenn Maynard

- 这种行为还是符合如下标准的（如whatwg的规范：http://www.whatwg.org/specs/web-apps/current-work/multipage/the-canvas-element.html#check-the-usability-of-the-image-argument）除了第2条：

> If the image argument is an *HTMLCanvasElement* object with either a horizontal dimension or a  vertical dimension equal to zero, then the implementation throw an InvalidStateError exception and return aborted.

 我认为中间的*HTMLCanvasElement*对象应该是*CanvasImageSource*对象 

非规范的drawImage (http://www.whatwg.org/specs/web-apps/current-work/multipage/the-canvas-element.html#dom-context-2d-drawimage)
如果image没有图片数据，抛出一个InvalidStateError 异常，如果图片没有完全加载完，将不会绘制任何东西

@Glenn Maynard 这是一个webkit的bug，能否给出一个测试用例

##@Robert O'Callahan
> I actually just wrote a patch to implement the spec behavior in Firefox.
> I think changing behavior from "throw" to "not throw" shouldn't have any
> compatibility concerns. I also think that "not throw" is better here than
> throwing; it's simpler to not distinguish "finished downloading but
> decoding failed" from "download in progress (but very slow perhaps)".

> In fact I question why the spec has us throw for zero-sized canvas source.
> It would seem to me to be simpler/better to just not draw and not throw in
> that case also.

刚写了个patch来确保在Firefox种按标准实现，我个人认为从抛异常到不抛异常将不会产生兼容性问题，而且不抛异常更好。
区分“下载完成但是解析失败“和”下载中（可能是很慢）“
我比较质疑为什么规范会对0尺寸的canvas源抛异常，我还是更倾向在这种情况下最好索性不要绘制而且不要抛出异常。


##@Glenn Maynard
回复@Rik Cabanier

推荐webkit来迎合firefox，因为我们已经在Firefox和IE上实现了，并且在这三个所有的加载中实现。  改变webkit的话就会统一，而改变火狐的话就会导致ie没有统一（没有测试IE10，只有在IE9上）

##@Rik Cabanier
回复@Glenn Maynard 

一张图片在下载中可能没有尺寸而引发表现改变，或许第一步应该如下：
一个没有完全加载完成的HTMLImageElement对象并且不是破损状态（http://www.whatwg.org/specs/web-apps/current-work/multipage/embedded-content-1.html#img-error
）

##@Glenn Maynard
回复@Rik Cabanier


1.图片参数是一个状态为破损的HTMLImageElement对象，那么抛出InvalidStateError的异常，然后中止
2.图片参数是没有完全加载完的HTMLImageElement对象，或者是一个HTMLVideoElement对象，其readyState要么是HAVE_NOTHING或者HAVE_METADATA，那么中止
3.原来第二条，不变

##@Rik Cabanie
回复@Glenn Maynard
原始的第二条，应该是*CanvasImageSource*而不是HTMLImageElement，这就是webkit的实现方案

##@Glenn Maynard
回复@Rik Cabanier 
That sounds fine too, as long as you mean in addition to the above and not
instead of.

(If you mean instead of, I disagree.  As far as throwing or not throwing
based on the broken or loading state question, it seems like WebKit should
change to match FF and IE, since those two already agree, and WebKit also
agrees with one of the two drawImage overloads tested.)

应该是附加而不是替换

因为抛不抛异常取决于是否破损或者是否是在加载中，看上去webkit应该修改下来统一FF和IE，因为FF和IE应该赞同，并且webkit在测试中drawImage两个中有一个是抛异常的

##@ Rik Cabanier
回复@Glenn Maynard 

是附加，因此:
- 附加你的意见到第一步
- 原始的第二条，HTMLImageElement修改成CanvasImageSource

##@Glenn Maynar
回复@Rik Cabanier
"image" is always a CanvasImageSource, so if this is what was intended it
would just omit the type check entirely.

But why does it throw this exception in the first place?  It's a weird
special case.  Blitting a zero-size image should do nothing, just like
drawImage(src, 0, 0, 0, 0).


图片一直以来都应该是CanvasImageSource，如果只是有意的，那应该忽略了类型检测

但是为什么没有在一开始就抛出异常，这是一个非常怪异的例子。块运算一张0字节的图片应该不做任何处理，就像drawImage(src, 0, 0, 0, 0)

##@Kang-Hao (Kenny) Lu

(13/03/02 5:59), Glenn Maynard wrote:
> I recommend leaving Firefox alone and changing WebKit (and the spec) to
> match Firefox, because we already have interop (at least in the cases I
> tested) between Firefox and IE, and we already have interop during loads in
> all three.  Changing WebKit to throw after loading will get everyone doing
> the same thing--changing Firefox will still leave IE out.
>
> (I haven't tested with IE10, FWIW, only IE9.)

IE10 是一样表现的

(13/03/02 6:45), Rik Cabanier wrote:
> Sorry about being unclear. Yes, I meant in addition of.
> So:
> - add your suggested step 1
> - change HTMLImageElement from original step 2 to CanvasImageSource

> Did you mean to say "change *HTMLCanvasElement* from original step 2 to
> CanvasImageSource" here? The original step 2 has

将原先第二条的*HTMLCanvasElement*改成CanvasImageSource？？原先的第二步是？？

>  # If the image argument is an HTMLCanvasElement object with either a
   # horizontal dimension or a vertical dimension equal to zero, then
   # the implementation throw an InvalidStateError exception and return
   # aborted.
（@Rik Cabanier 回复 是的）
(13/03/02 9:41), Glenn Maynard wrote:
> But why does it throw this exception in the first place?  It's a weird
> special case.  Blitting a zero-size image should do nothing, just like
> drawImage(src, 0, 0, 0, 0).

Just curious. Does it have to be SVG for an image to be zero-sized?
出于好奇，是不是需要将0字节的图片SVG化
（@Rik Cabanier 回复  我认为一个canvas元素可以是0*0的）
##@Rik Cabanier
回复：@ Glenn Maynard
在 https://www.w3.org/Bugs/Public/show_bug.cgi?id=21173 提交了bug

##@Robert O'Callahan
回复@Rik Cabanier


我的patch遵循当前的规范，不会修正 https://www.w3.org/Bugs/Public/show_bug.cgi?id=21173

对于我们来说，在我们给了一张无效的或者没有像素图片，那什么都不绘制确实会很简单。什么都不抛异常将很少有可能出现兼容性风险。

##@Rik Cabanier
回复 @robert@ocallahan.org

kenny找到一些例子当改变webkit中的这种行为将导致应用奔溃
- https://github.com/Animatron/player/pull/70
- https://github.com/aduros/flambe/issues/55
- http://groups.google.com/group/melonjs/msg/571b36150fd2760b
也就意味着将不会在webkit中增加这个canvas的实现

针对这点，我想我们可以去推动W3C来改变canvas规范（和实现），这样在drawImage时0尺寸图片将不会抛出任何异常

##@Robert O'Callahan
回复@Rik Cabanier

我们将统一这个异常当图片没有进入ready state（http://www.whatwg.org/specs/web-apps/current-work/multipage/the-canvas-element.html#check-the-usability-of-the-image-argument）,这样我们应该修改（虽然我也同意不应该在第一步就抛出异常）

exception-free 的drawimage 回调，可以使用 ImageBitmap 类








