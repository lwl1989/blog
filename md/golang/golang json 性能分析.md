CreateTime:2021-06-11 17:56:12.0

> 在看代码的过程中，发现很多代码并没有使用golang自带的json，而是使用json-iterator去做的json的编解码，好奇之下，便去研究了一下。

# json

Json 作为一种重要的数据格式，具有良好的可读性以及自描述性，广泛地应用在各种数据传输场景中。Go 语言里面原生支持了这种数据格式的序列化以及反序列化，内部使用反射机制实现，性能有点差，在高度依赖 json 解析的应用里，往往会成为性能瓶颈。

那我们今天的主角是GO的json，主要分析2个包，就是原生的encode/json和json-iterator。

## 测试

作为性能而言，我自然要先做压测，go自带了benchmark可以很方便我们对代码进行数据测试。

简单的结构没有测试意义，因此取了一个接口的返回值当测试数据。

```
package test

import (
	"encoding/json"
	json1 "github.com/json-iterator/go"
	"testing"
)


var str = `{
    "stat": 1,
    "code": 0,
    "msg": "成功",
    "data": {
        "courseInfos": [
            {
                "courseId": "161935",
                "course_name": "高三5科语数英物化提分特训班（20课时）",
                "gradeId": "13",
                "gradeName": "高三",
                "subjectName": "数学",
                "difficultyName": "目标A+",
                "type1Name": "特训班",
                "termIds": "4",
                "schoolTime": "1月29日-1月31日上课（详情见大纲）",
                "price": "799",
                "actualPrice": 20,
                "subjectId": 2,
                "type_1_id": "2064",
                "type_2_id": "2656",
                "type_3_id": "2661"
            }
        ],
        "ext": {
            "price": "799",
            "sale": "20",
            "wxShareObj": "{\"title\":\"语数双科提分特训班\",\"desc\":\"20元抢20课时名师直播好课，下单加送国风限量教辅礼包！\",\"imgUrl\":\"https://activity.xueersi.com/topic/growth/common/images/common/xes-logo.png\",\"miniImgUrl\":\"https://hw.xesimg.com/biz-growth-storage/operations/groupon/20201215/0f4f57c8c6f88b66b059881cfb050527.png\"}",
            "abTestPackage": "{\"h5\":[\"20_20ChineseA_sucaiH5\",\"Azhifudanye_H5\",\"Apintuan_H5\",\"Axueyuanpinglun_ceshi\"],\"smallProgram\":[]}"
        },
        "resourceConfig": {
            "bookImg": [
                {
                    "type": "img",
                    "url": "https://ek.xesimg.com/biz-growth-storage/activity/upload/20201216/86724030cc04b4ea029fe31e4042880c.png",
                    "name": "初高H5&amp;小程序随材图 .png"
                }
            ],
            "detailImg": [
                {
                    "type": "img",
                    "url": "https://oo.xesimg.com/biz-growth-storage/activity/upload/20210126/7e9bd0cb1810c9de1432bc8c570ce3f0.png",
                    "name": "1@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://ek.xesimg.com/biz-growth-storage/activity/upload/20210126/17bdceacf83feda19ee854b50c469152.png",
                    "name": "2@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://hw.xesimg.com/biz-growth-storage/activity/upload/20210126/21189d1f915feeaf0f103a5dfbe77950.png",
                    "name": "3@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://mr.xesimg.com/biz-growth-storage/activity/upload/20210126/fb106f25b740fc7e559bab5568958fe5.png",
                    "name": "4@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://oo.xesimg.com/biz-growth-storage/activity/upload/20210126/93f595fa4cd4fd72683f2faf1b33e86b.png",
                    "name": "5@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://oo.xesimg.com/biz-growth-storage/activity/upload/20210126/b5d789b9aa8833367d4d74af5d20bfc5.png",
                    "name": "7@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://oo.xesimg.com/biz-growth-storage/activity/upload/20210126/389308e953ac695a972840c796443de5.png",
                    "name": "6@2x.png"
                },
                {
                    "type": "img",
                    "url": "https://oo.xesimg.com/biz-growth-storage/activity/upload/20210126/29465ee6a664f36c4ae4b154f60c5b01.png",
                    "name": "矩阵@2x.png"
                }
            ],
            "headImg": [
                {
                    "type": "img",
                    "url": "https://ek.xesimg.com/biz-growth-storage/activity/upload/20210126/45e73133144cb179a1f33d85e5e02c68.png",
                    "name": "头图@2x.png"
                }
            ],
            "bookTextDesc": "多科目组合课程，为保障学习效果，暂不支持调课哦",
            "bookTextWx": "https://ek.xesimg.com/biz-growth-storage/operations/groupon/20201216/cda7dc272b4757efa7a21c3de30f87e8.png",
            "grade_id": "13",
            "feitoufang": "140012,140013,140014",
            "videoInfo": "{\"videoUrl\":\"https://activity.xueersi.com/oss/resource/%E9%AB%98%E4%B8%AD-1611580251682.mp4\",\"videoPoster\":\"https://activity.xueersi.com/oss/resource/%E9%AB%98%E4%B8%AD-1611658207877.png\"}"
        }
    }
}`

type T struct {
	Stat int    `json:"stat"`
	Code int    `json:"code"`
	Msg  string `json:"msg"`
	Data struct {
		CourseInfos []struct {
			CourseId       string `json:"courseId"`
			CourseName     string `json:"course_name"`
			GradeId        string `json:"gradeId"`
			GradeName      string `json:"gradeName"`
			SubjectName    string `json:"subjectName"`
			DifficultyName string `json:"difficultyName"`
			Type1Name      string `json:"type1Name"`
			TermIds        string `json:"termIds"`
			SchoolTime     string `json:"schoolTime"`
			Price          string `json:"price"`
			ActualPrice    int    `json:"actualPrice"`
			SubjectId      int    `json:"subjectId"`
			Type1Id        string `json:"type_1_id"`
			Type2Id        string `json:"type_2_id"`
			Type3Id        string `json:"type_3_id"`
		} `json:"courseInfos"`
		Ext struct {
			Price         string `json:"price"`
			Sale          string `json:"sale"`
			WxShareObj    string `json:"wxShareObj"`
			AbTestPackage string `json:"abTestPackage"`
		} `json:"ext"`
		ResourceConfig struct {
			BookImg []struct {
				Type string `json:"type"`
				Url  string `json:"url"`
				Name string `json:"name"`
			} `json:"bookImg"`
			DetailImg []struct {
				Type string `json:"type"`
				Url  string `json:"url"`
				Name string `json:"name"`
			} `json:"detailImg"`
			HeadImg []struct {
				Type string `json:"type"`
				Url  string `json:"url"`
				Name string `json:"name"`
			} `json:"headImg"`
			BookTextDesc string `json:"bookTextDesc"`
			BookTextWx   string `json:"bookTextWx"`
			GradeId      string `json:"grade_id"`
			Feitoufang   string `json:"feitoufang"`
			VideoInfo    string `json:"videoInfo"`
		} `json:"resourceConfig"`
	} `json:"data"`
}
//因为原生也不进行任何参数设置，所以都使用默认配置
var Json1 = json1.ConfigDefault
var T1 T
var Tmap map[string]interface{}
var TByte []byte
func init() {
	TByte = []byte(str)
	json1.Unmarshal(TByte, &T1)
	json1.Unmarshal(TByte, &Tmap)
}

func BenchmarkEncodeStructJson1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Json1.Marshal(T1)
	}
}

func BenchmarkEncodeStructJson(b *testing.B) {
	for i := 0; i < b.N; i++ { 
		json.Marshal(T1)
	}
}

func BenchmarkEncodeMapJson1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Json1.Marshal(Tmap)
	}
}

func BenchmarkEncodeMapJson(b *testing.B) {
	for i := 0; i < b.N; i++ {
		json.Marshal(Tmap)
	}
}

func BenchmarkDecodeStructJson1(b *testing.B) {
	var TStruct T
	for i := 0; i < b.N; i++ {
		Json1.Unmarshal(TByte, &TStruct)
	}
}

func BenchmarkDecodeStructJson(b *testing.B) {
	var TStruct T
	for i := 0; i < b.N; i++ {
		json.Unmarshal(TByte, &TStruct)
	}
}

func BenchmarkDecodeMapJson1(b *testing.B) {
	var TTMap map[string]interface{}
	for i := 0; i < b.N; i++ {
		Json1.Unmarshal(TByte, &TTMap)
	}
}

func BenchmarkDecodeMapJson(b *testing.B) {
	var TTMap map[string]interface{}
	for i := 0; i < b.N; i++ {
		json.Unmarshal(TByte, &TTMap)
	}
}
```

