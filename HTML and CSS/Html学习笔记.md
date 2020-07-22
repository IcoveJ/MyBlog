## 一.HTML与CSS

### 1.HTML是网页内容的载体

用于制作网页上的信息，比如说文字、图片、视频。

### 2.CSS样式是表现

用于制作网页的“外衣”，比如说标题的字体，颜色变化或为标题加入背景图片、边框等。

### 3.JavaScript是用来实现网页上的特效效果

用于实现页面上的动画、交互等，比如说鼠标滑过弹出下拉菜单、焦点新闻的轮换、鼠标滑过表格背景颜色的改变。

## 二.HTML标签

### 1.标签由英文尖括号”<“和”>”括起来，如<html>就是一个标签。

### 2.html中的标签一般都是成对出现的，分开始标签和结束标签。结束标签比开始标签多了一个”/”。

例如:

```
<p></p>;
<div></div>;
<span></span>
```

### 3.标签与标签之间是可以嵌套的，但先后顺序必须保持一致，如：<div>里嵌套<p>，那么</p>必须放在</div>的前面。

例如：

```
<div><p>是分区标签，用于页面布局，相当于一个大的“容器”，可以容纳无序列表，有序列表，表格，表单等块级标签，同时可以容纳普通标题，段落，文字，图片等内容。</p></div>
```

### 4.HTML的标签最好是小写。

## 三、html文件的基本结构

### 1.<html></html>为根标签。

### 2.<head>用于定义文档的头部，为所有标签的容器。

头部元素有：<title><scripe><style><link><meta>

### 3.<body></body>之间为网页的主要内容。

如：<h1>、<p>、<a>、<img>

## 四、标签

### 1.<head>:文档的头部描述文档的各种属性和信息，包括文档的标题

```
<head>
    <title>......</title>
    <meta>
    <link>
    <style>......</style>
    <scripe>......</scripe>
</head>
```

### 2.<title>：<title></title>之间的文字内容是网页的标题信息，出现浏览器的标题；

如：

```
<head>
    <title>hello world</title>
</head>
```

hello world就会出现在网页最上方的标题栏。

### 3.<body>：网页上展示的页面内容都要放在body标签中

例如：

```
<body>
    <h1>了不起的盖茨比</h1>
    <p>1922年的春天，一个想要成名名叫<em>尼克•卡拉威</em>（托比•马奎尔Tobey Maguire 饰）的作家，离开了美国中西部，来到了纽约。那是一个道德感渐失，爵士乐流行，走私为王，股票飞涨的时代。为了追寻他的<span>美国梦</span>，他搬入纽约附近一海湾居住。</p>
    
    <p>菲茨杰拉德，二十世纪美国文学巨擘之一，兼具作家和编剧双重身份。他以诗人的敏感和戏剧家的想象为<strong>"爵士乐时代"</strong>吟唱华丽挽歌，其诗人和梦想家的气质亦为那个奢靡年代的不二注解。</p>
</body>
```

### 4.<p>：显示一篇文章的段落内容，有几个段落，就用个几个<p>

以②中的例子为例，第一个<p></p>显示为第一段，第二个<p></p>显示第二段。

```
<p>这是第一段</p>
<p>这是第二段</p>
```

### 5.<hx>：用于制作文章的标题

一共有六级<h1>、<h2>、<h3>、<h4>、<h5>、<h6>

```
<h1>标签在网页中比较重要，一般用于网站名称，比如<h1>腾讯网</h1>；
h1-h6标签的样式：
<body>
    <h1>一级标题</h1>
    <h2>二级标题</h2>
    <h3>三级标题</h3>
    <h4>四级标题</h4>
    <h5>五级标题</h5>
    <h6>六级标题</h6>
</body>
字体依次缩小！！！
```

### 6.<strong>和<em>：用于强调一段话中的某些文字，是对语义的强调

<strong>：粗体表示

```
<strong>需要强调的文本</strong>
```

<em>：斜体表示

```
<em>需要强调的文本</em>
```

### 7.<span>：没有语义作用，只是为了设置单独的样式

比如说，“为了追寻他的中国梦”，将“中国梦”设置为蓝色，就是用<span>；

```
<!DOCTYPE HTML>
<html>
    <head>
        <meta charest="utf-8">
        <style>
            <span>{color:blue;}</style>
    </head>
    <body>
        <p>为了追寻他的<span>中国梦</span>，他搬回了上海</p>
    </body>
</html>
```

<span>中设置的样式要在<head>的<style>中进行说明

### 8.<q>：quote的缩写，对内容文本进行引用，但是是简短文本的引用

要注意，引用的文本不需要加双引号，浏览器会对q标签的内容**自动**加上双引号

例如：最初知道庄子，是从一首诗庄生晓梦迷蝴蝶。望帝春心托杜鹃。开始的。对”庄生晓梦迷蝴蝶。望帝春心托杜鹃。“加上引用，则为：

```
<p>最初知道庄子，是从一首诗<q>庄生晓梦迷蝴蝶。望帝春心托杜鹃。</q>开始的。</p>
```

### 9.<blockquote>：同样是对内容文本的引用，但是是对长文本的引用

