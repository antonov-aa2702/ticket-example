Hey team. I am creating an application that needs to call another microservice,
and add the minimum possible amount of latency doing so.
So for that I wanted to give netty and the Spring webclient a try.

On the first try i saw that calling the first time the netty version was much slower than
the simple curl based ones. 
I added warmup functionality but it didn't help much.
Is there anything else I can do to optimize the first call performance?

## Expected Behavior
The difference between the first and second request is not that big.

## Actual Behavior

**Config**
```java
WebClient webClient(SomeProperties properties) throws Exception {
    log.info("Initializing WebClient Bean");
    var sslContext = getSslContext();
    HttpClient httpClient = HttpClient.create()
            .metrics(true, Function.identity())
            .secure(sslContextSpec -> sslContextSpec.sslContext(sslContext));
    httpClient.warmup().block();
    ClientHttpConnector connector = new ReactorClientHttpConnector(httpClient);
    var webClient = WebClient.builder()
            .baseUrl(properties.getUrl().getBase())
            .clientConnector(connector)
            .build();
    log.info("WebClient initialized");
    return webClient;
}
```

**Request**
```java
    public byte[] sendSomeRequest(String message, String user, String password) {
    log.trace("Calling Some service for signing and encryption. Message: {}", message);
    final var messageInBase64 = Base64.getEncoder().encodeToString(message.getBytes(StandardCharsets.UTF_8));
    final var body = new SomeRequest().setPayload(messageInBase64);
    final var response = webClient.post()
            .uri(properties.getUrl().getSignAndEncrypt())
            .bodyValue(body)
            .headers(h -> h.setBasicAuth(user, password))
            .retrieve()
            .bodyToMono(SomeResponse.class)
            .doOnError(e -> {
                log.error("Error calling Some service for signing and encryption: " + e.getMessage(), e);
                throw new CryptoException(e.getMessage());
            })
            .block();
    final var encryptedMessageInBase64 = Objects.requireNonNull(response).getEncryptedMessage();
    final var encryptedMessage = Base64.getDecoder().decode(encryptedMessageInBase64);
    log.info("Successful Some Service signing and encryption, url: {}",
            properties.getUrl().getBase() + properties.getUrl().getSignAndEncrypt());
    return encryptedMessage;
}
```

**warmup**
```java
2025-11-20 07:15:46,801 INFO  com.example.demo :  Initializing WebClient Bean
2025-11-20 07:15:47,059 DEBUG reactor.netty.tcp.TcpResources :  [http] resources will use the default LoopResources: DefaultLoopResources {prefix=reactor-http, daemon=true, selectCount=40, workerCount=40}
2025-11-20 07:15:47,060 DEBUG reactor.netty.tcp.TcpResources :  [http] resources will use the default ConnectionProvider: reactor.netty.resources.DefaultPooledConnectionProvider@7e1953f7
2025-11-20 07:15:47,068 DEBUG reactor.netty.resources.DefaultLoopIOUring :  Default io_uring support : false
2025-11-20 07:15:47,173 DEBUG reactor.netty.resources.DefaultLoopEpoll :  Default Epoll support : true
2025-11-20 07:15:47,628 INFO  com.example.demo :  WebClient initialized
```

**first request ~2.2-2.8s average**

