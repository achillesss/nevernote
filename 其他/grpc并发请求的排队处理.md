# gRPC并发请求的排队处理

我们的服务采用grpc框架搭建，任何用户的任何请求都将并发的向服务端发送，服务端同样也是并发的处理所有的简单rpc。其实这样挺好，不用我们自己去并发处理用户请求了。

但是实际上我们又碰到了很多奇奇怪怪的问题。

比如我们的上传数据接口，在响应部分有AlreadyUploaded错误码，如果能在数据库中查询到用户想上传的数据，则返回该错误码。理想情况下，用户在成功上传了数据A之后，如果出现了某种bug导致用户再次上传数据A，此时服务端是能够从数据库中查询到用户的数据A的，并能正确返回AlreadyUploaded错误码。但是剧情往往不这么走：

某用户的网络时好时坏，当他在请求上传数据的时候，因网络问题请求堵塞了，收不到响应的用户一脸懵比，于是疯狂的通过界面不断请求上传数据，大概过了几秒钟，网络突然好了，于是在请求堵塞的那段时间，被该用户积攒起来的数十个请求同时并发的被服务端接收到，于是服务端开始处理这几十个包含着一模一样数据A的一模一样的上传请求。经过若干步之后，服务端开始准备写入数据了，在此之前，为了保证数据不重复，于是服务端开始向数据库查询数据A是否被上传过。假设如下表展示的一样，暂且叫这个请求为请求1，查询的时候花了1ms，查询完到开始插入的时间为0.5ms，插入数据耗时1ms。一切都是很正常的。然而该用户当时同时发送了几十条请求，假设就发了两条。于是有了请求2，并且是和请求1并发的请求2，我们暂且假设请求2比请求1晚个0.1ms，于是问题出来了。请求2在向数据库查询数据A时，依然认为数据A没有被上传，所以请求2依然会将数据A写入数据库。于是，相同的数据A就被写入到了数据库两次，这并不是我们想要的ಥ\_ಥ，没想到你是这样的并发ಥ\_ಥ

| 请求 | 开始查询时间 | 结束查询时间 | 开始插入时间 | 结束插入时间 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 00:00:00:000000000 | 00:00:00:001000000 | 00:00:00:001500000 | 00:00:00:002500000 |
| 2 | 00:00:00:000100000 | 00:00:00:001100000 | 00:00:00:001600000 | 00:00:00:002600000 |

于是，我们有两种解决这种问题的办法。

第一种，也是非常有效的一种：通过对数据库列的unique约束，直接从根本上杜绝数据被重复写入。这样很省力，很有效，但是。但是我写入数据库的时候，数据库会返回一个Error，我怎么知道这个Error是不是重复上传啊亲，难道每次写这样的数据时，我写不成功的时候再去查一遍数据库来确认数据是不是被上传过了吗\( ・◇・\)？可是我想写数据库时，报错就代表真的是数据（结构）有错，而不是数据没有问题，只是已经写进去了而已，那样心好累。

第二种，将该上传请求进行排队处理。我要把同一个用户的同一个接口直接放到一个队列里面，并且去即时的处理这个队列，将处理后的结果放到另外一个结果队列，返回响应的时候再从这个结果队列里面即时的去取结果。这样，这个用户的上传请求就被串行的执行，每次写入的时候，能保证永远只有一个该用户的请求在执行，查一遍数据库，存在则不写入，写入失败就返回错误，这样就挺好。而且不用修改数据表的任何结构和条件，对于新手的我来说，简直就是完美的做法。

好，代码怎么写？

grpc在搭建服务时，可以启用若干个options，其中可以加入一个unaryInterceptor的option，用来控制所有请求的拦截，所有的unaryHandler全部通过这个interceptor进行处理：

```go
func runPlannerGRPC() {
    if lis, err := net.Listen("tcp", conf.Host.Server.ExerciseGRPC); err != nil {
        fmt.Printf("failed to listen: %v", err)
    } else {
    opts := []grpc.ServerOption{grpc.UnaryInterceptor(planner.Interceptor)}
    grpcServer := grpc.NewServer(opts...)
    server := planner.NewServer()
    corepb.RegisterExercisePlansServer(grpcServer, server)
    grpcServer.Serve(lis)
    }
}
```

