## 自己动手实现okhttp
### 一. 问题
在分析okhttp源码之前，我想先提出一个问题，如果我们自己来设计一个网络请求库，这个库应该长什么样子？大致是什么结构呢？

下面我和大家一起来构建一个网络请求库，并在其中融入okhttp中核心的设计思想，希望借此让读者感受并学习到okhttp中的精华之处，而非仅限于了解其实现。

### 二.  思考  
首先，我们假设要构建的的网络请求库叫做WingjayHttpClient，那么，作为一个网络请求库，它最基本功能是什么呢？

在我看来应该是：接收用户的请求 -> 发出请求 -> 接收响应结果并返回给用户。

那么从使用者角度而言，需要做的事是：

 1.  创建一个Request：在里面设置好目标URL；请求method如GET/POST等；一些header如Host、User-Agent等；如果你在POST上传一个表单，那么还需要body。
 2.  将创建好的Request传递给WingjayHttpClient。
 3.  WingjayHttpClient去执行Request，并把返回结果封装成一个Response给用户。而一个Response里应该包括statusCode如200，一些header如content-type等，可能还有body    

到此即为一次完整请求的雏形。那么下面我们来具体实现这三步。

### 三. 雏形实现

#### 1.  创建Request类

首先，我们要建立一个Request类，利用Request类用户可以把自己需要的参数传入进去，基本形式如下：
~~~ java
class Request {
	String url;
	String method;
	Headers headers;
	Body requestBody;

	public Request(String url, String method, @Nullable Headers headers, @Nullable Body body) {
		this.url = url;
		...
	}
}
~~~
#### 2.  将Request对象传递给WingjayHttpClient

我们可以设计WingjayHttpClient如下：
~~~ java
class WingjayHttpClient {
	public Response sendRequest(Request request) {
		return executeRequest(request);
	}
}
~~~
#### 3. 执行Request，并把返回结果封装成一个Response返回

~~~ java
class WingjayHttpClient {
	...
	private Response executeRequest(Request request) {
		//使用socket来进行访问
		Socket socket = new Socket(request.getUrl(), 80);
		ResponseData data = socket.connect().getResponseData();
		return new Response(data);
	}
	...
}

class Response {
	int statusCode;
	Headers headers;
	Body responseBody
	...
}
~~~
### 四. 功能扩展
利用上面的雏形，可以得到其使用方法如下：
~~~ java
Request request = new Request("http://wingjay.com");
WingjayHttpClient client = new WingjayHttpClient();
Response response = client.sendRequest(request);
handle(response);
~~~

然而，上面的雏形是远远不能胜任常规的应用需求的，因此，下面再来对它添加一些常用的功能模块。

#### 1. 重新把简陋的user Request组装成一个规范的http request

一般的request中，往往用户只会指定一个URL和method，这个简单的user request是不足以成为一个http request，我们还需要为它添加一些header，如Content-Length, Transfer-Encoding, User-Agent, Host, Connection, 和 Content-Type，如果这个request使用了cookie，那我们还要将cookie添加到这个request中。

我们可以扩展上面的sendRequest(request)方法：

~~~ java
public Response sendRequest(Request userRequest) {
    Request httpRequest = expandHeaders(userRequest);
    return executeRequest(httpRequest);
}

private Request expandHeaders(Request userRequest) {
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
    ...
}
~~~

####  2. 支持自动重定向

有时我们请求的URL已经被移走了，此时server会返回301状态码和一个重定向的新URL，此时我们要能够支持自动访问新URL而不是向用户报错。

对于重定向这里有一个测试性URL：http://www.publicobject.com/helloworld.txt ，通过访问并抓包，可以看到如下信息：
![](markdown-img-paste-20170217172852205.png)


因此，我们在接收到Response后要根据status_code是否为重定向，如果是，则要从Response Header里解析出新的URL－Location并自动请求新URL。那么，我们可以继续改写sendRequest(request)方法：

~~~ java

private boolean allowRedirect = true;
// user can set redirect status when building WingjayHttpClient
public void setAllowRedirect(boolean allowRedirect) {
	this.allowRedirect = allowRedirect;
}

