CreateTime:2021-01-21 18:22:08.0

# 需求

首先说下需求。

最近一个朋友，遇到一个大数据处理，需要大量节约字符串空间，给我提了一个需求。大概是如此：

给定类似如下字符串，是一个由浮点数组成的字符串数字

	"499.00 499.00 490.00 490.00 47345"

要求的结果是什么呢？能生成一个压缩后的结果，尽可能减少存储空间，也就是字符长度。

也就是传说中的：

	怎么实现我不管，我只要它能变短

# 解决思路

emmm，一开始，根本没理解需求，和他撕了半天之后，才理解。

于是我死来想去，想出3种解决方案。

1. zlib glib库
2. map存储字符出现次数和位置
3. 不告诉你，看下文吧。

### zlib glib库

这个实现方案很简单，go提供了相应的包。
只需要调用api就行了

```
b := []byte(`499.00 499.00 490.00 490.00 47345`)
w := zlib.NewWriter(&in)
w.Write(b)
w.Close()
fmt.Println(len(b))
fmt.Println(len(in.Bytes()))

 var out bytes.Buffer
 r, _ := zlib.NewReader(&in)
 io.Copy(&out, r)
 fmt.Println(out.String())
```
但是运行结果，不是很明显，33个字符长度只压缩到了27。

再仔细看，NewWriter可以设置压缩比例（NewWriterLevel）。我设置为9（最好压缩flate.BestCompression）,对acsii中的字符并没有更多的优化。

无奈，只好另寻方案。

### map存储字符出现次数和位置

此方案就是遍历字符串，然后存储响应位置。
方案代码就不贴了，实际效果很差。

### 位运算合并存储

看需求分析，我们得到的是一个字符串为"float float float float int",所有字符由'0~9','.',' '组成。

嗯？ 有漏洞。

一个字节8bit（11111111），那么，1111 => 最多可以表示16种字符，完全可以使用位替代。高位一个字符，低位一个字符。

说做咱就做啊。

比如：

```
49=> 00000101 00001010
合并为一个
01011010
```

特殊字符怎么办？我们不是只用了0~9嘛，用剩下的， 比如10 替换成空格， 11替换成.。

高低位置换？这个比较简单：

	x:= value << 4 //高位使用value
	x = value | (x)  //低位使用value

还有一个细节，当我们整个字符串为奇数位时，要进行特殊操作。高位为value,低位要补充一个字符标识结束了。我这里用的12，比较随意

```
if i == l-1 {
	if supply {
		value = x << 4
		value = value | 12
		res[i/2] = value
		break
	}
}
```


具体实现如下：
```
func CompressFloatString(str string) []byte {
	l := len(str)
	supply := false
	var res []byte
	if l%2 != 0 {
		supply = true
		res = make([]byte, l/2+1)
	} else {
		res = make([]byte, l/2)
	}

	var value uint8
	for i, v := range str {
		v8 := uint8(v)
		x := NumberConvertNormal(v8)
		if i == l-1 {
			if supply {
				value = x << 4
				value = value | 12
				res[i/2] = value
				break
			}
		}

		if i%2 == 0 {
			value = x << 4
		} else {
			value = value | (x)
			res[i/2] = value
		}
	}
	return res
}

func NumberConvertNormal(v8 uint8) uint8 {

	var x uint8
	switch {
	// 0-9
	case v8 > 47 && v8 < 58:
		return v8 - 48
	case v8 == 46: //为小数点
		return 11
	case v8 == 32: //为空格
		return 10
	}
	return x
}
```

解码的过程一样，反过来就行。

```
func NumberConvertSmall(n uint8) uint8 {

	switch {
	case n < 10:
		return n + 48
	case n == 11:
		return 46
	case n == 10:
		return 32
	case n == 12:
		return 0
	}
	return 0
}

func UnCompressFloatString(bt []byte) []byte {
	var res []byte
	for _, v := range bt {
		res = append(res, NumberConvertSmall(v>>4))
		vNum := NumberConvertSmall(v & 15)
		if vNum == 0 {
			break
		}
		res = append(res, vNum)
	}
	return res
}
```

实现结果为：

```
原始串字符 499.00 499.00 490.00 490.00 47345
原始串bytes [52 57 57 46 48 48 32 52 57 57 46 48 48 32 52 57 48 46 48 48 32 52 57 48 46 48 48 32 52 55 51 52 53]
原始串bt len 33
压缩串bytes [73 155 0 164 153 176 10 73 11 0 164 144 176 10 71 52 92]
压缩串字符 I� ���
I ���
G4\
压缩串bt len 17
还原 499.00 499.00 490.00 490.00 47345
```

# 总结

在适当的时候使用位运算，能起到极好的作用。

很多通信协议，都是使用位来控制协议内容以及编解码。