```go
func Interceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (res interface{}, err error) {
    return handler(ctx, req)
}
```

用户每发送一个请求，就会新调用一个interceptor，所以为了对请求进行排队，我们得使用缓存，或者一个简单的channel：

```go
type handlerBody struct {
    requestID string //唯一标识一个request
    method    grpc.UnaryHandler
    ctx       context.Context
    req       interface{}
}

type handlerBodyChan chan *handlerBody //用来对请求进行排队的channel

var hbc = make(handlerBodyChan)

func newHandlerBodyChan(requestID string, ctx context.Context, req interface{}, grpcHandler grpc.UnaryHandler) {
    hb := new(handlerBody)
    hb.ctx = ctx
    hb.req = req
    hb.requestID = requestID //将请求进行标识
    hb.method = grpcHandler
    hbc <- hb
}
```

对请求进行排队之后，还得从队列中取出请求进行处理：

```go
type handlerResp struct {
    requestID string
    response  interface{}
    err       error
}

type handlerRespChan chan *handlerResp

var hrc = make(handlerRespChan)

func newHandlerRespChan(requestID string, resp interface{}, err error) {
    hr := new(handlerResp)
    hr.requestID = requestID //对请求的响应使用requestID进行标识
    hr.response = resp
    hr.err = err
    hrc <- hr
}
```

一个interceptor中的请求现在已经被放入到了请求队列中待处理，同时我们也能将这个请求的响应也放到一个队列中待取出，现在需要一个函数去处理队列中的请求：

```go
func handle(method grpc.UnaryHandler, ctx context.Context, req interface{}) (resp interface{}, err error) {
    return method(ctx, req)
}

func handleQuere() {
    for {
        select {
        case h := <-hbc: //监听channel中的请求，一旦发现，则立马处理，并将results放入响应队列中
            resp, err := handle(h.method, h.ctx, h.req)
            newHandlerRespChan(h.requestID, resp, err)
        default:
        }
    }
}
```

现在队列中的请求能被我们处理了，队列中的响应还需要我们正确的去返回：

```go
func receiveHandlerQueueResp(requestID string) (res interface{}, err error) {
    for {
        select {
        case h := <-hrc: // 监听channel中的响应，一旦发现，则立马返回正确的response
            if h.requestID != requestID { // 若requestID不匹配，则放入队列继续等待
                hrc <- h
            } else {
                return h.response, h.err
            }
        default:
        }
    }
}
```

到这里，我们已经可以正确的队列处理请求了，于是上面的Interceptor变成了：

```go
func Interceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (res interface{}, err error) {
    requestID := newRequestID(u.userID, info.FullMethod)
    newHandlerBodyChan(requestID, newCtx, req, handler)
    res, err = receiveHandlerQueueResp(requestID)
    return
}
```

同时，runGrpc时，也变成了：

```go
func runPlannerGRPC() {
    if lis, err := net.Listen("tcp", conf.Host.Server.ExerciseGRPC); err != nil {
        fmt.Printf("failed to listen: %v", err)
    } else {
        opts := []grpc.ServerOption{grpc.UnaryInterceptor(planner.Interceptor)}
        grpcServer := grpc.NewServer(opts...)
        server := planner.NewServer()
        corepb.RegisterExercisePlansServer(grpcServer, server)
        go planner.handleQuere()
        grpcServer.Serve(lis)
    }
}
```

此时，我们已经将任何请求都丢进了队列进行处理。然而这并不是我们想要的，我们需要对某一个用户的特定请求进行排队。所以，首先得列出一个待排队的请求列表名对请求进行筛选，其次，不同用户的请求要进行并发处理：

```go
func queue(userID int64, method grpc.UnaryHandler) bool {
// 判断该请求是否需要排队
}
```

于是Interceptor变成了：

```go
func Interceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (res interface{}, err error) {
    userID:= getUserID(ctx)
    if queue(userID, handler) {
        requestID := newRequestID(u.userID, info.FullMethod)
        newHandlerBodyChan(requestID, newCtx, req, handler)
        res, err = receiveHandlerQueueResp(requestID)
    } else {
        res, err = handler(ctx, request)
    }
    return
}
```
