# **Retrofit 2.9.0**

## 简单使用
```
val okHttpClient = OkHttpClient.Builder().apply {
    connectTimeout(10 * 1000L, TimeUnit.MILLISECONDS)
    readTimeout(10 * 1000L, TimeUnit.MILLISECONDS)
    writeTimeout(10 * 1000L, TimeUnit.MILLISECONDS)
    addInterceptor(HttpLoggingInterceptor())
}.build()
val retrofit = Retrofit.Builder().apply {
    baseUrl("https://www.google.com/")
    addConverterFactory(GsonConverterFactory.create())
    client(okHttpClient)
}.build()
val call = retrofit.create(Api::class.java)
    .startGoogle()
call.enqueue(object : Callback<Any> {
    override fun onResponse(call: Call<Any>, response: Response<Any>) {
        TODO("Not yet implemented")
    }

    override fun onFailure(call: Call<Any>, t: Throwable) {
        TODO("Not yet implemented")
    }
})
```

## Retrofit 的创建过程
```
public Retrofit build() {
    if (baseUrl == null) {
    t    hrow new IllegalStateException("Base URL required.");
    }
    
    // 创建 OkHttpClient 实例，callFactory 是通过 Builder 的 client 方法设置的 OkHttpClient
    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }
    
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        // 对于 Android 平台获取到的默认 Executor 为 MainThreadExecutor
        callbackExecutor = platform.defaultCallbackExecutor();
    }
    
    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
    
    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
    
    // Add the built-in converter factory first. This prevents overriding its behavior but also
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());
    
    // 调用 Retrofit 的构造方法创建实例
    return new Retrofit(
      callFactory,
      baseUrl,
      unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories),
      callbackExecutor,
      validateEagerly);
    }
}
```
在 Retrofit.Builder 的 build 方法中，会检查 baseUrl，如果没有设置 OkHttpClient 则创建默认的，检查 callbackExecutor 是否为空，是的话就获取平台默认的。合并添加的 CallAdapter 和平台默认的 CallAdapter，合并添加的 Converter 和平台默认的并添加内建的 Converter，调用 Retrofit 的构造函数创建实例。

## create 方法
```
public <T> T create(final Class<T> service) {
validateServiceInterface(service);
return (T)
    Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] {service},
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override
          public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            args = args != null ? args : emptyArgs;
            return platform.isDefaultMethod(method)
                ? platform.invokeDefaultMethod(method, service, proxy, args)
                : loadServiceMethod(method).invoke(args);
          }
        });
}
```
create 方法通过动态代理返回 ApiService 对象。运行期会生成 ApiService 的实现类，当调用 ApiService 的方法时，InvocationHandler 的 invoke 方法会拦截方法调用，也就是执行了 invoke 方法。在 invoke 方法中，如果 Method 是 Object 类型，则直接调用它的方法。如果是默认方法则执行平台默认方法，否则执行 loadServiceMethod。在我们的代码中，通常情况都是执行 loadServiceMethod：
```
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
在这个方法中会检查 ServiceMethod 是否有缓存，有的话直接复用。否则调用 ServiceMethod.parseAnnotations 创建 ServiceMethod 并放入 serviceMethodCache 中。接着看 ServiceMethod.parseAnnotations 方法：
```
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```
通过 RequestFactory.parseAnnotations 创建 RequestFactory，最后调用 HttpServiceMethod.parseAnnotations 方法：
```
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    // 下面代码主要是获取请求方法的返回类型，kotlin的挂起函数和普通函数获取方式有所不同
    if (isKotlinSuspendFunction) {
      Type[] parameterTypes = method.getGenericParameterTypes();
      Type responseType =
          Utils.getParameterLowerBound(
              0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // Unwrap the actual body type from Response<T>.
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        continuationWantsResponse = true;
      } else {
        // TODO figure out if type is nullable or not
        // Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
        // Find the entry for method
        // Determine if return type is nullable or not
      }

      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else {
      adapterType = method.getGenericReturnType();
    }

    // 获取 CallAdapter 对象
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    ...
    // 获取对应的 Converter 来转换返回的数据
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
        // 如果不是挂起函数则直接返回 CallAdapted
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
        // 需要返回完整的 Response
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
        // 只要返回 ResponseBody
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
  }
