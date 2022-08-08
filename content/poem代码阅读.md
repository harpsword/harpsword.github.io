+++

title = "poem代码阅读"
date = 2022-08-07

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++

# 背景
在Rust语言写的包中，poem包是一个出名的http包，除了基础的HTTP服务功能，还支持很多功能，包括tls、
session、log、错误处理等。

# Request定义
这些定义比较简单，在这里只是简单罗列一下。
```rust
/// Represents an HTTP request.
#[derive(Default)]
pub struct Request {
    method: Method,
    uri: Uri,
    version: Version,
    headers: HeaderMap,
    extensions: Extensions,
    body: Body,
    state: RequestState,
}
```

其他结构体
```rust
pub(crate) struct RequestState {
    pub(crate) local_addr: LocalAddr,
    pub(crate) remote_addr: RemoteAddr,
    pub(crate) scheme: Scheme,
    pub(crate) original_uri: Uri,
    pub(crate) match_params: PathParams,
    #[cfg(feature = "cookie")]
    pub(crate) cookie_jar: Option<CookieJar>,
    pub(crate) on_upgrade: Mutex<Option<OnUpgrade>>,
}

pub struct RequestParts {
    /// The request’s method
    pub method: Method,
    /// The request’s URI
    pub uri: Uri,
    /// The request’s version
    pub version: Version,
    /// The request’s headers
    pub headers: HeaderMap,
    /// The request’s extensions
    pub extensions: Extensions,
    pub(crate) state: RequestState,
}

pub struct RequestBuilder {
    method: Method,
    uri: Uri,
    version: Version,
    headers: HeaderMap,
    extensions: Extensions,
}
```

# Response定义
定义代码如下所示
```rust
/// Represents an HTTP response.
#[derive(Default)]
pub struct Response {
    status: StatusCode,
    version: Version,
    headers: HeaderMap,
    extensions: Extensions,
    body: Body,
}
```
和Request类似，也有一些相似的结构体定义。同样`ResponseBuilder`相较于`Response`少了一个body字段。
```rust
/// Component parts of an HTTP Response.
///
/// The HTTP response head consists of a status, version, and a set of header
/// fields.
pub struct ResponseParts {
    /// The response’s status
    pub status: StatusCode,
    /// The response’s version
    pub version: Version,
    /// The response’s headers
    pub headers: HeaderMap,
    /// The response’s extensions
    pub extensions: Extensions,
}
/// An response builder.
pub struct ResponseBuilder {
    status: StatusCode,
    version: Version,
    headers: HeaderMap,
    extensions: Extensions,
}
```

# 核心 Trait定义

## FromRequest
实现了`FromRequest` trait表示对应的数据能从`Reqeust`中获取到。采用async func的原因还未弄清楚。
```rust
#[async_trait::async_trait]
pub trait FromRequest<'a>: Sized {
    /// Extract from request head and body.
    async fn from_request(req: &'a Request, body: &mut RequestBody) -> Result<Self>;

    /// Extract from request head.
    ///
    /// If you know that this type does not need to extract the body, then you
    /// can just use it.
    ///
    /// For example [`Query`], [`Path`] they only extract the content from the
    /// request head, using this method would be more convenient.
    /// `String`,`Vec<u8>` they extract the body of the request, using this
    /// method will cause `ReadBodyError` error.
    async fn from_request_without_body(req: &'a Request) -> Result<Self> {
        Self::from_request(req, &mut Default::default()).await
    }
}
```

