# 代码生成器

`protoc`默认支持的只有`c++`，`java`和`python`，除此之外，提供了插件的接口。原生的也好插件也好，需要知道的是，`proto`文件的`token`解析，词法检查，语法检查都是在`protoc`中完成的，在完成之后每个`proto`文件会对应一棵语法树，此时如果是默认支持的三种语言会继续在`protoc`处理，否则会`fork`出一个子进程，它会查找插件并使用`exec`来执行插件。父进程会通过管道将请求传递出去，子进程会将回复传递回去。子进程会将标准输入和标准输出分别重定向到管道的读端和写端。因此插件需要做的就是读取标准输入反序列化到`Request`，完成工作后将`Response`序列化到标准输出。

`protoc`序列化的内容可以通过`src/google/protobuf/compiler/plugin.proto`反序列化回来，这样插件就得到了`protoc`的请求，也可以将最终的结果反序列化成字节流。==这个文件很重要，通过了解这个proto文件可以知道插件可以实现什么功能==。

请求中最重要的一个参数是`FileDescriptorProto `，这个结构是对`proto`文件的抽象，可以理解为`proto`文件的语法树，每一个编译的`proto`文件都会有一个`FileDescriptorProto `实例对应。其实`proto`中的类型的抽象结构都已经在`src/google/protobuf/descriptor.proto`文件中定义好了，而`FileDescriptorProto`正是使用其中抽象结构来抽象一个`proto`文件。

了解了上述内容就可以开发`protoc`的插件了，可以生成各种东西了，不止可以生成代码。最后说一句子进程查找可执行文件名称为`protoc-gen-xx`，这里的`xx`是从使用`protoc`传入的`--xx_out`决定的。好了，知道了这些，就可以操作起来了。