+++

title = "tower源码阅读"
date = 2022-08-13

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++


# 核心Trait

## Service
核心Trait定义如下所示，Response采样关联类型是因为一个Request和一个Service实现应该对应一个Response。
```rust
pub trait Service<Request> {
    /// Responses given by the service.
    type Response;

    /// Errors produced by the service.
    type Error;

    /// The future response value.
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    /// Returns `Poll::Ready(Ok(()))` when the service is able to process requests.
    ///
    /// If the service is at capacity, then `Poll::Pending` is returned and the task
    /// is notified when the service becomes ready again. This function is
    /// expected to be called while on a task. Generally, this can be done with
    /// a simple `futures::future::poll_fn` call.
    ///
    /// If `Poll::Ready(Err(_))` is returned, the service is no longer able to service requests
    /// and the caller should discard the service instance.
    ///
    /// Once `poll_ready` returns `Poll::Ready(Ok(()))`, a request may be dispatched to the
    /// service using `call`. Until a request is dispatched, repeated calls to
    /// `poll_ready` must return either `Poll::Ready(Ok(()))` or `Poll::Ready(Err(_))`.
    ///
    /// Note that `poll_ready` may reserve shared resources that are consumed in a subsequent
    /// invocation of `call`. Thus, it is critical for implementations to not assume that `call`
    /// will always be invoked to ensure that such resources are released if the service is
    /// dropped before `call` is invoked or the future returned by `call` is dropped before it
    /// is polled.
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    /// Process the request and return the response asynchronously.
    ///
    /// This function is expected to be callable off task. As such,
    /// implementations should take care to not call `poll_ready`.
    ///
    /// Before dispatching a request, `poll_ready` must be called and return
    /// `Poll::Ready(Ok(()))`.
    ///
    /// # Panics
    ///
    /// Implementations are permitted to panic if `call` is invoked without
    /// obtaining `Poll::Ready(Ok(()))` from `poll_ready`.
    fn call(&mut self, req: Request) -> Self::Future;
}

```
默认实现：
1. 针对独占引用(原类型S实现了Service)实现了Service
2. 针对`Box<S>`(原类型S实现了Service)实现了Service
这两个默认实现可以方便使用。
```rust
impl<'a, S, Request> Service<Request> for &'a mut S
where
    S: Service<Request> + 'a,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), S::Error>> {
        (**self).poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> S::Future {
        (**self).call(request)
    }
}

impl<S, Request> Service<Request> for Box<S>
where
    S: Service<Request> + ?Sized,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), S::Error>> {
        (**self).poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> S::Future {
        (**self).call(request)
    }
}
```
## Layer
输入是一个Service，输出也是一个Service。目的是为了方便做不同组件之间的套娃。
```rust
pub trait Layer<S> {
    /// The wrapped service
    type Service;
    /// Wrap the given service with the middleware, returning a new service
    /// that has been decorated with the middleware.
    fn layer(&self, inner: S) -> Self::Service;
}
```
默认实现，给共享引用实现了Layer(原变量实现了Layer)，和Service类似。
```rust
impl<'a, T, S> Layer<S> for &'a T
where
    T: ?Sized + Layer<S>,
{
    type Service = T::Service;

    fn layer(&self, inner: S) -> Self::Service {
        (**self).layer(inner)
    }
}
```

# 实现的一些中间件
## Timeout
```rust
#[derive(Debug, Clone)]
pub struct Timeout<T> {
    inner: T,
    timeout: Duration,
}

impl<S, Request> Service<Request> for Timeout<S>
where
    S: Service<Request>,
    S::Error: Into<crate::BoxError>,
{
    type Response = S::Response;
    type Error = crate::BoxError;
    type Future = ResponseFuture<S::Future>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        match self.inner.poll_ready(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(r) => Poll::Ready(r.map_err(Into::into)),
        }
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let response = self.inner.call(request);
        let sleep = tokio::time::sleep(self.timeout);

        ResponseFuture::new(response, sleep)
    }
}
```
实现过程中对于Self::Future采用了ResponseFuture struct来实现。在为ResponseFuture实现Future trait的过程中，需要Pin住`this.response, this.sleep`才能调用对应的poll方法，采用`pin_project`来实现。
```rust
pin_project! {
    /// [`Timeout`] response future
    ///
    /// [`Timeout`]: crate::timeout::Timeout
    #[derive(Debug)]
    pub struct ResponseFuture<T> {
        #[pin]
        response: T,
        #[pin]
        sleep: Sleep,
    }
}

impl<T> ResponseFuture<T> {
    pub(crate) fn new(response: T, sleep: Sleep) -> Self {
        ResponseFuture { response, sleep }
    }
}

impl<F, T, E> Future for ResponseFuture<F>
where
    F: Future<Output = Result<T, E>>,
    E: Into<crate::BoxError>,
{
    type Output = Result<T, crate::BoxError>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();

        // First, try polling the future
        // require this.response be Pin!
        match this.response.poll(cx) {
            Poll::Ready(v) => return Poll::Ready(v.map_err(Into::into)),
            Poll::Pending => {}
        }

        // Now check the sleep
        // require this.sleep be Pin!
        match this.sleep.poll(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(_) => Poll::Ready(Err(Elapsed(()).into())),
        }
    }
}
```

### Layer
为中间件实现了Layer trait，方便叠加。
```rust
#[derive(Debug, Clone)]
pub struct TimeoutLayer {
    timeout: Duration,
}

impl TimeoutLayer {
    /// Create a timeout from a duration
    pub fn new(timeout: Duration) -> Self {
        TimeoutLayer { timeout }
    }
}

impl<S> Layer<S> for TimeoutLayer {
    type Service = Timeout<S>;

    fn layer(&self, service: S) -> Self::Service {
        Timeout::new(service, self.timeout)
    }
}
```

# 思考
poem的整体设计和tower很像，应该进行了一定的参考。比如poem中的Endpoint和tower的Service trait很像，都是对一次请求处理的抽象，只不过Endpoint定位具体的场景，Request和Response采用了具体的结构体，而非关联对象或者泛型。

poem中的Middleware和tower中的Layer也很像。Middleware trait实现的方法输入和输出都是Endpoint trait object，可以实现Endpoint之间的连接。Layer trait实现的方法输入和输出没有强制限制都是Service triat object，但是可以用于这种场景。
# Ref
1. https://tokio.rs/blog/2021-05-14-inventing-the-service-trait#footnotes
2. 