注意，浏览器在对<blockquote>进行解析时，是缩进样式，两个字符，不会有引号的出现

```
<!DOCTYPE HTML>
<html>
    <head>
        <meta charest="utf-8">
        <title>bolockquote标签的使用</titletitle>
    </head>
    <body>
        <h2>心似桂花开</h2>
        <p>大家都在忙于自认为最重要的事情，却没能享受到人生的乐趣，反而要吞下苦果？</p>
        <blockquote>暗淡轻黄体性柔，情疏迹远只香留。何须浅碧深红色，自是花中第一流.</blockquote>
        <p>这是李清照《咏桂》中的词句，在李清照看来，桂花暗淡青黄，性情温柔，淡泊自适，远比那些大红大紫争奇斗艳花值得称道。</p>
    </body>
</html>
```

### 10 :分行显示文本，即回车换行，这是个空元素标签，即自身关闭

```
<h2>《咏桂》</h2>
<p>暗淡轻黄体性柔，<br/>情疏迹远只香留。<br/>何须浅碧深红色，<br/>自是花中第一流.</p>
```

其效果为：

《咏桂》

暗淡轻黄体性柔，

情疏迹远只香留。

何须浅碧深红色，

自是花中第一流.

注意，在html中是忽略回车和空格的，例如：

```
<h2>《咏桂》</h2>
<p>
    暗淡轻黄体性柔，
    情疏迹远只香留。
    何须浅碧深红色，
    自是花中第一流.
</p>
```

其效果为：

《咏桂》

暗淡轻黄体性柔，情疏迹远只香留。何须浅碧深红色，自是花中第一流.

### 11.空格：在11中提到html的代码中输入空格与回车都是没有作用的，要想输入空格，必须写入&nbsp

在html代码中无论输入多少个空格，在网页中显示都只显示一个空格；在html代码中，写入多少个&nbsp，网页中才会出现多少个空格（相邻&nbsp是否用“；”间隔都行）。例如：

```
<h2>《咏桂》</h2>
<p>
    暗淡&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;轻黄体性柔，情疏迹远只香留。何须浅碧深红色，自是花中第一流.
</p>
```

其效果为在”暗淡“与”轻黄体性柔“之间间隔五个空格

### 12.<hr>：添加水平分割横线

```
<hr>同样是一个空标签

<p>火车飞驰过暗夜里的村庄，月光，总是太容易让思念寂寞，太容易让人觉得孤独。</p>
<hr/>
<p>每一枚被风吹起的蒲公英，都载满了一双眼睛的深情告别与一个目光的依依不舍。那天，我拿着行李，带上一个背影的祝福与惆怅，挥手告别了这片土地。我不知道，我何时会回来。</p>
```

### 13.<address>：为网页加入地址信息

在浏览器上显示的样式为**斜体**，也可以定义一个地址、签名或者文档的作者身份

```
<address>联系地址信息</address>
<address>本文的作者：<a href="mailto:lilian@imooc.com">lilian</a></address>
```

### 14.<code>:加入一行代码

```
<code>代码语言</code>
例如:
<code>var i=i+300;</code>
```

### 15.<pre>：加入代码块

```
<pre>语言代码段</pre>
例如：
<pre>
var message='欢迎';
for(var i = 1; i <= 10; i++)
{
alert(message);
}
</pre>
```

此时，代码中的空格、回车都会被保留，不需要输入&nbsp；

注意：被<pre>标签包含的文本会呈现为等宽字体;

### 16.<ul>：添加新闻列表（无序标签）

```
<ul>
    <li>信息</li>
    <li>信息</li>
    .......
</ul>
例如：
<ul>
    <li>精彩少年</li>
    <li>美丽突然出现</li>
    <li>触动心灵的旋律</li>
</ul>
```

其效果为：

•精彩少年

•美丽突然出现

•触动心灵的旋律

### 17.<ol>:添加信息列表（有序标签）

```
<ol>
    <li>信息</li>
    <li>信息</li>
    .......
</ol>
例如：
<ol>
    <li>精彩少年</li>
    <li>美丽突然出现</li>
    <li>触动心灵的旋律</li>
</ol>
```

其效果为：

1.精彩少年

2.美丽突然出现

3.触动心灵的旋律

每项<li>前都自带一个序号，序号默认从1开始

### 18.<div>：用于排版

在网页制作过程过中，可以把一些独立的逻辑部分划分出来，放在一个<div>标签中，这个<div>标签的作用就相当于一个容器。

```
<div>......</div>
```

#### ①什么是独立的逻辑部分？

比如：网页中的独立的栏目板块……

#### ②如何给div命名，使逻辑更加清晰？

```
<div id="板块名称">......</div>
但是板块名称在浏览器中并不会显示
```

### 19.<table>：网页上的表格

创建表格的四个元素：

#### ①<table>…</table>：表示整个表格以<table>标记开始，以</table>>标记结束；

#### ②<tbody>…</tbody>：表格的内容部分；

#### ③<tr>…</tr>：表格的行；

#### ④<td>…</td>：表格的单元格，放在<tr>…</tr>之间，包含几对就说明这一行有几列；

#### ⑤<th>…</th>：表头头部的一个单元格，就是第一行的内容，第一行的每个单元格不用<td>，而用<th>，例如：