```java
2025-11-20 07:16:40,266 TRACE com.example.demo : Calling Some service for signing and encryption ...
2025-11-20 07:16:40,602 DEBUG reactor.netty.resources.PooledConnectionProvider : Creating a new [http] client pool [PoolFactory{evictionInterval=PT0S, leasingStrategy=fifo, maxConnections=500, maxIdleTime=-1, maxLifeTime=-1, metricsEnabled=false, pendingAcquireMaxCount=1000, pendingAcquireTimeout=45000}] for [some.host.example:4444]
        2025-11-20 07:16:41,007 DEBUG reactor.netty.resources.PooledConnectionProvider :  [6454b084] Created a new pooled channel, now: 0 active connections, 0 inactive connections and 0 pending acquire requests.
2025-11-20 07:16:41,098 DEBUG reactor.netty.tcp.SslProvider :  [6454b084] SSL enabled using engine sun.security.ssl.SSLEngineImpl@41fe1a91 and SNI some.host.example:4444
        2025-11-20 07:16:41,200 DEBUG reactor.netty.transport.TransportConfig :  [6454b084] Initialized pipeline DefaultChannelPipeline{(reactor.left.sslHandler = io.netty.handler.ssl.SslHandler), (reactor.left.sslReader = reactor.netty.tcp.SslProvider$SslReadHandler), (reactor.left.httpCodec = io.netty.handler.codec.http.HttpClientCodec), (reactor.right.reactiveBridge = reactor.netty.channel.ChannelOperationsHandler)}
2025-11-20 07:16:41,493 DEBUG reactor.netty.transport.TransportConnector :  [6454b084] Connecting to [some.host.example/40.40.40.40:4444].
        2025-11-20 07:16:41,506 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Registering pool release on close event for channel
2025-11-20 07:16:41,507 DEBUG reactor.netty.resources.PooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Channel connected, now: 1 active connections, 0 inactive connections and 0 pending acquire requests.
2025-11-20 07:16:41,564 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}, [connected])
        2025-11-20 07:16:41,616 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(GET{uri=null, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [configured])
        2025-11-20 07:16:41,625 DEBUG reactor.netty.http.client.HttpClientConnect :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Handler is being applied: {uri=https://some.host.example:4444someApi, method=POST}
        2025-11-20 07:16:41,648 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [request_prepared])
        2025-11-20 07:16:41,867 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [request_sent])
        2025-11-20 07:16:41,918 DEBUG reactor.netty.http.client.HttpClientOperations :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Received response (auto-read:false) : RESPONSE(decodeResult: success, version: HTTP/1.1)
HTTP/1.1 200
X-Content-Type-Options: <filtered>
X-XSS-Protection: <filtered>
Cache-Control: <filtered>
Pragma: <filtered>
Expires: <filtered>
Strict-Transport-Security: <filtered>
X-Frame-Options: <filtered>
Content-Type: <filtered>
Transfer-Encoding: <filtered>
Date: <filtered>
Server: <filtered>
2025-11-20 07:16:41,918 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [response_received])
        2025-11-20 07:16:42,078 DEBUG reactor.netty.channel.FluxReceive :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] FluxReceive{pending=0, cancelled=false, inboundDone=false, inboundError=null}: subscribing inbound receiver
2025-11-20 07:16:42,095 DEBUG reactor.netty.http.client.HttpClientOperations :  [6454b084-1, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Received last HTTP packet
2025-11-20 07:16:42,122 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [response_completed])
        2025-11-20 07:16:42,122 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444]}}, [disconnecting])
        2025-11-20 07:16:42,123 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Releasing channel
2025-11-20 07:16:42,135 DEBUG reactor.netty.resources.PooledConnectionProvider :  [6454b084, L:/30.30.30.30:3333 - R:some.host.example/40.40.40.40:4444] Channel cleaned, now: 0 active connections, 1 inactive connections and 0 pending acquire requests.
```

