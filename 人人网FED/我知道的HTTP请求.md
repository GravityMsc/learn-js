HTTP大家都不陌生，但是HTTP的许多细节就并不是很多人都知道了，本文将讨论一些容易忽略但又比较重要的点。

首先，怎么用原生JS写一个GET请求呢？如下代码，只需3行：

```javascript
let xhr = new XMLHttpRequest();
xhr.open("GET", "/list");
xhr.send();
```

xhr.open第一个参数是请求方法，第二个参数是请求url，然后把它send出去就行了。

如果需要加上请求参数，比如用jQuery的ajax，那么是这么写的：

```javascript
$.ajax({
    url: "/list",
    data: {
        page: 5
    }
});
```

如果是用原生的话就得拼接在请求url上面，即open的第二个参数：

![拼接参数](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b4b4f796?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

并且参数需要转义，如下代码所示：

```javascript
function ajax(url, data){
    let args = [];
    for(let key in data){
        // 参数需要转义
        args.push(`${encodeURIComponent(key)} = ${encodeURIComponent(data[key])}`);
    }
    let search = args.join("&");
    // 判断当前url是否已有参数
    url += ~url.indexOf("?") ? `&${search}` : `?${search}`;
    
    let xhr = new XMLHttpRequest();
    xhr.open("GET", url);
    xhr.send();
}
```

那为什么用jQuery就不用转义了呢？因为jQuery帮我们做了转义，jQuery的ajax支持一个叫processData的参数，默认为true：

```javascript
$.ajax({
    url: "/list",
    data: {
        page: 5
    },
    processData: true
});
```

这个参数的作用是用来处理传进来的data的，如下jQuery的源码：

![jQuery源码](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b560fcad?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果传了data，并且processData为true，并且data不是一个string了，就会调用param处理data。然后我们来看下这个param函数是怎么实现的：

![jQuery实现转义](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b472d7fd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到，它也是跟我自己实现的ajax类似，把key和value转义用“=”拼接，然后push到一个数组，最后再join起来。不一样的地方是它的判断逻辑比我的复杂，它会再调用一个buildParams的函数去处理key/value，因为value可能是一个数组，也可能是一个Object。如果value是一个Object，那么直接encode一下就会变成“[object Object]”的转义了：

![ecode的结果](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b4d8329d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

所以buildParams在处理每个key/value时，会先判断当前value是否是一个数组或者是Object，如下图所示：

![对value的判断](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b4c9d882?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果它是一个数组的话，这个数组的每一个元素都会变成单独的一个请求字段，它的key是父级的key拼上数组的索引得到，如{id: [1, 2, 3]}就会被拼成：ids[0] = 1、ids[1] = 2、ids[2] = 3，如果是一个Object的话key的后缀就是子Object的key，如{user: {id: 1333, name: "yin"}}会被拼成：user[id] = 1333、user[name] = yin，否则就认为它是一个简单的类型，就直接调用一下params函数定义的add，把它push到s那个数组。这里用到了递归调用，会不断地拼接key值，直到value是一个普通变量了，然后就到了最后面的else的逻辑。

也就是说，以下代码：

```javascript
$.ajax({
    url: "/list",
    data: {
        user: {
            name: "yin",
            age: 18
        }
    },
});
```

将会被拼成的url为：

/list?user[name]=yin&user[age]=18

注意上面的中括号还没有转义。而如果是一个数组的话：

```javascript
$.ajax({
    url:"/list",
    data: {
        ids: [1, 2, 3]
    },
});
```

拼成的请求url为：

/list?ids[0]=1&ids[1]=2&ids[2]=3

如果后端的Java的Spring MVC框架的话，是理解这种格式的，框架收到这样的参数会生成一个Array，传递给业务代码，业务代码是不用关心怎么处理这种参数的。其他的框架应该也类似。

怎么用原生JS写一个POST请求呢？如下图所示：

![用JS写一个原生的POST请求](https://user-gold-cdn.xitu.io/2018/2/3/1615af11b4e2f56f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

POST请求的参数就不是放在url了，而是放在send里面，即请求体。你可能会问：难道就不能放url么？我就要放url。如果你够任性，那么可以，前提是藕断所使用的http框架能够在url里面去数据，因为它一定会收到url，也一定会收到请求体，所以取决于它要怎么处理，按照http标准，如果请求方法是POST，那么应该是取请求体拿的，就不会再url的search上取了，当然它可以改一下，改成两个都可以拿。

然后我们会发现请求的mime类型时text/plain：

![POST请求的请求体](https://user-gold-cdn.xitu.io/2018/2/3/1615af11fe7bfa62?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

并且查看请求参数的时候，并不是平时所看到能够按照字段一行行地展示：

![通常的字段展示方法](https://user-gold-cdn.xitu.io/2018/2/3/1615af1200769315?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这是为什么呢？这是因为我们没有设置它的Content-Type，如下代码：

```javascript
let xhr = new XMLHttpRequest();
xhr.open("POST", "/add");
xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
xhr.send("id=5&name=yin");
```

如果设置Content-Type为x-www-form-urlencoded的话，那么在检查的话，Chrome也会按字段分行展示了。

![设置了Content-Type之后](https://user-gold-cdn.xitu.io/2018/2/3/1615af11fd50ae59?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个也是jQuery默认的Content-Type：

![jQuery的默认Content-Type](https://user-gold-cdn.xitu.io/2018/2/3/1615af11fce1f7de?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

它是一种最常用的请求编码方式，支持GET/POST等方法，特点是：所有的数据编程键值对的形式key1=value1&key2=value2的形式，并且特殊字符需要转义为utf-8编号，如空格会变成%20：

![特殊字符的转义](https://user-gold-cdn.xitu.io/2018/2/3/1615af1201471d48?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

由于中文在utf-8需要占用3个字符，所以它有3个%符号。