## IntoResponse
定义这个trait之后，在Endpoint中可以直接利用这个trait来定义Endpoint的返回值，用户只需要对自定义struct实现该接口，就可以把该自定义struct作为Endpoint定义函数的返回值。
```rust
pub trait IntoResponse: Send {
    /// Consume itself and return [`Response`].
    fn into_response(self) -> Response;

    /// Wrap an `impl IntoResponse` to add a header.
    ///
    /// # Example
    ///
    /// ```
    /// use poem::{http::HeaderValue, IntoResponse};
    ///
    /// # tokio::runtime::Runtime::new().unwrap().block_on(async {
    /// let resp = "hello".with_header("foo", "bar").into_response();
    /// assert_eq!(
    ///     resp.headers().get("foo"),
    ///     Some(&HeaderValue::from_static("bar"))
    /// );
    /// assert_eq!(resp.into_body().into_string().await.unwrap(), "hello");
    /// # });
    /// ```
    fn with_header<K, V>(self, key: K, value: V) -> WithHeader<Self>
    where
        K: TryInto<HeaderName>,
        V: TryInto<HeaderValue>,
        Self: Sized,
    {
        let key = key.try_into().ok();
        let value = value.try_into().ok();

        WithHeader {
            inner: self,
            header: key.zip(value),
        }
    }

    /// Wrap an `impl IntoResponse` to with a new content type.
    ///
    /// # Example
    ///
    /// ```
    /// use poem::{http::HeaderValue, IntoResponse};
    ///
    /// # tokio::runtime::Runtime::new().unwrap().block_on(async {
    /// let resp = "hello".with_content_type("text/abc").into_response();
    /// assert_eq!(resp.content_type(), Some("text/abc"));
    /// # });
    /// ```
    fn with_content_type<V>(self, content_type: V) -> WithContentType<Self>
    where
        V: TryInto<HeaderValue>,
        Self: Sized,
    {
        WithContentType {
            inner: self,
            content_type: content_type.try_into().ok(),
        }
    }

    /// Wrap an `impl IntoResponse` to set a status code.
    ///
    /// # Example
    ///
    /// ```
    /// use poem::{http::StatusCode, IntoResponse};
    ///
    /// # tokio::runtime::Runtime::new().unwrap().block_on(async {
    /// let resp = "hello".with_status(StatusCode::CONFLICT).into_response();
    /// assert_eq!(resp.status(), StatusCode::CONFLICT);
    /// assert_eq!(resp.into_body().into_string().await.unwrap(), "hello");
    /// # });
    /// ```
    fn with_status(self, status: StatusCode) -> WithStatus<Self>
    where
        Self: Sized,
    {
        WithStatus {
            inner: self,
            status,
        }
    }

    /// Wrap an `impl IntoResponse` to set a body.
    ///
    ///
    /// # Example
    ///
    /// ```
    /// use poem::{http::StatusCode, IntoResponse};
    ///
    /// # tokio::runtime::Runtime::new().unwrap().block_on(async {
    /// let resp = StatusCode::CONFLICT.with_body("hello").into_response();
    /// assert_eq!(resp.status(), StatusCode::CONFLICT);
    /// assert_eq!(resp.into_body().into_string().await.unwrap(), "hello");
    /// # });
    /// ```
    fn with_body(self, body: impl Into<Body>) -> WithBody<Self>
    where
        Self: Sized,
    {
        WithBody {
            inner: self,
            body: body.into(),
        }
    }
}
```

## Endpoint trait

核心 trait: Endpoint，如下代码所示，方法get_response有基本的实现，所以实现Endpoint方法时只需要实现call方法即可。这个trait和tower的Service trait抽象类似，只是更简单。

该trait利用了 `async_trait::async_trait`包来定义async func in trait. 没有这个包时无法定义async func in trait，可以阅读这篇文章来进一步了解[why async fn in traits are hard](https://smallcultfollowing.com/babysteps/blog/2019/10/26/async-fn-in-traits-are-hard/) 

```rust
/// An HTTP request handler.
#[async_trait::async_trait]
pub trait Endpoint: Send + Sync {
    /// Represents the response of the endpoint.
    type Output: IntoResponse;

    /// Get the response to the request.
    async fn call(&self, req: Request) -> Result<Self::Output>;

    /// Get the response to the request and return a [`Response`].
    ///
    /// Unlike [`Endpoint::call`], when an error occurs, it will also convert
    /// the error into a response object.
    ///
    /// # Example
    ///
    /// ```
    /// use poem::{
    ///     error::NotFoundError, handler, http::StatusCode, test::TestClient, Endpoint, Request,
    ///     Result,
    /// };
    ///
    /// #[handler]
    /// fn index() -> Result<()> {
    ///     Err(NotFoundError.into())
    /// }
    ///
    /// # tokio::runtime::Runtime::new().unwrap().block_on(async {
    /// TestClient::new(index)
    ///     .get("/")
    ///     .send()
    ///     .await
    ///     .assert_status(StatusCode::NOT_FOUND);
    /// # });
    /// ```
    async fn get_response(&self, req: Request) -> Response {
        self.call(req)
            .await
            .map(IntoResponse::into_response)
            .unwrap_or_else(|err| err.into_response())
    }
}
```

IntoEndpoint trait定义
```rust
pub trait IntoEndpoint {
    /// Represents the endpoint type.
    type Endpoint: Endpoint;

