title: jQuery 中 attr() 和 prop() 方法
author: cwen
date:  2016-03-09 
update:  2016-03-09
tags:
    - 编程 
    - 前端  
    - JQuery 
    - JavaScript  

--- 

昨天在使用JQuery实现一个一键全选的功能的时候，在设置 `checkbox` 属性后，只是在第一有效，过后的N次一直无法改变。一开始还以为自己代码是不是神马地方的逻辑出了问题，可是在检查的多次之后依然无法找到答案，最后只能求助Google(按常理来说一般先翻文档),先上我之前出问题的代码。<!--more--> 

```  
(function() {
    var dom = {
        dbTransferAll : $('#dbTransferAll'),
        dbCollections : $("form div:first input[name='collection']")
    }

    var dbTransfer = {
        init : function() {
            this.eventFn();
        },

        eventFn : function() {
            dom.dbTransferAll.bind('click',function() {
                
                if (this.checked == true) {
                    dom.dbCollections.attr("checked",true);
                } else {
                    dom.dbCollections.attr("checked", false); 
                }
                
            });
        }
    }
    dbTransfer.init();
})();
```   
最后通过诸位大神的博客得知，是使用了 `attr()` 方法的问题， 在JQuery在1.6版本之后新增了一个 `prop()` 方法(羞愧啊！一直不知道),应该使用 `prop()` 方法替换 `attr()` 。 

#### 为什么要新加 `prop()` 方法   

jQuery 1.6之前 ，`attr()` 方法在取某些 `attribute` 的值时，会返回 `property` 的值，这就导致了结果的不一致。从 jQuery 1.6 开始， `prop()` 方法 方法返回 `property` 的值,而 `attr()` 方法返回 `attributes` 的值。  

#### attribute和property的区别  

`attribute` 翻译成中文术语为“特性”，`property` 翻译成中文术语为“属性”，从中文的字面意思来看，确实是有点区别了，先来说说`attribute` 。  
attribute是一个特性节点，每个DOM元素都有一个对应的 `attributes` 属性来存放所有的 `attribute` 节点，`attributes` 是一个类数组的容器，说得准确点就是 `NameNodeMap`，总之就是一个类似数组但又和数组不太一样的容器。`attributes` 的每个数字索引以名值对 `(name=”value”)` 的形式存放了一个 `attribute` 节点。  
```
<div class="box" id="box" gameid="880">hello</div>
```   
 上面的div元素的HTML代码中有class、id还有自定义的gameid，这些特性都存放在attributes中，类似下面的形式：
view sourceprint?   
```  
[ class="box", id="box", gameid="880" ]
```   
`property` 就是一个属性，如果把DOM元素看成是一个普通的 `Object` 对象，那么 `property` 就是一个以名值对`(name=”value”)` 的形式存放在 `Object` 中的属性。要添加和删除 `property` 也简单多了，和普通的对象没啥分别。   
之所以 `attribute` 和 `property` 容易混倄在一起的原因是，很多 `attribute` 节点还有一个相对应的 `property` 属性，比如上面的 `div` 元素的 `id` 和 `class` 既是 `attribute` ，也有对应的 `property` ，不管使用哪种方法都可以访问和修改。  

DOM元素一些默认常见的 `attribute` 节点都有与之对应的 `property` 属性，比较特殊的是一些值为 `Boolean` 类型的`property`(这个地方`attr()`与`prop()`不同点)，如一些表单元素：  
```  
<input type="radio" checked="checked" id="raido">
var radio = document.getElementById( 'radio' );
console.log( radio.getAttribute('checked') ); // checked
console.log( radio.checked ); // true
``` 
对于这些特殊的`attribute`节点，只有存在该节点，对应的`property` 的值就为true，如：  
```
<input type="radio" checked="anything" id="raido">
var radio = document.getElementById( 'radio' );
console.log( radio.getAttribute('checked') ); // anything
console.log( radio.checked ); // true
```
最后为了更好的区分`attribute`和`property`，基本可以总结为`attribute`节点都是在HTML代码中可见的，而`property`只是一个普通的名值对属性。
```  
// gameid和id都是attribute节点
// id同时又可以通过property来访问和修改
<div gameid="880" id="box">hello</div>
// areaid仅仅是property
elem.areaid = 900;
```   

#### 神马时候使用 `attr()` ? 神马时候使用 `prop()` ?   
根据官方的建议：具有 `true` 和 `false` 两个属性的属性，如 `checked`, `selected` 或者 `disabled` 使用`prop()`，其他的使用 `attr()` 。   

> 文章部分参考 [attribute和property的区别](http://stylechen.com/attribute-property.html)  