代码准备好之后，就直接开始测试吧

	go test -bench=. -benchtime=3s -benchmem -run=none

测试结果

	goos: darwin
	goarch: amd64
	pkg: test
	cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
	BenchmarkEncodeStructJson1-12             450108              6911 ns/op            3365 B/op          2 allocs/op
	BenchmarkEncodeStructJson-12              538249              6508 ns/op            3365 B/op          2 allocs/op
	BenchmarkEncodeMapJson1-12                281665             11958 ns/op            4634 B/op         31 allocs/op
	BenchmarkEncodeMapJson-12                 123217             28541 ns/op           12701 B/op        219 allocs/op
	BenchmarkDecodeStructJson1-12             307185             10635 ns/op            4521 B/op        100 allocs/op
	BenchmarkDecodeStructJson-12              103711             34813 ns/op            3512 B/op         64 allocs/op
	BenchmarkDecodeMapJson1-12                149858             23326 ns/op           13663 B/op        310 allocs/op
	BenchmarkDecodeMapJson-12                  96092             37011 ns/op           11801 B/op        231 allocs/op

## 测试分析

1. encode结构体的情况下，数据差异不大，原生甚至一定程度上稍微领先了。
2. encode Map的情况下，原生性能下降，看到大量的内存分配。
3. decode结构体的情况下，原生性能不如json-iterator，但是内存擦操作次数少于json-iterator。
3. decode Map的情况下，同上。

