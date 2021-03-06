---
title: 这 10 行比较字符串相等的代码给我整懵逼了，不信你也来看看
layout: post
categories: 
  - MyLife 
tags: 
  - 经验技巧
  - 网络安全
---

抱歉用这种标题吸引你点进来了，不过你不妨看完，看看能否让你有所收获。​（有收获，请评论区留个言，没收获，下周末我直播吃**，哈哈，这你也信）

>补充说明：微信公众号改版，对各个号主影响还挺大的。目前从后台数据来看，对我影响不大，因为我这反正都是小号，😂阅读量本身就少的可怜，真相了，🐶狗头（刚从交流群学会的表情）。

先直接上代码： 

```java
boolean safeEqual(String a, String b) {
   if (a.length() != b.length()) {
       return false;
   }
   int equal = 0;
   for (int i = 0; i < a.length(); i++) {
       equal |= a.charAt(i) ^ b.charAt(i);
   }
   return equal == 0;
}
```

上面的代码是我根据原版（`Scala`）翻译成 `Java`的，`Scala` 版本（最开始吸引程序猿石头注意力的代码）如下：

```scala
def safeEqual(a: String, b: String) = {
  if (a.length != b.length) {
    false
  } else {
    var equal = 0
    for (i <- Array.range(0, a.length)) {
      equal |= a(i) ^ b(i)
    }
    equal == 0
  }
}
```

刚开始看到这段源码感觉挺奇怪的，这个函数的功能是比较两个字符串是否相等，首先“长度不等结果肯定不等，立即返回”这个很好理解。

再看看后面的，稍微动下脑筋，转弯下也能明白这其中的门道：通过异或操作`1^1=0, 1^0=1, 0^0=0`，来比较每一位，如果每一位都相等的话，两个字符串肯定相等，最后存储累计异或值的变量`equal`必定为 0，否则为 1。

## 再细想一下呢？

```scala
for (i <- Array.range(0, a.length)) {
  if (a(i) ^ b(i) != 0) // or a(i) != b[i]
    return false
}
```

我们常常讲性能优化，从效率角度上讲，难道不是应该只要中途发现某一位的结果不同了（即为1）就可以立即返回两个字符串不相等了吗？(如上所示) 

这其中肯定有……