public Response sendRequest(Request userRequest) {
		Request httpRequest = expandHeaders(userRequest);
		Response response = executeRequest(httpRequest);
		switch (response.statusCode()) {
			// 300: multi choice; 301: moven permanently;
			// 302: moved temporarily; 303: see other;
			// 307: redirect temporarily; 308: redirect permanently
			case 300:
			case 301:
			case 302:
			case 303:
			case 307:
			case 308:
				return handleRedirect(response);
			default:
				return response;
		}

}
// the max times of followup request
private static final int MAX_FOLLOW_UPS = 20;
private int followupCount = 0;

private Response handleRedirect(Response response) {
	// Does the WingjayHttpClient allow redirect?
	if (!client.allowRedirect()) {
		return null;
	}

	// Get the redirecting url
	String nextUrl = response.header("Location");

	// Construct a redirecting request
	Request followup = new Request(nextUrl);

	// check the max followupCount
	if (++followupCount > MAX_FOLLOW_UPS) {
		throw new Exception("Too many follow-up requests: " + followUpCount);
	}

	// not reach the max followup times, send followup request then.
	return sendRequest(followup);
}
~~~
利用上面的代码，我们通过获取原始userRequest的返回结果，判断结果是否为重定向，并做出自动followup处理。

一些常用的状态码
100~199：指示信息，表示请求已接收，继续处理
200~299：请求成功，表示请求已被成功接收、理解、接受
300~399：重定向，要完成请求必须进行更进一步的操作
400~499：客户端错误，请求有语法错误或请求无法实现
500~599：服务器端错误，服务器未能实现合法的请求

#### 3. 支持重试机制

所谓重试，和重定向非常类似，即通过判断Response状态，如果连接服务器失败等，那么可以尝试获取一个新的路径进行重新连接，大致的实现和重定向非常类似，此不赘述。


#### 4. Request & Response 拦截机制

这是非常核心的部分。

通过上面的重新组装request和重定向机制，我们可以感受的，一个request从user创建出来后，会经过层层处理后，才真正发出去，而一个response，也会经过各种处理，最终返回给用户。

笔者认为这和网络协议栈非常相似，用户在应用层发出简单的数据，然后经过传输层、网络层等，层层封装后真正把请求从物理层发出去，当请求结果回来后又层层解析，最终把最直接的结果返回给用户使用。

最重要的是，每一层都是抽象的，互不相关的！

因此在我们设计时，也可以借鉴这个思想，通过设置拦截器Interceptor，每个拦截器会做两件事情：

1. 接收上一层拦截器封装后的request，然后自身对这个request进行处理，例如添加一些header，处理后向下传递；
2. 接收下一层拦截器传递回来的response，然后自身对response进行处理，例如判断返回的statusCode，然后进一步处理。
那么，我们可以为拦截器定义一个抽象接口，然后去实现具体的拦截器。

~~~ java
interface Interceptor {
	Response intercept(Request request);
}
~~~
大家可以看下上面这个拦截器设计是否有问题？
我们想象这个拦截器能够接收一个request，进行拦截处理，并返回结果。

但实际上，它无法返回结果，而且它在处理request后，并不能继续向下传递，因为它并不知道下一个Interceptor在哪里，也就无法继续向下传递。

那么，如何解决才能把所有Interceptor串在一起，并能够依次传递下去。

~~~ java
public interface Interceptor {
  Response intercept(Chain chain);

  interface Chain {
    Request request();

    Response proceed(Request request);
  }
}
~~~
使用方法如下：假如我们现在有三个Interceptor需要依次拦截：

~~~ java
// Build a full stack of interceptors.
List<Interceptor> interceptors = new ArrayList<>();
interceptors.add(new MyInterceptor1());
interceptors.add(new MyInterceptor2());
interceptors.add(new MyInterceptor3());

Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, 0, originalRequest);
chain.proceed(originalRequest);
~~~
里面的RealInterceptorChain的基本思想是：我们把所有interceptors传进去，然后chain去依次把request传入到每一个interceptors进行拦截即可。