综上，我做一些推断：

### encode源码分析比较

原生marshal，基本使用的就是递归反射
```
func (e *encodeState) marshal(v interface{}, opts encOpts) (err error) {
	defer func() {
		if r := recover(); r != nil {
			if je, ok := r.(jsonError); ok {
				err = je.error
			} else {
				panic(r)
			}
		}
	}()
	e.reflectValue(reflect.ValueOf(v), opts)
	return nil
}
```

json-iterator也思路一样
```
// WriteVal copy the go interface into underlying JSON, same as json.Marshal
func (stream *Stream) WriteVal(val interface{}) {
	if nil == val {
		stream.WriteNil()
		return
	}
	cacheKey := reflect2.RTypeOf(val)
	encoder := stream.cfg.getEncoderFromCache(cacheKey)
	if encoder == nil {
		typ := reflect2.TypeOf(val)
		encoder = stream.cfg.EncoderOf(typ)
	}
	encoder.Encode(reflect2.PtrOf(val), stream)
}
```
但是仔细观察:
1. 他的代码对反射包进行了重新设计
2. 对不同的类型构建了不同的encoder

通过阅读代码，主要是通过每个重复类型，进行了判定此类型的encoder是否存在，存在则取之前的encoder，因此大量减少的encoder的创建和销毁操作。

这一块有个细节的数据结构操作，通过类型type地址去做。我摘取了部分源码出来演示并测试。

```
func TestUnsafePointer(t *testing.T) {
	s := unpackEFace(&T1)
	fmt.Println(uintptr(s.rtype))
	var T2 T
	s2 := unpackEFace(&T2)
	fmt.Println(uintptr(s2.rtype))
}

type eface struct {
	rtype unsafe.Pointer
	data  unsafe.Pointer
}

func unpackEFace(obj interface{}) *eface {
	return (*eface)(unsafe.Pointer(&obj))
}
=== RUN   TestUnsafePointer
18478592
18478592
--- PASS: TestUnsafePointer (0.00s)
```
因此这块就能解释为什么json-iterator的在encode的时候内存操作次数会远远低于原生。