```
<table>
    <tbody>
        <tr>
            <th>学生姓名</th>
            <th>性别</th>
            <th>班级</th>
            <th>成绩</th>
        </tr>
    </tbody>
</table>
```

★特别：如何为表格加上表格框架呢？

##### ①使用<table border=”1″>；

##### ②添加css样式：

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style type="text/css">
    table tr td,th{border:1px solid #000;}
    </style>
    <title>表格尝试</title>
</head>
<body>
    <table summary="">
        <tbody>
            <tr>
                <th>班级</th>
                <th>学生数</th>
                <th>平均成绩</th>
            </tr>
        </tbody>
    </table>
    </body>
</html>
```

即在<head>里面添加

```
<style type="text/css">
table tr td,th{border:1px solid #000;}
</style>
```

#### ⑥<caption>：为表格添加标题和摘要

```
<table summary="表格的摘要">
    <caption>表格的标题</caption>
    <tr>
        <th>...</th>
    </tr>
</table>
```

### 21.<a>：实现超链接

```
<a  href="目标网址"  title="鼠标滑过显示的文本">网页界面显示的文本</a>
例如：
<a href="http://csdn.net" title="CSDN">点击进入CSDN</a>
```

如何在新建浏览器窗口打开链接？

```
<a href="http://csdn.net" target="_blank" title="CSDN">点击进入CSDN</a>
```

在当前窗口打开链接：

```
<a href="http://csdn.net" target="_self" title="CSDN">点击进入CSDN</a>
```

同时，<a>也能实现与邮件地址相连：

```
1.mailto：<a href="mailto:1443162545@qq.com">发送</a>
点击页面上的“发送”，会调用系统默认的客户端电子邮件程序，并在收件人一栏自动填上mailto：后接的地址；   ——————注意：mailto后接多个参数时，第一个参数必须以“？”开头，后面的参数以“&”分隔；
2.cc=：<a href="mailto:1443162545@qq.com?cc=473264150@qq.com">发送</a>
点击页面上的“发送”，在发给144。。。的同时，将副本发给473。。。
3.bcc=： <a href="mailto:1443162545@qq.com?bcc=473264150@qq.com">发送</a>
大体上与cc相同，区别在于cc发给的主收件人有权限知道发送给的副收件人(第一个地址为主收件人)，而bcc的主收件人没有权限；
4.subject=：<a href="mailto:1443162545@qq.com?subject=发送电子邮件">发送</a>
给发送的邮件添加主题"发送电子邮件"
5.body=：<a href="mailto:1443162545@qq.com?body=发送电子邮件">发送</a>
给发送的邮件添加内容"发送电子邮件"
```

### 22.<img>:为网页插入图片

```
<img src="图片的绝对地址" alt="下载失败时的替换文本" title="提示文本">
例如：
<img src = "myimage.gif" alt = "My Image" title = "My Image" />
```

#### ①src:标识图像的位置

#### ②alt:src指定的图像的描述性文本，当图像因为其他原因不可见（比如下载不成功）时，可见到该图像的描述性文本；

#### ③title:鼠标滑过图像时显示的文本；

### 23<header>:定义头部区域

即定义网页中常见的最顶部的那一区域；作用等同于div

### 24<footer>:定义底部区域

即定义网页最底部区域；作用等同于div

### 25<section>:定义一个区域

比如说一个网站中的专栏部分；作用等同于div

### 26<aside>:定义一个侧边栏区域

比如说一个网站中侧栏部分；作用等同于div

## 三.<form>表单标签：实现网站与用户的数据交互，通过表单可以把浏览者输入的数据传送到服务器端，服务器端的程序就可以处理表单传来的数据

```
<form method="传送方式" action="服务器文件">...</form>
例如：
<form method="post" action="save.php">
    <label for="username">用户名:</label>
    <input type="text" name="usrname" />
    <label for="pass">密码:</label>
    <input type="password" name="pass" />
</form>
```

### 1.<form>：

成对出现

### 2.method：

数据传送的方式，只有**get**和**pos**t两种，默认为**get**，一般写为**post**

### 3.action：

浏览者输入的数据被传送到的地方，比如一个PHP页面

### 5.文本输入框和密码输入框：

```
<form>
    <input type="text/password" name="名称" value="文本" />
</form>
```

#### ①type:

当**type=“text”**；输入框为**文本输入框**；

当**type=“password”**；输入框为**密码输入框**；

#### ②name：

为文本框命名，以备后台程序ASP、PHP使用

#### ③value：

为文本输入框设置默认值，比如说：

```
<label for="pass">密码:</label>
<input type="password" name="pass" value="请输入密码" />
则在文本框中会出现隐藏密码的的小黑点
<label for="username">用户名:</label>
<input type="text" name="username" value="请输入姓名" />
则在文本框中会出现“请输入姓名”
```

★注意：在使用input前，最好先使用label；

### 6.文本域支持多行输入

```
<textarea rows="行数" clos="列数">文本</textarea>
```

#### ①<textarea>是成对出现；

#### ②rows，cols分别为行数，列数；可以交换位置；

#### ③在<textarea></textarea>之间可以输入默认值；

### 7.使用单选框、复选框，让用户选择

```
<input type="radio/checkbox" value="值" name="名称" check="checked" />
```

#### ①type：

当**type=“radio”**时，控件为**单选框**，为一个圆圈；

当**type=“checkbox”**时控件为**复选框**，为一个正方形框；

#### ②value：

提交数据到服务器的值（后台程序PHP使用）,value后接的文本为单选框/复选框后接的文本，比如

```
<input type="radio" name="radiolove" value="喜欢" checked="checked">喜欢
(后一个喜欢时显示的值，前一个喜欢是向服务器提交的值)
```

#### ③name：

为控件命名，以备后台程序 ASP、PHP 使用；但是单选框同一组的按钮的name必须一致；复选框不能一致

#### ④checked：

当设置checked=”checked”时，该选项被默认选中，即框内为黑

### 8.使用下拉列表框，节省空间

```
<form action="save.php" method="post">
    <label>爱好：</label>
    <select>
        <option value="看书">看书</option>
        <option value="旅游">旅游</option>
        <option value="运动">运动</option>
    </select>
</form>
```

#### ①value：

```
<option value="向服务器提交的值">选项显示的值</option>
```

#### ②selected=selected：

选项被默认选中，即页面优先显示的选项内容

#### ③在<select>标签中加入multiple=”multiple”，可以实现多选功能

在Windows系统中，进行多选时按下ctrl+鼠标左键单击；不能与seleted=“selected”同时使用

```
<select multiple="multiple">
```

### 9.<input>的提交按钮，提交数据

```
<input type="submit" value="提交">
```

#### ①type：

只有当type设置了submit时，按钮才有提交作用

#### ②value：

“文本”为网页上按钮显示的文字

### 10.<input>的重置按钮，重置表单信息

```
<input type="reset" value="重置">
```

type与value的作用与9一致

### 11.<form>的<label>标签：

label标签不会向用户呈现任何特殊效果，当用户单击选中该标签，浏览器会自动将焦点转到和标签相关的表单控件上；

```
<label for="控件id的名称">
注意：标签中for属性中的值一定要与相关控件的id属性值相同
例如：
<form>
    <label for="male">男性：</label>
    <input type="radio" name="gender" id="male" />
</form>
```

### 12.placeholder：输入框占位符

可以为输入框添加提示的输入信息

```
例如：
姓名：
<input type="text" name="myname" placeholder="请输入用户名">
</br>
密码：
<input type="password" name="password" placeholder="请输入密码">
```

### 13.数字输入框：将输入框设置为只能输入数字

```
格式为：<input type="number">
```

### 14.网址输入框：将输入框设置为只能输入网址

注意：输入框的内容必须以http://或者https://开头,且后面必须要有内容，否则表单提交的时候会报错

```
格式为：<input type="url">
```

### 15.邮箱输入框：将输入框设置为只能输入邮箱地址

注意：输入框的内容必须包含@，且其后必须要有内容，否则提交的时候会报错

```
格式为：<input type="email">
注意将type的属性值设置为email时，中间没有“—”
```

## 四、CSS样式基本知识

主要用于**定义**HTML内容在浏览器内的显示样式，如文字大小、颜色、字体加粗等。

好处是：通过**定义**某个样式，可以让不同网页位置的字体有着统一的字体、字号或者颜色。

### 1.语法：

```
<style type="text/css">
    p{color:blue;
    font-size:12px;}
    span{color:red}
</style>
<body>
    <p><span>慕课网</span>，超酷的互联网、IT技术免费学习平台.</p>
</body>
```

css样式由**选择符**和**声明**组成，**声明**又由**属性**和**值**组成

以上述代码为例：

①、p为选择符，又称选择器，其作用是指明网页中要应用样式规则的元素。在上例中<p>之间的内容为12号+蓝色字体；<span>之间的内容为红色字体；

②、{}内的内同容为声明，属性和值之间用：号隔开，每条声明之间用；隔开；

### 2、css样式的书写位置

#### ①内联式

```
<p style="color:red">这里的文字是红色</p>
```

同样，若有多条css样式代码设置，可以同时写在style中，用；号隔开；

#### ②嵌入式

```
即在第一点语法中的例子
<style type="text/css">
    p{color:blue;
    font-size:12px;}
</style>
就是将css样式代码写在<style type="text/css"></style>之间，可以简化多处字体的设置；
```

#### ③外部式

之前两种，内联式写在<body>内；嵌入式写在<head>的<style>内；而外部式则写在<head>的<link>内，这种写法以.css为扩展名；

```
<link href="base.css" rel="stylesheet" type="text/css">
```

★三种方法的优先级：

对于同一个元素我们同时用了三种方法设置css样式，**内联式 > 嵌入式 > 外部式**；其中**嵌入式>外部式**有一个前提：嵌入式css的样式的位置必须在外部式的后面，即：

```
<link href="style.css" ......>
.....
<style type="text/css">...</style>
```

总的来说就是**”就近原则“**，离被设置元素越近优先级别越高

### 3、选择器

每一条css样式声明（定义）由两部分组成，形式如下：

```
选择器{
样式；
}
```

{}之前的部分为“选择器”，”选择器“指明了{}中”样式“的作用对象，即”样式“作用于网页的元素；比如：

```
<style type="text/css">
body{
    font-size:12px;
    color:red;
}
    p{
        font-size:12px;
        line-height:1.6em;
    }
</style>
```

#### ①标签选择器：

如上例中的标签，还有<html>、<h1>、<img>等

#### ②类选择器：

```
.类选器名称{css样式代码;}
```

类选器名称可以任意取名，除了中文；必须以英文句号开头。

在使用内联式，比如：

```
<style type="text/css">
    .stress{
        color:red;
    }
</style>
<body>
    <p>123456<span class="stress">789</span>abcdef987654321</p>
```

#### ③ID选择器：

类比于类选择器：①、为标签设置为id=”ID名称“，而不是class=”类名称”；

②、ID选择符的前面是#，而不是.

```
<style type="text/css">
    #stress{
        color:red;
    }
</style>
<body>
    <p>123456789<span id="stress">David</span>987654321</p>
</body>
```

★ID选择器与类选择器的区别： 1、同一个ID选择器只能在文档中使用一次；而类选择器可以使用多次；

2、可以使用类选择器列表的方法为一个元素同时设置多个样式；ID选择器不行；

```
.stress{
color:red;
}
.bigsize{
font-size:25px;
}
<p>123456789<span class="stress bigsize">IcoveJ</span>987654321
```

#### ④子选择器

使用大于符号>，用于选择指定标签元素的**第一代子元素**，样式为：

```
.food>li{border:1px solid red;}
```

例如

```
<style type="text/css">
    .food>li{border:1px solid red;}
</style>
<body>
    <ul class="food">
        <li>水果//这是第一代子标签
            <ul>
                <li>香蕉</li>
                <li>苹果</li>
                <li>梨</li>
            </ul>
        </li>
    </ul>
</body>
```

效果为：

#### ⑤包含选择器：

将子代选择器中的大于符号改为空格，样式为：

```
.first span{color:red;}
```

用于选择指定标签元素下的所有后辈元素。例如：

```
<style type="text/css">
    .food li{border:1px solid red;}
</style>
<body>
    <ul class="food">
        <li>水果//这是第一代子标签
            <ul>
                <li>香蕉</li>
                <li>苹果</li>
                <li>梨</li>
            </ul>
        </li>
    </ul>
</body>
```

效果为：

#### ⑥通用选择器

功能最强大的选择器，它使用一个*号指定，作用是匹配html中的所有标签元素

```
*{color:red;}
```

#### ⑦伪选择符

允许给html不存在的标签（标签的某种状态）设置样式，比如说，给html中一个标签元素的鼠标滑过的状态来设置字体颜色：

```
a:hover{color:red;}
```

例子中的代码就是为a标签鼠标滑过的状态设置字体颜色变红

伪选择符通常应用于<a>,它表示4种不同的状态：link(未访问链接)、visited(已访问链接)、active(激活链接或者说链接被点击的时候)、hover(鼠标停留在连接上)

```
a:link{}
a:visited{}
a:hover{}
a:active{}
```

#### ⑧分组选择符

为html中的多个标签元素设置同一选择符，样式为

```
标签,标签{样式;}
```

例如：

```
h1,span{color:red;}
等价于：
h1{color:red;}
span{color:red;}
```

注意若分组选择符与包含选择符同时使用时，写法的样式。见例子：

```
.first,#second span{color:red;}
语句的意思是：.first这个类选择器的内容变红，#second这个ID选择器的span标签内容变红；不要将其理解为：.first的span与·#second的span同时变红
```

★选择器的优先级

内联样式 > id选择器 > 类选择器 > 标签选择器 > 通配符选择器

## 五、CSS的继承、层叠和特殊性

### 1、继承

CSS的**某些样式**是具有继承性的，它允许该样式不仅应用于某个特定的html标签元素，而且应用于其后代。比如：

```
p{color:red;}
<p>三年级时，我还是一个<span>胆小如鼠</span>的小朋友</p>

这个颜色设置不仅适用于<p>标签，还适用于它的子标签<span>
p{border:1px solid red;}
<p>三年级时，我还是一个<span>胆小如鼠</span>的小朋友</p>

这个样式的设置就对于<p>的子标签<span>没有起到作用
```

### 2、特殊性

当同时为同一个元素设置了不同的css样式代码，那么元素会启用哪一个css样式？

例如：

```
p{color:red;}
.first{color:green;}
<p class="first">三年级时，我还是一个胆小如鼠的小女孩</p>
```

此时，浏览器会根据**权值**来判断使用哪种样式

**权值的规则为：标签的权值为1，类选择符权值为10，ID选择符的权值最高为100，继承权值最低**

```
p{color:red;}/*权值为1*/
p span{color:red;}/*权值为1+1=2*/
.warning{color:red;}/*权值为10*/
p span .warning{color:red;}/*权值为1+1+10=12*/
#footer .note p{color:red;}/*权值为100+10+1=111*/

权值相同时，后者优先
```

因此，在上例中，会优先选择green样式

### 3、层叠

在html文件中对于同一个元素可以有多个css样式存在，当有相同权重的样式存在时，会根据这些css样式的前后顺序来决定，处于最后面的css样式会被应用

```
p{color:red;}
p{color:green;}
<p>三年级时，我还是一个<span>胆小如鼠</span>的小女孩</p>
最后p中的文本会显示为green，即，后面的样式会覆盖掉前面的样式
```

又反过来证明了优先级：内联式样表（在标签内部）> 嵌入式样表（当前文件中）> 外部式样表（外部文件中）

### 4、重要性

为某些样式设置最高权值，可以使用important

```
例如：
p{color:red!important;}
p{color:green;}
<p>......</p>
p段落中的文本会显示red
```

*!important要写在；之前

## 六、CSS格式排版

```
以下样式可以缩写为一句：
body{font:italic bold 12px/1.5em "宋体",sans-serif;}

在缩写时，要注意：
（1）简写时至少要指定font-size和font-family的属性，其他的font属性如果未指定，将自动使用默认值；
（2）在缩写时font-size和line-height中间要加入“/”斜杠；
（3）
```

### 1、文字排版–字体

```
body{font-family:"微软雅黑";}或者body{font-family:"Microsoft Yahei"}

font-family

注意1：设置的字体要是常用的，且，用户能看到我们设置的字体样式取决于用户本地电脑上是否安装了我们设置的样式
注意2：用英文比用中文的兼容性更好
```

### 2、文字排版–字号、颜色

```
body{font-size:12px;color:#666;}

设置为12像素（不是12号）字体，颜色为灰色（#666为灰色）

font-size


color的设置有三种
（1）英文命令颜色：p{color:red}；
（2）RGB颜色：与photoshop中的RGB颜色一致，由R（红色）G（绿色）B（蓝色）按比例来配色，p{color:rgb(133,45,200);}或者p{color:rgb(20%,33%,25%);}每一项值可以是0~255之间的整数，也可以是0%~100%间的百分数；
（3）十六进制颜色：这种设置方式目前更普遍使用，其原理也是RGB设置，但是每一项的值由0~255变成了十六进制的00~ff。p{color:#00ffff}.
```

### 3、文字排版–粗体

```
p span{font-weight:bold;}
```

### 4、文字排版–斜体

```
p a{font-style:italic;}
```

### 5、文字排版–下划线

```
p a{text-decoration:underline;}
```

### 6、文字排版–删除线

```
.oldPrice{text-decoration:line-through}
```

### 7、文字排版–上顶线

```
span{text-decoration:overline;}
```

### 8、段落排版–缩进

```
p{text-indent:2em;}
```

### 9、段落排版–行间距（行高）

```
p{line-height:1.5em;}
```

### 10、段落排版–中文字间距、字母间距

```
单个字或字母的间距：h1{letter-spacing:50px;}
英文单词的间距：h1{word-spacing:50px;}
```

### 11、段落排版–对齐

```
居中：h1{text-aligen:center;}
居左：h1{text-aligen:left;}
居右：h1{text-aligen:right;}
```

### 12、长度值px、em、%

这三个单位都是相对单位

#### ①px：像素，90像素=1英寸

#### ②em：本元素给定字体的font-size值，如果元素的font-size为14px，那么1em=14px；如果元素的font-size为18px，那么1em=18px；例如：

```
p{font-size:12px;text-indent:2em;}
就可以实现段落首行缩进20px。

特别：当给font-szie设置单位为em时，此时的标准以该标签的父元素的font-size为基础，如：
html：<p>以这个<span>例子</span>为例.</p>
css:p{font-size:14px;}
    span{font-size:0.8em;}
结果span中的字体“例子”字体大小就为14*0.8=11.2px；
```

#### ③百分比

```
p{font-size:12px;line-height:130%}
设置行高（行间距）为字体12*1.3=15.6px
```

## 七、CSS盒模型

常见的使用盒模型的标签有：

```
<div><ul><ol><p><h><table>
```

### 1、元素分类–块级元素

在html中，<div><p><hx><form><ul><li>就是**块级元素**，其特点是：①一个块级元素独占一行；②元素的高度、宽度、行高以及顶和底边距都可设置；③元素宽度在不设置的情况下，是它本身父容器的100%（和父元素的宽度一致），除非设定一个宽度。

```
使用display:block可以将其他元素设置为块级元素
a{display:block;}
```

### 2、元素分类–内联元素

在html中，<span>、<a>、<label>、 <strong> 和<em>就是典型的**内联元素**（**行内元素**）（inline）元素，其特点是：①和其它元素都在一行上；②元素的高度、宽度及顶部和底部边距不可设置；③元素的宽度就是它包含的文字或图片的宽度，不可改变。

```
使用display:inline可以将其他元素设置为块级元素
div{display:inline;}
```

### 3、元素分类–内联块状元素

即同时具备内联元素和块级元素的特点，如：<img><input>；inline-block 元素特点：①和其他元素都在一行上；②元素的高度、宽度、行高以及顶和底边距都可设置

```
使用display:inline-block可以将其他元素设置为内联块级元素
```

### 4、盒子模型的属性

content:width宽，height高。这里的宽与高是指填充以里的内容范围，一个元素的实际宽度（盒子的宽度）=左边 界+左边框+内容宽度+右填充+右边框+右边界；高度同理。

padding：内边距；

margin：外边距；

border：边框；

例如：

```
css：
div{
width:200px;
padding:20px;
border:1px solid red;
margin:10px;
}

html:
<body>
    <div>文本内容</div>
</body>
```

那么元素的实际宽度为：10+1+20+200+20+1+10=262px

### 5、为行内元素或者块状元素设置背景色

为标签设置背景色使用“background-color:颜色值”来实现，例如：

```
div{background-color:red;}//为块状元素设置
a{bakground-color:green;}//为行内元素设置
```

### 6、使用border为盒子添加边框（一）

盒子模型的边框就是围绕着内容及补白的线，这条线你可以设置它的粗细、样式和颜色(边框三个属性)；例如：

```
div{
border:2px solid red;
}
或者
div{
border-width:12px;
border-style:solid;
border-color:red;
}
```

#### ①border-style的常见样式有：

dashed(虚线)、dotted(点线)、solid(实线)、hidden(隐藏)、double(双线)、groove(凹槽边框)、ridge(垄状边框)、inset(嵌入边框)、outset(外凸边框)、none(没有)；

可以分别为四个方向设置不同的样式：border-left/right/bottom/top-style;

#### ②border-color的颜色可设置为十六进制颜色

例如：border-color:#888;//不要忘掉井号；

可以分别为四个方向设置不同的颜色：border-left/right/bottom/top-color;

#### ③border-width中的宽度可以设置为：

thin、medium、thick，最常用的还是像素（px）；

### 7、使用border为盒子添加边框（二）

使用border-xxx，单独为标签设置某一边的边框，例如：

```
div{border-bottom:1px solid red;}
或者div{border-right:1px solid red;}
或者div{border-left:1px solid red;}
或者div{border-top:1px solid red;}
```

### 8、为边框设置圆角

使用border-radius来设置左上、左下、右上、右下的圆角效果，注意：左上与右下相对应，左下与右上相对应；例如：

```
div{
    border-top-left-radius: 20px;
   border-top-right-radius: 10px;
   border-bottom-right-radius: 15px;
   border-bottom-left-radius: 30px;
}
若四个圆角的像素值一样，则可缩写为
div{border-radius:10px;}
若左上、右下相同，左下、右上相同，则可缩写为
div{border-radius:10px 20px;}效果为左上与右下为10px,左下与右上为20px
特别注意：
当圆角的效果值设置为盒子宽度的一半时，显示的效果为圆。
```

### 9、使用padding为盒子设置内边距（填充）

填充的数值按照顺时针方向（上、右、左、下），例如：

```
div{padding:20px 10px 15px 30px;}切记不要搞混了顺序
若四个方向填充一致，则为：
div{padding:10px;}
若上下填充一样，左右填充一样，则为：
div{padding：10px 20px;}
```

### 10、使用margin为盒子设置外边距（边界）

使用方法和注意项与padding一致。

## 八、CSS的布局模型

一共有三种布局：流动模型、浮动模型、层模型

### 1、流动模型

默认的网页布局模式

①**块状元素**都会在所处的**包含元素**内自上而下按顺序垂直延伸分布，在默认状态下，块状元素的宽度都为100%

②在流动模型下，**内联元素**都会在所处的包含元素内从左到右水平显示分布

### 2、浮动模型

实现块状元素的并排显示

```
div{
width:200px;
height:200px;
border:2px red solid;
float:left;
}
<div id="div1">栏目1</div>
<div id="div2">栏目2</div>
效果为在网页左方出现两个并排的方框；

若为float:right;
效果为在网页右方出现两个并排的方框；

div{
width:200px;
height:200px;
border:2px red solid;
}
#div1{float:left;}
#div2{float:right;}
<div id="div1">栏目1</div>
<div id="div2">栏目2</div>
效果为在网页的左右方个出现一个方框
```

### 3、层模型

三种形式：绝对定位、相对定位、固定定位

#### ①绝对定位

设置position:absolute（表示绝对定位）将元素从文档流中拖出来，再使用left、right、top、bottom属性相对于其最近的一个具有定位属性的父包含块进行绝对定位。如果不存在这样的包含快，则相对于body元素，即相对于浏览器。例如：

```
div{
width:200px;
height:200px;
border:2px red solid;
position:absolute;
left:100px;
top:50px;
}
<div id="div1"></div>
效果为板块距浏览器左边100px，距浏览器顶部50px；
```

#### ②相对定位

设置position:relative（表示相对定位），它通过left、right、top、bottom属性确定元素在**正常文档流中**的偏移位置。相对定位完成的过程是首先按static(float)方式生成一个元素(并且元素像层一样浮动了起来)，然后相对于**以前的位置移动，**移动的方向和幅度由left、right、top、bottom属性确定，偏移前的位置保留不动。

```
#div1{
width:200px;
height:200px;
border:3px red solid;
position:relative;
left:100px;
top:50px;
}
<div id="div1"></div><span>偏移前的位置还保留不动，覆盖不了前面的div没有偏移前的位置</span>

效果为：一个红色方框浮在“偏移前的位置还保留不动，覆盖不了前面的div没有偏移前的位置”上方
```

#### ③固定定位

设置position:fixed，它不会随浏览器窗口滚动条的滚动而变化，最典型的应用就是**网页中右下角的小广告**

```
#div1{
    width:200px;
    height:200px;
    border:2px red solid;
    position:fixed;
    bottom:0;
    right:0;
}
<div id="div1"></div>
```

#### ④Relative与Absolute的组合使用

①参照定位的元素必须是相对定位的元素的前辈元素，例如：

```
<div id="box1"><!--参照定位的元素-->
    <div id="box2">相对参照元素进行定位</div><!--相对定位元素-->
</div>

此代码中，box1是box2的父辈或者前辈元素
```

②参照定位的元素必须加入position:relative

```
#box1{
width:200px;
height:200px;
position:relative;
}
```

③定位元素加入position:absolute，便可以使用top,bottom,left,right来进行偏移定位

```
#box2{
position:absolute;
top:20px;
left:30px;
}
```

设置完成后，box2就是相对于父元素box1定位了（此时的参照物就不是浏览器本身了）

## 九、弹性盒模型

### 1、flex属性

即实现多个类似于div的块级元素的并行排列

①设置display:flex属性可以把块级元素在一排显示；

②flex属性必须添加在父元素上，从而改变；

③默认为从左到右排列，且和父元素之间没有空隙；

```
<style>
    .box{
        height:400px;
        background:skyblue;
        display:flex;
    }
    .box1{
        width:200px;
        height:200px;
        background:red;
    }
    .box2{
        width:200px;
        height:200px;
        background:orange;
    }
    .box3{
        width:200px;
        height:200px;
        background:green;
    }
