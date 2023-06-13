# IO
![image.png](https://cdn.nlark.com/yuque/0/2023/png/362867/1686640005354-efab225e-95aa-46d8-aa32-2bcb7e13a096.png#averageHue=%23565452&clientId=u24f31a96-8647-4&from=paste&height=236&id=u794d4809&originHeight=471&originWidth=1080&originalType=binary&ratio=2&rotation=0&showTitle=false&size=72476&status=done&style=none&taskId=u61512ff0-9598-4cef-8544-c47216ea256&title=&width=540)

- io：基础的 IO 库，提供了`Reader`和`Writer`接口。
> 其中的`os`包、`net`包、`string`包、`bytes`包以及`bufio`包都实现了`io`中的`Reader`或`Writer`接口。

- os：提供了访问底层操作系统资源的能力，如文件读写、进程控制等。
- net：提供了网络相关的 IO 功能，如TCP、UDP通信、HTTP请求等。
- string.Reader：提供了`string`的读取。因为`string`不能写，所以只有`Reader`。
- bytes.Buffer和Reader：提供了对字节内容的读写。
- bufio：提供带缓存的 I/O 操作，解决频繁、少量读取场景下的性能问题。这里利用了计算机的局部性原理。
- ioutil：提供了一些方便的文件读写函数，如`ReadFile`和`WriteFile`。
