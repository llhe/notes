哈希函数碰撞与数字签名伪造 
==========================

哈希函数
-----------
用于加密或者数字签名的哈希函数需要满足三个特性：  
  1. 对于任意x，找到 f(y) = x 很困难 -- 破解密码  
  2. 对于任意x，找到y使得 f(y) = f(x) 很困难 -- ？？即使找到y也不是正常字符串  
  3. 找出任意一对x，y，使得f(x) = f(y) -- 前缀攻击

对于一般用于安全的哈娴熟，1，2几乎不能破解，几乎所有研究都是针对3，降低搜索空间使得当前硬件计算成为可能。

针对3的攻击称为Chosen-prefix collision attack，且已有真实案例。
其中MD5已经在1996年被找到，SHA1一样，所以MD5和SHA1做数字签名算法是不安全的。

Chosen-prefix collision attack
-------------
简单原理如下，对于支持高级内容的文件(ps, word?, pdf?，甚至说脚本语言)及其viewer(解释器)，如果支持条件语句，则可能实施此类攻击，伪造文件担保正签名不变仍然有效。
假设对于签名的哈希函数h找到一组数据:  
	h(str1) = h(str2)

拿去签名的文件:

	s = str1;
	if (s == str1) {
	  display_or_exec(good);
	} else {
	  display_or_exec(evil);
	}

伪造的文件，签名跟原文件相同:

	s = str2;
	if (s == str1) {
	  display_or_exec(good);
	} else {
	  display_or_exec(evil);
	}

当然实际str1, str2会有不同非法字符等问题，此处仅为示例。

case study
-------------
以色列间谍木马伪造微软 MD5签名:
[Flame](http://en.wikipedia.org/wiki/Flame_(malware\))
> In 2005, researchers were able to create pairs of PostScript documents and X.509 certificates with the same hash. Later that year, MD5's designer Ron Rivest wrote, "md5 and sha1 are both clearly broken (in terms of collision-resistance)."
> [more n wiki](http://en.wikipedia.org/wiki/Chosen-prefix_collision_attack#Chosen_prefix_collision_attack)

参考文献：  
http://en.wikipedia.org/wiki/MD5  
http://www.cits.rub.de/imperia/md/content/magnus/rump_ec05.pdf  
http://zhiqiang.org/blog/science/computer-science/preliminary-computer-theory-xiao-yun-wang-from-the-hash-function-to-crack-md5.html  
http://www.zhihu.com/question/19765456  
