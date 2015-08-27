I’m currently implementing a strangler service (http://www.martinfowler.com/bliki/StranglerApplication.html) in order to kill one of our old APIs. The plan is to point the old API host to a Finagle server, with a service that dispatches every request to the old API (which will respond to a new host name).

```
case class HttpRequestDispatcher(val host: String, port: Int) 
	extends Service[HttpRequest, HttpResponse] {
  
  private val httpClient = new MyHttpClient(s"$host:$port")
    .releaseOnShutdown()

  def apply(originalRequest: HttpRequest) = {
    val originalHeaders = originalRequest.headerMap.toList

    val request = httpClient
      .uri(originalRequest.uri)
      .headers(originalHeaders)

    originalRequest.method match {
      case Get    => request.get
      case Head   => request.head
      case Delete => request.delete
      case Put    => request.put(originalRequest.contentString)
      case Post   => request.post(originalRequest.contentString)
      case _      => MethodNotAllowed().toFuture
    }
  }
}

object OldApiDispatcher {
  def apply() =
    HttpRequestDispatcher(host = "old.api.host.com", port = 80)
}

object OldApiHandler extends TwitterServer {
  val server = Httpx.serve("9991", OldApiDispatcher())
  onExit { server.close() }
}
```

Everything seems to be working, it's dispatching for all kinds of HTTP methods. The tricky part is when I start to strangle endpoints. By strangling endpoints I mean handle them on the new API (Finch + Finagle) and have a translation layer that translates new API contract to old API contract, so I don't break up any clients. So I created ```HandledEndpoints``` object.

```
object HandledEndpoints extends Endpoint[HttpRequest, HttpResponse] {
  def route = {
    case Get -> Root / "ws" / "user" / Long(id) ~ "json" => GetUser(id)
  }
}
```