![](https://imgkr.cn-bj.ufileos.com/0d90724a-9e67-42ff-9db5-53453fba1abf.png)


## 再再细想一下呢？
 
结合方法名称 `safeEquals` 可能知道些眉目，与安全有关。

>本文开篇的代码来自playframewok 里用来验证cookie(session)中的数据是否合法(包含签名的验证)，也是石头写这篇文章的由来。

以前知道通过延迟计算等手段来提高效率的手段，但这种已经算出结果却延迟返回的，还是头一回！

我们来看看，JDK 中也有类似的方法，如下代码摘自 `java.security.MessageDigest`：

```java
public static boolean isEqual(byte[] digesta, byte[] digestb) {
   if (digesta == digestb) return true;
   if (digesta == null || digestb == null) {
       return false;
   }
   if (digesta.length != digestb.length) {
       return false;
   }

   int result = 0;
   // time-constant comparison
   for (int i = 0; i < digesta.length; i++) {
       result |= digesta[i] ^ digestb[i];
   }
   return result == 0;
}
```

看注释知道了，目的是为了用常量时间复杂度进行比较。

但这个计算过程耗费的时间不是常量有啥风险？ （脑海里响起了背景音乐：“小朋友，你是否有很多问号？”）

![](https://imgkr.cn-bj.ufileos.com/327a7a80-6fd7-4b4f-b836-78fc9399ce34.png)


## 真相大白

再深入探索和了解了一下，原来这么做是为了防止**计时攻击**（Timing Attack）。（也有人翻译成时序攻击​）​

![](https://imgkr.cn-bj.ufileos.com/43ae6be8-c869-4bfa-bab9-77435893afb4.png)


## 计时攻击(Timing Attack)

计时攻击是边信道攻击(或称"侧信道攻击"， Side Channel Attack， 简称SCA) 的一种，边信道攻击是一种针对软件或硬件设计缺陷，走“歪门邪道”的一种攻击方式。

这种攻击方式是通过功耗、时序、电磁泄漏等方式达到破解目的。在很多物理隔绝的环境中，往往也能出奇制胜，这类新型攻击的有效性**远高于**传统的密码分析的数学方法（某百科上说的）。

这种手段可以让调用 `safeEquals("abcdefghijklmn", "xbcdefghijklmn")` （只有首位不相同）和调用 `safeEquals("abcdefghijklmn", "abcdefghijklmn")` （两个完全相同的字符串）的所耗费的时间一样。防止通过大量的改变输入并通过统计运行时间来暴力破解出要比较的字符串。 

举个🌰，如果用之前说的“高效”的方式来实现的话。假设某个用户设置了密码为 `password`，通过从a到z（实际范围可能更广）不断枚举第一位，最终统计发现 `p0000000` 的运行时间比其他从任意`a~z`的都长（因为要到第二位才能发现不同，其他非 `p` 开头的字符串第一位不同就直接返回了），这样就能猜测出用户密码的第一位很可能是`p`了，然后再不断一位一位迭代下去最终破解出用户的密码。

当然，以上是从理论角度分析，确实容易理解。但实际上好像通过统计运行时间总感觉不太靠谱，这个运行时间对环境太敏感了，比如网络，内存，CPU负载等等都会影响。

但安全问题感觉更像是 “宁可信其有，不可信其无”。为了防止(特别是与签名/密码验证等相关的操作)被 **timing attack**，目前各大语言都提供了相应的安全比较函数。各种软件系统（例如 OpenSSL）、框架（例如 Play）的实现也都采用了这种方式。 

例如 “世界上最好的编程语言”（粉丝较少，评论区应该打不起架来）—— php中的: 

```php
// Compares two strings using the same time whether they're equal or not.
// This function should be used to mitigate timing attacks; 
// for instance, when testing crypt() password hashes.
bool hash_equals ( string $known_string , string $user_string )

//This function is safe against timing attacks.
boolean password_verify ( string $password , string $hash )
```

其实各种语言版本的实现方式都与上面的版本差不多，将两个字符串每一位取出来异或(`^`)并用或(`|`)保存，最后通过判断结果是否为 0 来确定两个字符串是否相等。 

如果刚开始没有用 `safeEquals` 去实现，后续的版本还会通过打补丁的方式去修复这样的安全隐患。  

例如 [JDK 1.6.0_17 中的Release Notes](http://www.oracle.com/technetwork/java/javase/6u17-141447.html "Release Notes")中就提到了`MessageDigest.isEqual` 中的bug的修复，如下图所示：

![MessageDigest timing attack vulnerabilities](https://static01.imgkr.com/temp/7e264b6f9f6141e9a779f07bd29ebd64.png)

大家可以看看这次变更的的详细信息[openjdk中的 bug fix diff](http://hg.openjdk.java.net/jdk6/jdk6/jdk/rev/562da0baf70b "openjdk中的 bug fix diff")为：

![MessageDigest.isEqual计时攻击](https://static01.imgkr.com/temp/87adf41576964e8bafb2abf48c6a95ab.png)


## Timing Attack 真的可行吗？

我觉得各大语言的 API 都用这种实现，肯定还是有道理的，理论上应该可以被利用的。 
这不，学术界的这篇论文就宣称用这种计时攻击的方法破解了 `OpenSSL 0.9.7` 的RSA加密算法了。关于 [RSA 算法的介绍](http://mp.weixin.qq.com/s?__biz=MzI3OTUzMzcwNw==&mid=100000109&idx=1&sn=e65143adf4f81cc5c559b03c58d029e9&chksm=6b4700895c30899f6148025c6120844bd0aeb2d09413ad4ece33f2ea49296fa6f54d17447f63#rd)可以看看之前本人写的这篇文章。 

这篇[Remote Timing Attacks are Practical](http://crypto.stanford.edu/~dabo/papers/ssl-timing.pdf "Remote Timing Attacks are Practical") 论文中指出（我大致翻译下摘要，感兴趣的同学可以通过文末链接去看原文）：

计时攻击往往用于攻击一些性能较弱的计算设备，例如一些智能卡。我们通过实验发现，也能用于攻击普通的软件系统。本文通过实验证明，通过这种计时攻击方式能够攻破一个基于 OpenSSL 的 web 服务器的私钥。结果证明计时攻击用于进行网络攻击在实践中可行的，因此各大安全系统需要抵御这种风险。

最后，本人毕竟不是专研完全方向，以上描述是基于本人的理解，如果有不对的地方，还请大家留言指出来。感谢。

>补充说明2：感谢正在阅读文章的你，让我还有动力继续坚持更新原创。
>
>本人发文不多，但希望写的文章能达到的目的是：占用你的阅读时间，就尽量能够让你有所收获。
>
>如果你觉得我的文章有所帮助，还请你帮忙转发分享，另外请别忘了点击公众号右上角加个星标，好让你别错过后续的精彩文章（微信改版了，或许我发的文章都不能推送到你那了）。

​原创真心不易，希望你能帮我个小忙呗，如果本文内容你觉得有所启发，有所收获，请帮忙点个“在看”呗，或者转发分享让更多的小伙伴看到。 
​
参考资料：

- [Timing Attacks on RSA: Revealing Your Secrets through the Fourth Dimension](http://www.cs.sjsu.edu/faculty/stamp/students/article.html)
- [Remote Timing Attacks are Practical](http://crypto.stanford.edu/~dabo/papers/ssl-timing.pdf) 