**second request ~ 0.20.5s average**
```java
2025-11-20 07:18:51,267 TRACE com.example.demo : Calling Some service for signing and encryption ...
2025-11-20 07:18:51,294 DEBUG reactor.netty.resources.PooledConnectionProvider :  [32068e6f] Created a new pooled channel, now: 0 active connections, 0 inactive connections and 0 pending acquire requests.
2025-11-20 07:18:51,296 DEBUG reactor.netty.tcp.SslProvider :  [32068e6f] SSL enabled using engine sun.security.ssl.SSLEngineImpl@123bea09 and SNI some.host.example:4444
        2025-11-20 07:18:51,298 DEBUG reactor.netty.transport.TransportConfig :  [32068e6f] Initialized pipeline DefaultChannelPipeline{(reactor.left.sslHandler = io.netty.handler.ssl.SslHandler), (reactor.left.sslReader = reactor.netty.tcp.SslProvider$SslReadHandler), (reactor.left.httpCodec = io.netty.handler.codec.http.HttpClientCodec), (reactor.right.reactiveBridge = reactor.netty.channel.ChannelOperationsHandler)}
2025-11-20 07:18:51,337 DEBUG reactor.netty.transport.TransportConnector :  [32068e6f] Connecting to [some.host.example/40.40.40.40:4444].
        2025-11-20 07:18:51,339 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Registering pool release on close event for channel
2025-11-20 07:18:51,340 DEBUG reactor.netty.resources.PooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Channel connected, now: 1 active connections, 0 inactive connections and 0 pending acquire requests.
2025-11-20 07:18:51,416 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}, [connected])
        2025-11-20 07:18:51,416 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(GET{uri=null, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [configured])
        2025-11-20 07:18:51,417 DEBUG reactor.netty.http.client.HttpClientConnect :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Handler is being applied: {uri=https://some.host.example:4444someApi, method=POST}
        2025-11-20 07:18:51,418 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [request_prepared])
        2025-11-20 07:18:51,426 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [request_sent])
        2025-11-20 07:18:51,440 DEBUG reactor.netty.http.client.HttpClientOperations :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Received response (auto-read:false) : RESPONSE(decodeResult: success, version: HTTP/1.1)
HTTP/1.1 200
X-Content-Type-Options: <filtered>
X-XSS-Protection: <filtered>
Cache-Control: <filtered>
Pragma: <filtered>
Expires: <filtered>
Strict-Transport-Security: <filtered>
X-Frame-Options: <filtered>
Content-Type: <filtered>
Transfer-Encoding: <filtered>
Date: <filtered>
Server: <filtered>
2025-11-20 07:18:51,441 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [response_received])
        2025-11-20 07:18:51,443 DEBUG reactor.netty.channel.FluxReceive :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] FluxReceive{pending=0, cancelled=false, inboundDone=false, inboundError=null}: subscribing inbound receiver
2025-11-20 07:18:51,444 DEBUG reactor.netty.http.client.HttpClientOperations :  [32068e6f-1, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Received last HTTP packet
2025-11-20 07:18:51,445 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [response_completed])
        2025-11-20 07:18:51,446 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] onStateChange(POST{uri=someApi, connection=PooledConnection{channel=[id: 0x32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444]}}, [disconnecting])
        2025-11-20 07:18:51,446 DEBUG reactor.netty.resources.DefaultPooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Releasing channel
2025-11-20 07:18:51,446 DEBUG reactor.netty.resources.PooledConnectionProvider :  [32068e6f, L:/30.30.30.30:42604 - R:some.host.example/40.40.40.40:4444] Channel cleaned, now: 0 active connections, 1 inactive connections and 0 pending acquire requests.
```

**curl request**
```java
curl -k -s -o /dev/null -w "namelookup:%{time_namelookup} connect:%{time_connect} appconnect:%{time_appconnect} starttransfer:%{time_starttransfer} total:%{time_total}\n" "https://some.host.example:443/someApi"
```

**curl response**
```java
namelookup:0.005330 connect:0.008379 appconnect:0.033125 starttransfer:0.065702 total:0.066037  
```

**prometheus-metrics**
```java
reactor_netty_http_client_connect_time_seconds_count{remote_address="some.host.example:4444",status="SUCCESS",} 1.0
reactor_netty_http_client_connect_time_seconds_sum{remote_address="some.host.example:4444",status="SUCCESS",} 0.018151858

reactor_netty_http_client_address_resolver_seconds_count{remote_address="some.host.example:4444",status="SUCCESS",} 1.0
reactor_netty_http_client_address_resolver_seconds_max{remote_address="some.host.example:4444",status="SUCCESS",} 0.390862126
```

## Steps to Reproduce
We cannot reproduce this in our test environment.

## Your Environment
* Reactor version(s) used: 1.0.24
* Spring Boot: 2.7.5 
* JVM version (`java -version`): openjdk-alpine:11.0.26-jdk
* OS and version (eg. `uname -a`): 6.8.0-79-generic #79-Ubuntu SMP PREEMPT_DYNAMIC Tue Aug 12 14:42:46 UTC 2025 x86_64 Linux
