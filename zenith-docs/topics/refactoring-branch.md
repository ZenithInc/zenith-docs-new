# 分支语句的重构

在《代码里的世界观-通往架构师之路》一书的前言中，有这么一句话：我认为编程中组织代码的能力，说白了就是将各种 API 串起来的能力，无论是针对新鲜的、高大上的 API，还是已经淘汰的 API。

作者说又说到：我不断追逐技术潮流，但更感兴趣的始终不是新框架带来了哪些新概念，而是背后那些最朴实、最基本的代码结构的本质，是那些最通用的编程技巧。

是的，我也一直在写代码。如今的代码都是一些“面向程序员”的代码，你并不需要关注细节，你只要知道如何调用已有的 API 就好了。这也给我造成了很多的困惑，很多的问题都源于我对技术原理、本质的不理解。我试图通过对代码向剥洋葱一样的解剖，来还愿其本来面貌。就如同上面提到的这本书的作者所言，组织代码就是将 API 进行串联然后运行。**运行有几种形式，顺序结构、分支结构以及循环结构。这些结构加上对 API 的使用，就构成了代码的本质**。

代码的本质是不是很简单？**简单的事物是很容易上手的，但是不一定是容易熟练掌握的，这中间缺少的(编程的)技巧和经验**。比如分支结构，常见的有`if...else...`以及`switch`语法。这些语法都十分容易理解，但是想用的好，是需要技巧和经验的。

所以，这一篇文档就来描述分支结构的重构。

## 使用三元运算符 {id="ternary-operator"}

在实际的开发中，一些简单的分支逻辑，我们会使用三元运算符来简化:

```php
if ( $a < $b ) {
	$min = $a;
} else {
	$min = $b;
}
```

上面的代码可以转换为`$min = (a < b) ? a : b`这样的形式，更为简洁。

## 使用 switch 语句 {id="switch"}

在一些对个别值进行判断的常见中，我们应该优先使用 `switch...case...` 语句，而不是 `if...else...`语句。

在使用`switch...case...`的时候，要注意`case`中，最好不用书写大量的逻辑代码，将代码尽可能都分装成方法，然后在 `case`中调用方法。

另外，所有的 `case` 的后面最好都加上`default`语句，即使什么也不写。

在 Python 中，并不存在 Switch 语句，可以使用其他方法代码，而且更加优雅。将在后面中，介绍。

## 避免使用过多层次的分支 {id="avoid-excessive-nesting-of-branches"}

看如下这个例子，就不恰当的使用了多层次的分支判断，导致了代码可读性的下降。

```php
if($a < $b) {
	if($a < $c) {
		return $a;
	}
}
```

这样的多层判断，我们就可以简写为一层判断:

```php
if ($a < $b && $a < $c) {
	return $a
}
```

## 使用逻辑运算符简化分支 {id="simplify-branches-with-logical-operators"}

依然采用上文中的例子，有如何分支:

```php
if ($a < $b) {
	$min = $a;
}
```

这段代码我们可以简写为如下形式:

```php
($a < $b) && $min = $a;
```

同样的效果，但是代码更为简洁。

## 使用数组来简化分支 {id="simplify-branches-with-array"}

下面这个例子是我在《代码大全（第二版》中看到的，那时候我刚入行不久，看到之后异常兴奋，原来代码还可以这样写，这是一个转换星期数字和显示值的例子:

```php
if ($week === 0) {
	echo '星期日';
} else if ($week === 1) {
	echo '星期一';
} else if ($week === 2) {
	echo '星期二';
} else if ($week === 3) {
	echo '星期三';
} else if ($week === 4) {
	echo '星期四';
} else if ($week === 5) {
	echo '星期五';
} else if ($week === 6) {
	echo '星期六';
}
```

这段代码看起来非常冗长，如果我们重构成如下结构，运行结果完全一致，并且代码更加简洁:

```php
$arr = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六'];
echo $arr[$week];
```

从表面上看，我们通过将值写入数组，并利用李数组的下标关联了 `$week` 的值，重构了这段代码。但是这段代码的意义不仅仅于此，更在于数据和逻辑的分离。

## 数据和逻辑分离 {id="data-and-logic-separation"}

通过上面的例子，我们知道了可以利用数组来简化分支的书写。但是我们又提到，这样改写的意义是在于数据和逻辑的分离。接下来，举一个实际的例子来说明。

在实际的开发中，我们数据库存储的值和应用中实际显示的值往往是不一样的。比如说数据库中存储的订单状态，出于节省磁盘空间和查询性能优化的目的，我们使用的是数值，而实际在页面中显示的是字符串。所以我们经常在代码中，使用分支来转换数值和字符串。这时候，我就会利用数组的形式简化分支，示例代码如下:

```php
public function getStatusText(int $status): string {
	$statusMap = ['创建', '支付', '发货', '收获', '评价', '完结'];
	return $statusMap[$status];
}
```

这样的写法简化了分支是自然的，更重要的是将数据和逻辑分离了。怎么说？在这个方法中，逻辑是不变的，要将数据库中存储的数值转换成用户看到的字符串，对应的代码是`return $statusMap[$status]`。实际的开发中，变化的是数据,也会用不了多久，这个购物流程中新增退款退货流程，需要新增一些状态，比如带退款中、退款退货中、退款完成、拒绝退款等等。当发生变化的时候，我们并不需要去更改逻辑，只需要在数组中，新增多个元素即可。这样的代码的可扩展性就很高。

综上所述，**使用分支结构是将代码逻辑和数据进行了耦合，当数据发生了变化，那么逻辑也会随之变化。而将数据和逻辑分离，会使得代码的可扩展性更高**。

> 在实际开发中，数据不一定是存放在数组中的，也可能存放在数据库中、配置文件中。我们并不关心数据最终的存储位置，更关心的是数据和逻辑的分离形式。


## 总结 {id="summary"}

关于分支的重构，一方面在于简化分支的编写，另一方面在于数据和逻辑的分离。即将代码中变化的内容提取出来，是的逻辑可以不随着数据的更迭而变化。我们可以总结出这样一个道理，代码的复杂在于需求的不断变化，让代码更容易应对变化是我们面临的最重要的课题，即如何提升代码的可扩展性。

通过上面的例子，我们可以总结一句话，代码是数据和逻辑的结合。这一点在《代码的世界观-通往架构师之路》一书中有更详细的解释:

> 数据和代码是组成程序的两个基本元素。数据是目的，代码是手段。一定要明白代码是为数据服务的，数据才是整个系统的中心。要时刻提醒自己:归根结底，面向用户的是数据。


关于重构，这本书中如是说:

> 如果重构一个系统，抓不住头绪，不妨从数据的角度进行重新梳理和思考。这样抓住源头往往能拨开迷雾，站在更高的角度去理解这个系统，从而生成最佳的想法。