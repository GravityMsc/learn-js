自从React/Vue等框架流行之后，jQuery被打上了面条式代码的标签，甚至成了“过街老鼠”，好像谁还在用jQuery，谁就还活在旧时代，很多人都争先恐后地拥抱新框架，各大博客网站有很大一部分的博客都在介绍新的框架，争当时代的“弄潮儿”。新框架带来的新的理念，新的开发方式不可否认带来了生产效率，但是jQuery等就应该被打上“旧时代”面条式代码的标签么？

我们从一篇文章说起：《[React.js介绍 - 针对了解jQuery的工程师（译）](https://segmentfault.com/a/1190000003501752)》，英文原文是这个《[React.js Introduction For People Who Know Just Enough jQuery To Get By](http://chibicode.com/react-js-introduction-for-people-who-know-just-enough-jquery-to-get-by/)》，这篇文章我好久前就看过，现在再把它翻出来，里面对比了下jQuery和React分别实现一个发推的功能，作者用jQuery写着写着代码就乱套了，而用React不管需求多复杂，代码条理依旧很清晰。

我们一步步按照原文作者的思路来拆解。

## （1）输入个数为0时，发送按钮不可点击

如下图所示，当输入框没有内容时，发推按钮置灰不可点击，有内容才能点击。

![tweet的设计](https://user-gold-cdn.xitu.io/2017/9/3/7afd5c868c72e77da7476aec4765557f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

作者写的代码是这样的：

```javascript
// 初始化状态
$("button").prop("disabled", true);

// 文本框的值发生变化时
$("textarea").on("input", function() {
  // 只要超过一个字符，就
  if ($(this).val().length > 0) {
    // 按钮可以点击
    $("button").prop("disabled", false);
  } else {
    //否则，按钮不能点击
    $("button").prop("disabled", true);
  }
});
```

这个代码本身写的很累赘，首先，既然一开始那个button是disabled的，那就直接在html上写个disabled属性就行了：

```html
<form class="tweet-box">
    <textarea name="textMsg"></textarea>
    <input disabled type="submit" name="tweet" value="Tweet">
</form>
```

第二个要控制按钮的状态，其实核心只要一行代码就行了，不需要写那么长：

```javascript
let form = $(".tweet-box")[0];
$(form.textMsg).on("input", function() {
    form.tweet.disabled = this.value.length <= 0;
}).trigger("input");
```

这个代码应该够简洁了吧，而且代码在jQuery和原生之间来回切换，游刃有余。

## （2）实现剩余字数功能

如下图所示：

![tweet剩余字数](https://user-gold-cdn.xitu.io/2017/9/3/4d3762afb3322ba02b83f89c679e2518?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个也好实现：

```javascript
let form = $(".tweet-box")[0],
    $leftWordCount = $("#left-word-count");

$(form.textMsg).on("input", function() {
    // 已有字数
    let wordsCount = this.value.length;
    $leftWordCount.text(140 - wordsCount);
    form.tweet.disabled = wordsCount <= 0; 
});
```

## （3）添加图片按钮

如下图所示，左下角多了一个选择照片的按钮：

![tweet增加选择照片的按钮](https://user-gold-cdn.xitu.io/2017/9/3/9333e9e76e653cfdc57d2cf3813cae4c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果用户选择了照片，那么可输入字数将会减少23个字符，并且Add Photo文案要变成Photo Added。我们先来看下作者是怎么实现的，如下代码：

```javascript
if ($(this).hasClass("is-on")) {
  $(this)
    .removeClass("is-on")
    .text("Add Photo");
  $("span").text(140 - $("textarea").val().length);
} else {
  $(this)
    .addClass("is-on")
    .text("✓ Photo Added");
  $("span").text(140 - 23 - $("textarea").val().length);
}
```

如果代码像作者这样写的haunted确实是比较混乱，而且比较面条式。但是我们可以优雅地实现。首先，选择照片一般会写一个input[type=file]的隐藏输入框盖在上传图标下面：

```html
<div class="upload-container">
    <img src="upload-icon.png" alt>
    <span id="add-photo">Add Photo</span>
    <input type="file" name="photoUpload">
</div>
```

然后监听它的change事件，在change事件里面给form套一个类：

```javascript
$(form.photoUpload).on("change", function() {
    // 如果选择了照片则添加一个photo-added的类
    this.value.length ? $(form).addClass("photo-added")
                // 否则去掉
                : $(form).removeClass("photo-added");
            
});
```

然后就可以来实现文案改变的需求了，把上面#add-photo的span标签添加两个data属性，分别是照片添加和未添加的文案，如下代码所示：

```html
<span id="add-photo" data-added-text="Photo Added" 
      data-notadded-text="Add Photo"></span>
```

通过form的类结合before/after伪类控制html上文案，如下代码所示：

```css
#add-photo:before {
    content: attr(data-empty-text);
}

form.photo-added #add-photo:before {
    content: attr("data-added-text);
}
```

这样就可以了，我们算是用了一个比较优雅的方式实现了一个文案变化的功能，其中css的attr可以兼容到IE9，并且这里html/css/js相配合，共同完成这个变化的功能，这应该也挺好玩的。

剩下一个要减掉23字符的需求，只需要在减掉的时候判断一下：

```javascript
$(form.textMsg).on("input", function() {
    // 已有字数
    let wordsCount = this.value.length;
    form.tweet.disabled = wordsCount <= 0;
    $leftWordCount.text(140 - wordsCount - 
            //如果已经添加了图片再减掉23个字符
            ($(form).hasClass("photo-added") ? 23 : 0));
});
```

然后再选择图片之后trigger一下，让文字发生变化，如下代码倒数第二行：

```javascript
/*
 * @trigger 会触发文字输入框的input事件以更新剩余字数
 */
$(form.photoUpload).on("change", function() {
    // 如果选择了照片则添加一个photo-added的类
    this.value.length ? $(form).addClass("photo-added") : 
                // 否则去掉
                $(form).removeClass("photo-added");
    $(form.textMsg).trigger("input");
});
```

这里又使用了事件的机制，用react应该基本上都是用状态state控制了。

再来看最后一个功能。

## （4）没有文字但是有照片发推按钮要可点击

上面是只要没有文字，那么发推按钮不可点击，现在要求有图片就可点击。这个也好办，因为如果有图片的话，form已经有一个类，所以只要再加一个判断就可以了：

```javascript
$(form.textMsg).on("input", function() {
    // 已有字数
    let wordsCount = this.value.length;
    form.tweet.disabled = wordsCount <= 0 
            //disabled再添加一个与判断
            && !$(form).hasClass("photo-added");
    $leftWordCount.text(140 - wordsCount - 
            //如果已经添加了图片再减掉23个字符
            ($(form).hasClass("photo-added") ? 23 : 0));
});
```

最后再看一下，汇总的JS代码，加上空行和注释总共只有23行：

```javascript
let form = $(".tweet-box")[0],
    $leftWordCount = $("#left-word-count");

$(form.textMsg).on("input", function() {
    // 已有字数
    let wordsCount = this.value.length;
    form.tweet.disabled = wordsCount <= 0 
            //disabled再添加一个与判断
            && !$(form).hasClass("photo-added");
    $leftWordCount.text(140 - wordsCount - 
            //如果已经添加了图片再减掉23个字符
            ($(form).hasClass("photo-added") ? 23 : 0));
});

/*
 * @trigger 会触发文字输入框的input事件以更新剩余字数
 */
$(form.photoUpload).on("change", function() {
    // 如果选择了照片则添加一个photo-added的类
    this.value.length ? $(form).addClass("photo-added") : 
            // 否则去掉
            $(form).removeClass("photo-added");
    $(form.textMsg).trigger("input");
});
```

html大概有10行，还有6行核心CSS，不过这两个比较易读。再来看一下React的完整版本，作者的实现：

```react
var TweetBox = React.createClass({
  getInitialState: function() {
    return {
      text: "",
      photoAdded: false
    };
  },
  handleChange: function(event) {
    this.setState({ text: event.target.value });
  },
  togglePhoto: function(event) {
    this.setState({ photoAdded: !this.state.photoAdded });
  },
  remainingCharacters: function() {
    if (this.state.photoAdded) {
      return 140 - 23 - this.state.text.length;
    } else {
      return 140 - this.state.text.length;
    }
  },
  render: function() {
    return (
      <div className="well clearfix">
        <textarea className="form-control"
                  onChange={this.handleChange}></textarea>
        <br/>
        <span>{ this.remainingCharacters() }</span>
        <button className="btn btn-primary pull-right"
          disabled={this.state.text.length === 0 && !this.state.photoAdded}>Tweet</button>
        <button className="btn btn-default pull-right"
          onClick={this.togglePhoto}>
          {this.state.photoAdded ? "✓ Photo Added" : "Add Photo" }
        </button>
      </div>
    );
  }
});

React.render(
  <TweetBox />,
  document.body
);
```

React的套路是监听事件然后改变state，在jsx的模板里，使用这些state展示，而jQuery的套路是监听事件，然后自己去控制DOM展示。React帮你操作DOM，jQuery要自己去操作DOM，前者提供了便利但同时也失去了灵活性，后者增加了灵活性但同时增加了复杂度。

使用jQuery不少人容易写出面条式的代码，但是写代码的风格我觉得和框架没关系，关键还在于你的编码素质，就像你用了React写class，你就可以说你就是面向对象了？不见得，我在《[JS与面向对象](https://fed.renren.com/2017/05/21/js-oop/)》这篇文章提到，写class并不代表你就是面向对象，面向对象是一种思想而不是你代码的组织形式。一旦你离开了React的框架，是不是又要回到面条式代码的风格了？如果是的话那就说明你并没有掌握面向对象的思想。不过，React等框架能够方便地组件化，这点是不可否认的。

还有一个需要注意的是，框架会帮你屏蔽掉很多原生的细节，让你专心于业务逻辑，但往往也让你丧失了原生的能力不管是HTML还是JS，而这才是最重要的功底。例如说对于事件，由于所有的事件都是直接绑在目标元素，然后通过state或者其他第三方的框架进行传递，这样其实就没什么事件的概念了。所以需要警惕使用了框架但是丧失了基本的前端能力，再如ajax分页改变url，或者说单页面路由的实现方式，还有前后退的控制，基本上能够完整回答的比较少。很多人都会用框架做页面，但是不懂JS。