但是这块有个疑问，为什么struct encode原生并不差，只是map差？

查阅源码，发现一个这玩意：
	// ConfigCompatibleWithStandardLibrary tries to be 100% compatible with standard library behavior
	var ConfigCompatibleWithStandardLibrary = Config{
		EscapeHTML:             true,
		SortMapKeys:            true,
		ValidateJsonRawMessage: true,
	}.Froze()

原来默认的并不是和官方结果百分百相似，因此，将测试用例调整为此对象。

```
json-iterator默认配置：
BenchmarkEncodeMapJson1-12                 96573             12353 ns/op            4633 B/op         31 allocs/op
BenchmarkEncodeMapJson-12                  41634             28370 ns/op           12701 B/op        219 allocs/op

json-iterator标准配置:
BenchmarkEncodeMapJson1-12                141724             25356 ns/op           11113 B/op        158 allocs/op
BenchmarkEncodeMapJson-12                 124508             29054 ns/op           12701 B/op        219 allocs/op
```

可以看出，内存虽然有一定优化，但是不如之前明显了。

1. struct 多次压缩时，encoding 中会缓存 name 信息, 以及对应val的类型，直接调用相应的encoder 即可;相反，map 则每次需要对key 做反射,根据类型判断获取key的值，val值也需要反射获取相应的encoder，时间浪费较多。
2. map 在做json 的解析的结果，会做排序操作。若修改源码，将排序操作屏蔽,key 越多，需要的时间越多。

而经过测试，结果体的结果基本不变，原生稍微比json-iterator优势。

### decode源码分析比较

其实两者的思路都是一样，遍历字符串，当遇到特殊的字符串的时候进行下一步操作。

原生的使用的是一次遍历完，并且通过反射填充数据，主要分为3种方式: object、array、其他。

原生json有几个重要对象，一个scanner对象，一个decodeState对象。

1. scanner :A scanner is a JSON scanning state machine. 官方的说法就是一个状态记录机。用其中一个parseState记录了特殊操作符的位置，可以理解为stack结构。
2. decodeState表示解码JSON值时的状态，可以理解为步操作机。
```
type scanner struct {
	step func(*scanner, byte) int
	endTop bool
	parseState []int
	err error
	bytes int64
}
type decodeState struct {
	data         []byte
	off          int // next read offset in data
	opcode       int // last read result
	scan         scanner
	errorContext struct { // provides context for type errors
		Struct     reflect.Type
		FieldStack []string
	}
	savedError            error
	useNumber             bool
	disallowUnknownFields bool
}
```

而json-iterator只有一个对象Iterator，作用就是用于迭代当前的bytes数据。

```
type Iterator struct {
	cfg              *frozenConfig
	reader           io.Reader
	buf              []byte
	head             int
	tail             int
	depth            int
	captureStartedAt int
	captured         []byte
	Error            error
	Attachment       interface{} // open for customized decoder
}
type ValDecoder interface {
	Decode(ptr unsafe.Pointer, iter *Iterator)
}
```
另外json-iterator提供了一个可供适配的接口ValDecoder用于解析不同的数据类型。也就是说，每个不同的数据类型，会生成新的对象，这也就能解释，为啥在decode json的时候，json-iterator的内存分配操作次数会大于原生。

为了证实我的推论，我测试一个最简单的json。

```
var TBytes1 = []byte(`{
	"foo":"test"
}`)
func BenchmarkSampleDecodeMapJson1(b *testing.B) {
	var TTMap map[string]interface{}
	for i := 0; i < b.N; i++ { //use b.N for looping
		Json1.Unmarshal(TBytes1, &TTMap)
	}
}

func BenchmarkSampleDecodeMapJson(b *testing.B) {
	var TTMap map[string]interface{}
	for i := 0; i < b.N; i++ { //use b.N for looping
		json.Unmarshal(TBytes1, &TTMap)
	}
}
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkSampleDecodeMapJson1-12         3971719               273.0 ns/op            56 B/op          5 allocs/op
BenchmarkSampleDecodeMapJson-12          1708197               687.0 ns/op           264 B/op          8 allocs/op
```