</style>
<body>
    <div class="box">
        <div class="box1"></div>
        <div class="box2"></div>
        <div class="box3"></div>
    </div>
</body>
```

### 2、justify-content属性

属性值分别为：flex-start | flex-end | center | space-between | space-around

#### ①flex-start：交叉轴的起点对齐

```
.box{
background:blue;
display:flex;
jsutify-content:flex-start;
}
```

#### ②flex-end：右对齐

```
.box{
background:blue;
dsiplay:flex;
jsutify-content:flex-end;
}
```

#### ③center：居中

```
.box{
background:blue;
dsiplay:flex;
jsutify-content:center;
}
```

#### ④space-between：两端对齐，项目之间的间隔都相等

```
.box{
background:blue;
dsiplay:flex;
jsutify-content:space-between;
}
```

#### ⑤space-around：每个项目两侧的间隔相等，所以，项目之间的间隔比项目与边框的间隔大一倍

```
.box{
background:blue;
dsiplay:flex;
jsutify-content:space-around;
}
```

### 3、align-items属性

属性值分别为：flex-start | flex-end | center | baseline | stretch

#### ①flex-start：默认值，左对齐

```
.box{
height:700px;
background:blue;
dsiplay:flex;
align-items:flex-start;
}
```

#### ②flex-end：交叉轴的中点对齐

```
.box{
height:70px;
background:blue;
dsiplay:flex;
align-items:flex-end;
}
```

#### ③center：交叉轴的中点对齐

```
.box{
height:70px
background:blue;
dsiplay:flex;
align-items:center;
}
```

#### ④baseline：项目的第一行文字的基线（文字的下沿线）对齐

```
.box{
height:70px
background:blue;
dsiplay:flex;
align-items:baseline;
}
```

#### ⑤stretch：如果项目未设置高度或者设为auto，将占满整个容其高度

```
.box {
height: 300px;
background: blue;
display: flex;
align-items: stretch;
}
.box div {
/*不设置高度，元素在垂直方向上铺满父容器*/
width: 200px;
}  
```

### 4、给子元素设置flex占比

#### ①给子元素设置flex属性，可以设置子元素相对于父元素的占比；

#### ②flex属性的值只能是正整数，表示占比多少；

#### ③给子元素设置了flex后，其宽度属性会失效；

```
.box {
height: 300px;
background: blue;
display: flex;
}
.box div {
width: 200px;
height: 200px;
}
.box1 {
flex: 1;
background: red;
}
.box2 {
flex: 3;
background: orange;
}
.box3 {
flex: 2;
background: green;
}
```

### 十、水平居中设置

#### ①行内元素：设置文本、图片等行内元素，水平居中是通过给父元素设置text-align:center来实现，例如：

```
<style>
    .txtCenter{
        text-align:center;
    }
</style>
<body>
    <div class="txtCenter">我想要在父元素中水平居中显示</div>
</body>
```

#### ②块状元素：分为两种—-定宽块状元素和不定宽块状元素

**定宽块状元素**：块状元素的宽度width为固定值；满足定宽和块状的两个条件的元素是可以通过设置“左右margin”值为“auto”来实现居中

```
<style>
div{
border:1px solid red;/*为了显示居中效果明显为 div 设置了边框*/
width:200px;/*定宽*/
margin:20px auto;/* margin-left 与 margin-right 设置为 auto */
}
</style>
<body>
  <div>我是定宽块状元素，水平居中显示。</div>
</body>
```

注意：定宽与块状两个条件缺一不可