Okhttp is a high-quality HTTP client with easy-used API, it is already integrated in Android SDK.
Compared with android raw HTTP interface, Okhttp encapsulate handy API for most use cases.

Because of the rudness design for android raw HTTP, You must write a lot of utility, and hard-code like code, which is ineffective for coding.
Apache HTTP client is widely used before, but now abandoned from Android SDK, and not recommended to use. It's too heavy, moreover bugs found in it.

### OkhttpClient object && timeout

As for Okhttp, OkhttpClient object should be shared, creating a client object for each request wastes resources. Please refer to comments from OkhttpClient.java file.
so below is the skeleton of the wrapper class of OkhttpClient which is shared and set timeout properly.
``` 
public class MyHttpClient {
    /* static shared */
    private static OkhttpClient defaultClient;
    
    public void synchronized init(int connectTimeout, int readTimeout) {
        /* construct the defaultClient object with connect timeout and read timeout. */
	if (defaultClient == null) {
	    defaultClient = new OkhttpClient().builder().readTimeout(readTimeout, TimeUnit.MILLISECONDS).connectTimeout(connectTimeout, TimeUnit.MILLISECONDS).build();
	}        
    }
    
    /* execute http method with default timeout arguments for most cases. */
    public InputStream visit(String url, byte[] data, Map<String, String> headers, HttpMethod method) {
        /* construct Request. */
	Request request = ...
	/* using the default http client to execute the request. 
        for instance using synchronized call: */
	defaultClient.newCall(request).execute();
	...	
    }
	
    /* 
    execute http method with custom timeout arguments,
    for debugging or some special cases using different timeout.
    */
    public InputStream visit(String url, byte[] data, Map<String, String> headers, HttpMethod method, int connectTimeout, int readTimeout) {
        /* customClient is just a reference to defaultClient, but with different timeout attributes,
        so constructing it is fast. */
        OkhttpClient customClient = defaultClient.newBuilder().readTimeout(readTimeout, TimeUnit.MILLISECONDS).connectTimeout(connectTimeout, TimeUnit.MILLISECONDS).build();		 
        /* construct Request. */
        Request request = ...
        /* using the custom http client to execute this request. for instance: */
        customClient.newCall(request).execute();
        ...	
    }	
	
    /* NOT recommended. creating a new http client for each request. */
    public InputStream badVisit(String url, byte[] data, Map<String, String> headers, HttpMethod method) {
        OkhttpClient newClient = new OkhttpClient().builder().build();
	Request request = ...
	newClient.newCall(request).execute();
	...
    }
}
```

### Decompress Okhttp response 
About the Okhttp response, should you check to decompress? there are two solutions you can do, but the first is recommended.

* Do not set the header `Accept-Encoding` with value `gzip` explicitly.
Okhttp interceptor will check the request headers, if not found `Accept-Encoding`, it add this header automatically, as a result, 
it is responsible for decompressing the response stream and remove the `Content-Encoding` and `Content-Length` headers from response.
all these activity are not involved with caller.

 Please refer to the following details(okhttp3/internal/http/BridgeInterceptor.java):
 
```
   public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.  
    
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }
        

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```



* You can set the header `Accept-Encoding` with value `gzip` for request explicitly. as we see above, in this situation, Okhttp will not decompress
response stream and you must check the `Content-Encoding` yourself and decide if decompress. This is helpful and only necessary for using non-gzip compress method which both client and server support.


  