## 其他

ummashal测试过程中，发现错误的json格式的话，官方会极快的返回并很少的内存操作，但是json-iterator会去遍历生成迭代期对象直到错误为止。因此在使用过程中，如果决定使用json-iterator，最好保证是正确的数据。

下面是错误数据的decode的操作结果。
```
BenchmarkSampleDecodeMapJson1-12          423016              2469 ns/op            1673 B/op         41 allocs/op
BenchmarkSampleDecodeMapJson-12          1821427               656.6 ns/op           256 B/op          5 allocs/op
```

而官方做的是load数据的时候做检测，生成特殊op的操作位。
```
// checkValid verifies that data is valid JSON-encoded data.
// scan is passed in for use by checkValid to avoid an allocation.
func checkValid(data []byte, scan *scanner) error {
	scan.reset()
	for _, c := range data {
		scan.bytes++
		if scan.step(scan, c) == scanError {
			return scan.err
		}
	}
	if scan.eof() == scanError {
		return scan.err
	}
	return nil
}
```
而json-iterator并没有这个检测机制，好处是不用遍历2次。但是同样，如果是错误的json，会像上面所述的一样，错误直到解析不对为止。


## 更多的优化细节：

[官方优化思路](http://jsoniter.com/benchmark.html#optimization-used "优化思路")

另外，我当前测试的版本是1.16..5,而历史版本也需要测试，因此我选取版本为1.12.*、1.14.*，因此，最后追加这两个版本的测试结果。


```
mars@loalhost% /usr/local/go112/bin/go test -bench=. -benchtime=1s -benchmem -run=none 
goos: darwin
goarch: amd64
pkg: test
BenchmarkEncodeStructJson1-12    	  200000	      7487 ns/op	    3366 B/op	       2 allocs/op
BenchmarkEncodeStructJson-12     	  200000	      7353 ns/op	    3368 B/op	       2 allocs/op
BenchmarkEncodeMapJson1-12       	   50000	     26436 ns/op	   11744 B/op	     173 allocs/op
BenchmarkEncodeMapJson-12        	   50000	     30261 ns/op	   12821 B/op	     219 allocs/op
BenchmarkDecodeStructJson1-12    	  100000	     12610 ns/op	    4529 B/op	     100 allocs/op
BenchmarkDecodeStructJson-12     	   30000	     46576 ns/op	    3392 B/op	      61 allocs/op
BenchmarkDecodeMapJson1-12       	   50000	     24597 ns/op	   13704 B/op	     310 allocs/op
BenchmarkDecodeMapJson-12        	   30000	     48988 ns/op	   11908 B/op	     234 allocs/op
PASS
ok  	test	13.619s
mars@loalhot% /usr/local/go114/bin/go test -bench=. -benchtime=1s -benchmem -run=none 
goos: darwin
goarch: amd64
pkg: test
BenchmarkEncodeStructJson1-12    	  161550	      7294 ns/op	    3364 B/op	       2 allocs/op
BenchmarkEncodeStructJson-12     	  165632	      7202 ns/op	    3365 B/op	       2 allocs/op
BenchmarkEncodeMapJson1-12       	   44802	     26748 ns/op	   11716 B/op	     173 allocs/op
BenchmarkEncodeMapJson-12        	   38680	     30433 ns/op	   12821 B/op	     219 allocs/op
BenchmarkDecodeStructJson1-12    	   93560	     12380 ns/op	    4529 B/op	     100 allocs/op
BenchmarkDecodeStructJson-12     	   30903	     39145 ns/op	    3520 B/op	      64 allocs/op
BenchmarkDecodeMapJson1-12       	   47817	     25344 ns/op	   13703 B/op	     310 allocs/op
BenchmarkDecodeMapJson-12        	   29022	     41598 ns/op	   11921 B/op	     234 allocs/op
PASS
ok  	test	11.808s
```

可以看出，整体结果和1.16几乎没有差距。