```
我们再看 ServiceMethod 的 invoke 方法，实现类是 HttpServiceMethod：
```
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```
在 invoke 方法中创建了 OkHttpCall，并调用了 adapt 方法，实现就在HttpServiceMethod的三个静态内部类CallAdapted、SuspendForResponse、SuspendForBody中。默认情况下回执行 CallAdapted 的 adapt 方法：
```
    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
```
默认情况会执行 DefaultCallAdapterFactory 的 adapt：
```
  @Override
  public Call<Object> adapt(Call<Object> call) {
    return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
  }
```
ExecutorCallbackCall 内部封装了 OkHttpCall 和 MainThreadExecutor：
```
    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override
    public void enqueue(final Callback<T> callback) {
      Objects.requireNonNull(callback, "callback == null");

      delegate.enqueue(
          new Callback<T>() {
            @Override
            public void onResponse(Call<T> call, final Response<T> response) {
              callbackExecutor.execute(
                  () -> {
                    if (delegate.isCanceled()) {
                      // Emulate OkHttp's behavior of throwing/delivering an IOException on
                      // cancellation.
                      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                      callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                  });
            }

            @Override
            public void onFailure(Call<T> call, final Throwable t) {
              callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
            }
          });
    }
```
获取到 Call 对象后，会调用 Call 的 enqueue 方法执行异步请求，它会调用 OkHttpCall 的 enqueue 方法：
```
    call.enqueue(
        new okhttp3.Callback() {
          @Override
          public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
            Response<T> response;
            try {
                // 解析 Response
              response = parseResponse(rawResponse);
            } catch (Throwable e) {
              throwIfFatal(e);
              callFailure(e);
              return;
            }

            try {
                //通过回调调用 DefaultCallAdapterFactory 中 enqueue 方法回调接口的 onResponse 实现，它会通过 MainThreadExecutor 将解析后的数据回调到主线程
                //也就是说，我们在使用 call.enqueue(...) 时其中的 onResponse 和 onFailure 方法在主线程执行
              callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }

          @Override
          public void onFailure(okhttp3.Call call, IOException e) {
            callFailure(e);
          }

          private void callFailure(Throwable e) {
            try {
              callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }
        });
```
调用 parseResponse 解析返回数据并回调 onResponse。在 parseResponse 方法中会调用 Converter 的 convert 方法转换返回数据，也就是 GsonResponseBodyConverter 的 convert 方法：
```
  @Override
  public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
        throw new JsonIOException("JSON document was not fully consumed.");
      }
      return result;
    } finally {
      value.close();
    }
  }
```
在这里会通过 Gson 将返回数据转换成我们想要的数据类型。

**总结：1、通过 Retrofit.Builder 的 build 方法创建 Retrofit 实例。
2、调用 Retrofit 的 create 方法获取动态代理对象，当调用 ApiService 的方法时，会执行 InvocationHandler 的 invoke 方法，进而执行 loadServiceMethod(method).invoke(args)，loadServiceMethod 从缓存中查找 ServiceMethod 或生成一个新的。invoke 方法会调用 ServiceMethod 的 adapt 方法，对于kotlin的挂起函数，调用 adapt 方法会直接调用 Call 对象相应的扩展函数，扩展函数内部会直接调用它的 enqueue 方法发起异步请求；对于普通方法，invoke方法会创建一个 ExecutorCallbackCall 对象将 OkHttpCall 对象和 MainThreadExecutor 封装到其中，在我们显示调用 Call 的 enqueue 方法时会调用到ExecutorCallbackCall的enqueue方法，它通过调用 OkHttpCall的enqueue 方法发起网络请求，在回调中会在构建 Retrofit 对象时添加的 Converter 中寻找相应的转换器对返回的数据进行转换。而在 ExecutorCallbackCall 的 enqueue 方法中会通过 MainThreadExecutor 将数据回调到主线程。**