通过下面的示意图可以明确看出拦截流程：
![](markdown-img-paste-20170217173721221.png)


其中，RetryAndFollowupInterceptor是用来做自动重试和自动重定向的拦截器；BridgeInterceptor是用来扩展request的header的拦截器。这两个拦截器存在于okhttp里，实际上在okhttp里还有好几个拦截器，这里暂时不做深入分析。

![](markdown-img-paste-20170217173814186.png)

1.  CacheInterceptor
这是用来拦截请求并提供缓存的，当request进入这一层，它会自动去检查缓存，如果有，就直接返回缓存结果；否则的话才将request继续向下传递。而且，当下层把response返回到这一层，它会根据需求进行缓存处理；

2. ConnectInterceptor
这一层是用来与目标服务器建立连接

3. CallServerInterceptor
这一层位于最底层，直接向服务器发出请求，并接收服务器返回的response，并向上层层传递。

上面几个都是okhttp自带的，也就是说需要在WingjayHttpClient自己实现的。除了这几个功能性的拦截器，我们还要支持用户自定义拦截器，主要有以下两种（见图中非虚线框蓝色字部分）：

1. interceptors
这里的拦截器是拦截用户最原始的request。

2. NetworkInterceptor
这是最底层的request拦截器。

如何区分这两个呢？举个例子，我创建两个LoggingInterceptor，分别放在interceptors层和NetworkInterceptor层，然后访问一个会重定向的URL_1，当访问完URL_1后会再去访问重定向后的新地址URL_2。对于这个过程，interceptors层的拦截器只会拦截到URL_1的request，而在NetworkInterceptor层的拦截器则会同时拦截到URL_1和URL_2两个request。具体原因可以看上面的图。


####  5. 同步、异步 Request池管理机制
在实际应用中，一个WingjayHttpClient可能会被用于同时处理几十个用户request，而且这些request里还分成了同步和异步两种不同的请求方式，所以我们显然不能简单把一个request直接塞给WingjayHttpClient。

我们知道，一个request除了上面定义的http协议相关的内容，还应该要设置其处理方式同步和异步。那这些信息应该存在哪里呢？两种选择：

直接放入Request
从理论上来讲是可以的，但是却违背了初衷。我们最开始是希望用Request来构造符合http协议的一个请求，里面应该包含的是请求目标网址URL，请求端口，请求方法等等信息，而http协议是不关心这个request是同步还是异步之类的信息

创建一个类，专门来管理Request的状态
这是更为合适的，我们可以更好的拆分职责。

因此，这里选择创建两个类SyncCall和AsyncCall，用来区分同步和异步。

~~~ java
class SyncCall {
	private Request userRequest;

	public SyncCall(Request userRequest) {
		this.userRequest = userRequest;
	}
}

class AsyncCall {
	private Request userRequest;
	private Callback callback;

	public AsyncCall(Request userRequest, Callback callback) {
		this.userRequest = userRequest;
		this.callback = callback;
	}

	interface Callback {
		void onFailure(Call call, IOException e);
		void onResponse(Call call, Response response) throws IOException;
	}
}
~~~
基于上面两个类，我们的使用场景如下：

~~~ java
WingjayHttpClient client = new WingjayHttpClient();
// Sync
Request syncRequest = new Request("http://wingjay.com");
SyncCall syncCall = new SyncCall(request);
Response response = client.sendSyncCall(syncCall);
handle(response);

// Async
AsyncCall asyncCall = new AsyncCall(request, new CallBack() {
	  @Override
      public void onFailure(Call call, IOException e) {}

      @Override
      public void onResponse(Call call, Response response) throws IOException {
        handle(response);
      }
});
client.equeueAsyncCall(asyncCall);
~~~
从上面的代码可以看到，WingjayHttpClient的职责发生了变化：以前是response = client.sendRequest(request);，而现在变成了

~~~ java
response = client.sendSyncCall(syncCall);
client.equeueAsyncCall(asyncCall);
~~~
