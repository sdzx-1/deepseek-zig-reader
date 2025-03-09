# deepseek-zig-reader
使用deepseek api自动总结一些zig代码。

zig 编译器代码：
1. https://github.com/ziglang/zig/tree/master/src
2. https://github.com/ziglang/zig/tree/master/lib/std
2. https://github.com/ziglang/zig/tree/master/tools

zls 代码：
1. https://github.com/zigtools/zls/tree/master/src

libxev 代码：
1. https://github.com/mitchellh/libxev/tree/main/src

mach 代码：
1. https://github.com/hexops/mach/tree/main/src

当前只阅读目录下所有的zig文件。每个.zig文件都有对应的.md总结文件。

在.md文件中通过分割线区分推理部分和实际输出部分。

*.error.log 文件中代表没有成功通过deepseek总结的代码，一般的原因是文件太大。

当然也可能有其他的错误信息,比如服务器响应错误。

我这里限制文件小于180K。如果文件大于180K将不会进行deepseek分析。

对于大于180K的文件，更好的方式是将文件中单个将函数提取出来，再交给deepseek分析。

至于为啥限制文件大小为180K? 这个是我凭经验随便定的。。。。。

提示词是: 总结这个代码，总结主要函数的流程

温度：0.6

上传文件的格式采用官方推荐的方式。

```
file_template = \
"""[file name]: {file_name}
[file content begin]
{file_content}
[file content end]
{question}"""
```

由于最近deepseek 在0 ~ 8点之间价格为原来的四分之一。于是我在凌晨运行总结脚本，因此上述api调用的最终费用小于20 元。


