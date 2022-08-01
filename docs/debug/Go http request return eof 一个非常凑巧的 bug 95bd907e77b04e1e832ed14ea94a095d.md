# Go http request return eof: 一个非常凑巧的 bug

Create Time: July 30, 2022 12:08 AM
Created: July 30, 2022 12:08 AM
Last Edit Time: August 1, 2022 9:59 AM
Last Edited Time: August 1, 2022 9:59 AM
Type: Go Language

https://github.com/pyroscope-io/pyroscope 是一个持续 profiling 的平台，在接入官方的 https://github.com/pyroscope-io/client 之后，在上报 go pprof 数据的时候时不时的会出现报错：

```
2022-07-29 16:11:40.038 ERROR   biz/pprofx/logger.go:30 upload profile: do http request: Post "http://172.29.250.214:4040/ingest?aggregationType=sum&from=1659111090&name=dev--aoigate-0%7B%7D&sampleRate=100&spyName=gospy&units=samples&until=1659111100": EOF
```

看上去像是服务器关闭了连接，但是在 pyroscope 服务器上并没有看到错误日子。也怀疑是不是服务器上的文件描述符上限设置太低，尝试修改之后并没有效果。

于是，通过 wireshark 在 client 侧和 server 侧进行抓包，发现了个奇怪的现象，在 client 侧和 server 侧都抓到了报错的请求，并且没有回包，服务器日志中也不存在该请求。

开始怀疑是 pyroscope 服务器代码有 bug，在某个特殊情况下没有对请求进行处理，也没有打印错误日志。于是下载了一份源码，在请求处理的最上层添加了日志，还是没能打印出出错的请求。

继续往外层看代码，发现了 http server 的配置如下：

```go
ctrl.httpServer = &http.Server{
		Addr:           ctrl.config.APIBindAddr,
		Handler:        handler,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   15 * time.Second,
		IdleTimeout:    30 * time.Second,
		MaxHeaderBytes: 1 << 20,
		ErrorLog:       golog.New(w, "", 0),
	}
```

怀疑是否是 ReadTimeout 设置太短，导致超时关闭连接，于是将其修改为 30s，修改之后错误不再出现。

但是，回头去看客户端出错的日志，请求是再发出后立马就打印了错误，感觉不像请求超时导致的。

再将 ReadTimeout 设置成 3s, 5s, 发现也不会报错。显得有些奇怪。

没办法，只好再用 wireshark 进行抓包，这次不光看了 http 记录，还看了该连接完整的 tcp 记录。数据如下：

![Untitled](Go%20http%20request%20return%20eof%20%E4%B8%80%E4%B8%AA%E9%9D%9E%E5%B8%B8%E5%87%91%E5%B7%A7%E7%9A%84%20bug%2095bd907e77b04e1e832ed14ea94a095d/Untitled.png)

- tcp 建立连接
- tcp keep-alive 维持连接 10s
- 没有搜到请求，server 端主动关闭连接；与此同时，客户端发出请求
- 客户端搜到 FIN 包，报 EOF 错误
- 服务端搜到 http 请求，但是 socket 以及关闭，返回 RST 包

到这里，流程以及比较清晰了，一个不活跃的连接在 10s 后被 server 关闭，与此同时，客户端正好发出请求(每 10s 发送)， 在成功 Write 之后，才读到 EOF，产生了是发送请求之后才被 server 主动关闭连接的错觉。

IdleTimeOut 是 30s， 为什么 10s 服务器就主动关闭了连接？

```go
func Serve() error {
  for {
     req, err := read()
     if err != nil {
        return err
     }
     // handle req
     if d := c.server.idleTimeout(); d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
  }
}
```

看了下 http server 的源码，伪代码如上，server 是先成功读取第一个请求之后，才会进入 idle timeout 的逻辑，也就是说，如果建立连接之后没有在这个连接上发送请求，这个连接的超时时间实际上是 ReadTimeout.

client 会建立连接但是不发送请求吗？

```go
func (t *Transport) dialConnFor(w *wantConn) {
	defer w.afterDial()

	pc, err := t.dialConn(w.ctx, w.cm)
	delivered := w.tryDeliver(pc, err)
	if err == nil && (!delivered || pc.alt != nil) {
		// pconn was not passed to w,
		// or it is HTTP/2 and can be shared.
		// Add to the idle connection pool.
		t.putOrCloseIdleConn(pc)
	}
	if err != nil {
		t.decConnsPerHost(w.key)
	}
}
```

看了下 http client 的源码，建立连接是异步的，建立连接之后会尝试给它分配请求，但是又可能出现没有需要发出的请求，直接将连接放到 Idle 池子中的情况。

# 总结

- Server ReadTimeout 设置为 10s， 连接成功建立之后未收到请求，10s后会主动关闭连接
- Client 10s 前一次性发送多个请求，异步创建了多个 conn，但是有概率出现 conn 创建之后请求已经都完成的情况，导致出现建立之后不会离开发出请求的 conn
- 10s 之后，新的请求，取到了这个特殊的 conn，Write 成功(Server 端还没有收到)，与此同时 Server 端主动由于超时主动关闭了该 conn，client 侧读取到 EOF，报错
- 服务器在已经关闭的 conn 上收到 http 请求， 这也是服务器上用 wireshark 可以抓到包，但是 server 读取不到请求的原因
- 服务器返回 rst 包，但 client conn 读取到 eof 已经结束流程，也不会报 rst 错误