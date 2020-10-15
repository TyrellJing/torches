# grpc拦截器的那些事

上一篇介绍了gRPC的接口认证，我们客户端需要实现gRPC提供的接口，然后在服务端接口实现中通过metadata获取认证信息，进行判断，那么当我们有几十个几百个业务接口时，如果都在接口实现中去做，那将是一个噩梦，也不符合 DRY (Don't Repeat Yourself) 原则，今天一起看看如何通过gRPC的拦截器做到同一接口认证工作。

## 初识gRPC拦截器

gRPC 在 grpc 包中定义了一个变量，如下，这个变量叫做 UnaryServerInterceptor（一元服务拦截器）

```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, 
        handler UnaryHandler)  (resp interface{}, err error
)
```

其类型是一个函数，这个函数有，4 个入参，两个出参，介绍如下

- ctx context.Context 上下文

- req interface {} 用户请求的参数

- info UnaryServerInfo RPC 方法的所有信息，定义如下

```go
type UnaryServerInfo struct {        
// Server is the service implementation the user provides. 
// This is read-only.        
    Server interface{}        
// FullMethod is the full RPC method string,
// package.service/method.        
    FullMethod string
}
```
- handler UnaryHandler RPC 方法本身

- resp interface {} RPC 方法执行结果

```go
type UnaryHandler func(ctx context.Context,  req interface{}) (interface{}, error)
```

## 如何使用

首先定义一个拦截器

```go
//声明一个变量
var serverIntercept  grpc.UnaryServerInterceptor
 //为这个变量赋值
serverIntercept = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, 
        handler grpc.UnaryHandler) (resp interface{}, err error) {
         //这个方法的具体实现
         err = check(ctx) //权限校验
         if err != nil {
              return
         }
         // 校验通过后继续处理请求
         return handler(ctx, req)
}
```
在服务端启动时将拦截器添加进去

```go
 //将拦截器添加进去
gRpcServer :=grpc.NewServer(
    []grpc.ServerOption{grpc.UnaryInterceptor(serverIntercept)}...
)
```

如上就是服务端使用拦截器的所有步骤，客户端在访问服务端时就会被拦截

## 如何添加多个拦截器

有人说我看上面代码 grpc.NewServer 是一个可变参数，我传多个不就好了吗？整的是这样吗，我们来试试，代码如下我们添加了两个拦截器

```go
gRpcServer :=grpc.NewServer(
    []grpc.ServerOption{
    grpc.UnaryInterceptor(serverIntercept1),
    grpc.UnaryInterceptor(serverIntercept2)}...
)
```
很遗憾，服务端启动失败，报错信息如下，什么含义呢，意思是说，这个一元服务拦截器只能设置一个，不能重复，其实从名字就能看出，一元拦截器，就是说只能设置一个拦截器，gRPC 有意的阻止拦截器链的形式

```
panic: The unary server interceptor was already set and may not be reset. [recovered]
panic: The unary server interceptor was already set and may not be reset.
```

那我们如果真的需要拦截器链，该如何配置呢，核心思想是递归，代码如下:

```go
func InterceptChain(intercepts...grpc.UnaryServerInterceptor)grpc.UnaryServerInterceptor{
    //获取拦截器的长度
    l := len(intercepts)
    //如下我们返回一个拦截器
    return func(ctx context.Context, req interface{},
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error){
        //在这个拦截器中，我们做一些操作
        //构造一个链
        chain  := func(currentInter grpc.UnaryServerInterceptor,currentHandler grpc.UnaryHandler) 
        grpc.UnaryHandler {
            return func(currentCtx context.Context,
                currentReq interface{})(interface{}, error) {
                return currentInter(
                    currentCtx,
                    currentReq,
                    info,
                    currentHandler)
            }
        }
        //声明一个handler
        chainHandler := handler
        for i :=  l-1; i >= 0; i-- {
            //递归一层一层调用
            chainHandler = chain(intercepts[i],chainHandler)
        }
        //返回结果
        return chainHandler(ctx,req)
    }
}
```

github 上已经有这么一个项目，如下

https://github.com/grpc-ecosystem/go-grpc-...

这个项目提供了拦截器的 interceptor 链式的功能，还有其它一些功能，大家可以去学习学习。

## gRPC还有哪些拦截器

### 统一在 grpc 包下，其他拦截器如下

1. type UnaryClientInterceptor

> 这是一个客户端上的拦截器，在客户端真正发起调用之前，进行拦截，这是一个实验性的 api，这是 gRPC 官方的说法

2. type StreamClientInterceptor

> 在流式客户端调用时，通过拦截 clientstream 的创建，返回一个自定义的 clientstream, 可以做一些额外的操作，这是一个实验性的 api，这是 gRPC 官方的说法

3. type UnaryServerInterceptor (就是上面我们 demo 中的拦截器)

4. type StreamServerInterceptor

> 拦截服务器上流式 rpc 的执行

## gRPC 的拦截分类

按功能来分

- 一元拦截器 UnaryInterceptor

- 流式拦截器 StreamInterceptor

按端来分

- 客户端拦截器 ClientInterceptor

- 服务端拦截器 ServerInterceptor