    /// Converts this object into endpoint.
    fn into_endpoint(self) -> Self::Endpoint;
}
// 对于实现了Endpoint trait的类型，自动实现IntoEndpoint
impl<T: Endpoint> IntoEndpoint for T {
    type Endpoint = T;

    fn into_endpoint(self) -> Self::Endpoint {
        self
    }
}
```

### EndpointExt
EndpointExt 中有很多很有意思的方法，比如`before`, `after`, `around`等等，可以对Endpoint的各个部分做定制化逻辑。
```rust
pub trait EndpointExt: IntoEndpoint {
	 fn boxed<'a>(self) -> BoxEndpoint<'a, <Self::Endpoint as Endpoint>::Output>
    where
        Self: Sized + 'a,
    {
        Box::new(self.into_endpoint())
    }
    ....
}
```


### handler 宏

handler宏的目的是为了把一个async函数包装成一个`Endpoint`。

example 如下所示。
```rust
#[handler]
fn hello(Path(name): Path<String>) -> String {
    format!("hello: {}", name)
}
```
expand之后的代码如下所示（经过删减），原始代码在附录A中。可以看到原始代码被塞在了`call`方法中。另外输入参数要求实现 `poem::FromRequest` trait，输出参数实现`poem::IntoResponse`trait。
```rust
struct Exmpale {}
#[allow(non_camel_case_types)]
struct hello;
impl poem::Endpoint for hello {
    type Output = poem::Response;
    fn call<'life0, 'async_trait>(
        &'life0 self,
        req: poem::Request,
    ) -> ::core::pin::Pin<
        Box<
            dyn ::core::future::Future<Output = poem::Result<Self::Output>>
                + ::core::marker::Send
                + 'async_trait,
        >,
    >
    where
        'life0: 'async_trait,
        Self: 'async_trait,
    {
        Box::pin(async move {
            if let ::core::option::Option::Some(__ret) =
                ::core::option::Option::None::<poem::Result<Self::Output>>
            {
                return __ret;
            }
            let __self = self;
            let mut req = req;
            let __ret: poem::Result<Self::Output> = {
                let (req, mut body) = req.split();
                let p0 = <Path<String> as poem::FromRequest>::from_request(&req, &mut body).await?;
                fn hello(Path(name): Path<String>) -> String {
                    {
                        let res = ::alloc::fmt::format(::core::fmt::Arguments::new_v1(
                            &["hello: "],
                            &[::core::fmt::ArgumentV1::new_display(&name)],
                        ));
                        res
                    }
                }
                let res = hello(p0);
                let res = poem::error::IntoResult::into_result(res);
                std::result::Result::map(res, poem::IntoResponse::into_response)
            };
            #[allow(unreachable_code)]
            __ret
        })
    }
}
```


# Middleware

## 核心trait: Middleware
目的是把一个Endpoint转换成另一个Endpoint。
```rust
pub trait Middleware<E: Endpoint> {
    /// New endpoint type.
    ///
    /// If you don't know what type to use, then you can use
    /// [`BoxEndpoint`](crate::endpoint::BoxEndpoint), which will bring some
    /// performance loss, but it is insignificant.
    type Output: Endpoint;

    /// Transform the input [`Endpoint`] to another one.
    fn transform(&self, ep: E) -> Self::Output;
}
```
poem中预先定义了很多Middleware

# Router

## RouteMethod
对于同一个url，不同的方法是绑在同一个`RouteMethod`上的。结构定义如下所示：
```rust
#[derive(Default)]
pub struct RouteMethod {
    methods: Vec<(Method, BoxEndpoint<'static>)>,
}

#[derive(Clone, PartialEq, Eq, Hash)]
pub struct Method(Inner);

#[derive(Clone, PartialEq, Eq, Hash)]
enum Inner {
    Options,
    Get,
    Post,
    Put,
    Delete,
    Head,
    Trace,
    Connect,
    Patch,
    // If the extension is short enough, store it inline
    ExtensionInline(InlineExtension),
    // Otherwise, allocate it
    ExtensionAllocated(AllocatedExtension),
}
```
`RouteMethod`也实现了`Endpoint`trait，在执行过程中会遍历`methods`来查找对应的handler(也是`Endpoint`)。

>问题：为啥`RouteMethod.methods`是Vec？而不是Map。可能是Method的数量比较少，用Vec性能更好。

```rust
#[async_trait::async_trait]
impl Endpoint for RouteMethod {
    type Output = Response;

    async fn call(&self, mut req: Request) -> Result<Self::Output> {
        match self
            .methods
            .iter()
            .find(|(method, _)| method == req.method())
            .map(|(_, ep)| ep)
        {
            Some(ep) => ep.call(req).await,
            None => {
                if req.method() == Method::HEAD {
                    req.set_method(Method::GET);
                    let mut resp = self.call(req).await?;
                    resp.set_body(());
                    return Ok(resp);
                }
                Err(MethodNotAllowedError.into())
            }
        }
    }
}
```

## RouteScheme
和RouteMethod不同，`RouteScheme`是在scheme维度上做路由，即http或者https
```rust
/// Routing object for request scheme
///
/// # Errors
///
/// - [`NotFoundError`]
#[derive(Default)]
pub struct RouteScheme {
    schemes: Vec<(Scheme, BoxEndpoint<'static>)>,
    fallback: Option<BoxEndpoint<'static>>,
}
```

## RouteDomain
`RouteDomain`是在域名维度上做路由
```rust

```

## Router
`Router`是在uri维度上做路由，采用RadixTree来作为数据结构（包括算法）。[RadixTree](https://toutiao.io/posts/o9anj0/preview)
```rust
#[derive(Default)]
pub struct Route {
    tree: RadixTree<BoxEndpoint<'static>>,
}
```

Route也实现了 `Endpoint`，调用call的时候会找到对应匹配的`Endpoint`继续往下走，直到最终走到`#[handler]`生成的业务代码。
```rust
#[async_trait::async_trait]
impl Endpoint for Route {
    type Output = Response;

    async fn call(&self, mut req: Request) -> Result<Self::Output> {
        match self.tree.matches(req.uri().path()) {
            Some(matches) => {
                req.state_mut().match_params.extend(matches.params);
                matches.data.call(req).await
            }
            None => Err(NotFoundError.into()),
        }
    }
}

```

## 寻找过程
server.rs中的`serve_connection`是处理过程的入口，参数中的`ep`就是从 `server.run(route)`中不断往下传的route(struct Route)。

```rust
async fn serve_connection(
    socket: impl AsyncRead + AsyncWrite + Send + Unpin + 'static,
    local_addr: LocalAddr,
    remote_addr: RemoteAddr,
    scheme: Scheme,
    ep: Arc<dyn Endpoint<Output = Response>>,
) {
    let service = hyper::service::service_fn({
        move |req: hyper::Request<hyper::Body>| {
            let ep = ep.clone();
            let local_addr = local_addr.clone();
            let remote_addr = remote_addr.clone();
            let scheme = scheme.clone();
            async move {
                Ok::<http::Response<_>, Infallible>(
                    ep.get_response((req, local_addr, remote_addr, scheme).into())
                        .await
                        .into(),
                )
            }
        }
    });

    let conn = Http::new()
        .serve_connection(socket, service)
        .with_upgrades();
    let _ = conn.await;
}
```


# 错误处理的一个例子
下面是一个handler的写法，在这个写法中，Error将会采用用户自定义的Error。
```rust
#[derive(Error, Debug)]
enum MyError {

}

impl ResponseError for MyError {
    fn status(&self) -> poem::http::StatusCode {
        StatusCode::BAD_REQUEST
    }
}
#[handler]
fn hello(Path(name): Path<String>) -> Result<String, MyError> {
    Ok(format!("hello: {}", name))
}
```
核心处理部分expand之后的代码如下所示
```rust
			let __ret: poem::Result<Self::Output> = {
                let (req, mut body) = req.split();
                let p0 = <Path<String> as poem::FromRequest>::from_request(&req, &mut body).await?;
                fn hello(Path(name): Path<String>) -> Result<String, MyError> {
                    Ok({
                        let res = ::alloc::fmt::format(::core::fmt::Arguments::new_v1(
                            &["hello: "],
                            &[::core::fmt::ArgumentV1::new_display(&name)],
                        ));
                        res
                    })
                }
                let res = hello(p0);
                let res = poem::error::IntoResult::into_result(res);
                std::result::Result::map(res, poem::IntoResponse::into_response)
            };
```
可以看到只需要为`Result<T, E>`实现`IntoResult`即可。
转换过程如下所示
	`MyError` -> `StatusCode` -> `poem::error::Error`
调用方法`IntoResult::into_result`，会使用默认实现来转换`E`为`poem::error::Error`。恰好`E`实现了`ResponseError+StdError+Send+Sync+'static`，所以转换能够成功。设计很精巧，最初设计的时候应该会有一个详细的转换图来方便设计，没有这张图的话，阅读源码的时候很难找到对应的默认实现，
```rust
pub trait IntoResult<T: IntoResponse> {
    /// Consumes this value returns a `poem::Result<T>`.
    fn into_result(self) -> Result<T>;
}

impl<T, E> IntoResult<T> for Result<T, E>
where
    T: IntoResponse,
    E: Into<Error> + Send + Sync + 'static,
{
    #[inline]
    fn into_result(self) -> Result<T> {
        self.map_err(Into::into)
    }
}

impl<T: IntoResponse> IntoResult<T> for T {
    #[inline]
    fn into_result(self) -> Result<T> {
        Ok(self)
    }
}

impl<T: ResponseError + StdError + Send + Sync + 'static> From<T> for Error {
    fn from(err: T) -> Self {
        Error {
            as_response: AsResponse::from_type::<T>(),
            source: Some(ErrorSource::BoxedError(Box::new(err))),
        }
    }
}
```


# 学到的经验
1. 链式套娃 `Endpoint` trait
	1. 把Router、RouteMethod、RouteScheme、RouteDomain都抽象为`Endpoint`
	2. 除了Router，还有Middleware,等等
2. 类似`Response` struct和 `IntoResponse` trait，在关联类型上可以采用 `Response` struct；generic上采用`IntoResponse` bound.


# 附录
## A. handler expand 结果
```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
use poem::{handler, web::Path};
struct Exmpale {}
#[allow(non_camel_case_types)]
struct hello;
impl poem::Endpoint for hello {
    type Output = poem::Response;
    #[allow(unused_mut)]
    #[allow(
        clippy::let_unit_value,
        clippy::no_effect_underscore_binding,
        clippy::shadow_same,
        clippy::type_complexity,
        clippy::type_repetition_in_bounds,
        clippy::used_underscore_binding
    )]
    fn call<'life0, 'async_trait>(
        &'life0 self,
        req: poem::Request,
    ) -> ::core::pin::Pin<
        Box<
            dyn ::core::future::Future<Output = poem::Result<Self::Output>>
                + ::core::marker::Send
                + 'async_trait,
        >,
    >
    where
        'life0: 'async_trait,
        Self: 'async_trait,
    {
        Box::pin(async move {
            if let ::core::option::Option::Some(__ret) =
                ::core::option::Option::None::<poem::Result<Self::Output>>
            {
                return __ret;
            }
            let __self = self;
            let mut req = req;
            let __ret: poem::Result<Self::Output> = {
                let (req, mut body) = req.split();
                let p0 = <Path<String> as poem::FromRequest>::from_request(&req, &mut body).await?;
                fn hello(Path(name): Path<String>) -> String {
                    {
                        let res = ::alloc::fmt::format(::core::fmt::Arguments::new_v1(
                            &["hello: "],
                            &[::core::fmt::ArgumentV1::new_display(&name)],
                        ));
                        res
                    }
                }
                let res = hello(p0);
                let res = poem::error::IntoResult::into_result(res);
                std::result::Result::map(res, poem::IntoResponse::into_response)
            };
            #[allow(unreachable_code)]
            __ret
        })
    }
}
```