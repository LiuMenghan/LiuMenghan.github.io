---
title: Grpc详解
---

# RPC框架的选择

常见的RPC框架主要分为轻重两种。较轻的框架一般只负责通信，如rmi、webservice、restful、Thrift、Grpc等。较重的框架一般包括完整的服务发现、负载均衡策略等等如BAT三家的Dubbo、brpc、Tars之类。
	
框架选择时个人认为首先要考虑的是框架的历史和项目的活跃程度。一个历史悠久的活跃项目（大概可以至少保证每两到三个月可以有一次小版本的更新）可以保证各种问题早已暴露并修复，让我们可以更专注于我们自己的项目本身，而不是要担心究竟是我们自己的代码有问题还是框架本身就有问题。
	
重量级RPC框架有一个主要问题就是结构复杂，另外主语言之外的代码质量也不太容易保证。个人认为这些重型RPC框架如果想要成功需要有一个活跃的社区，以及一个活跃的开源管理团队。比如我们项目组试用过腾讯的Tars，C++同学表示没有任何问题，然后JAVA同学表示java版本的各种bug，提了好多Bug修改请求至今还挂在github上，而这个项目已经快两年没更新了。
	
轻量级rpc框架中，restful可以被视作标杆。由于restful基于http协议，天然被各种框架支持，而且非常灵活。restful的缺点有两方面，一是过于灵活，缺少根据协议生成服务端和客户端代码的工具，联调往往要花更多的时间；二是大部分序列化基于json或者xml，相对来讲效率不理想。和restful相比，很多轻量级框架都有这样或者那样的缺点，有的缺少跨语言支持（rmi），有的很繁琐（webservice）。个人认为其中相对理想的是Grpc和Thrift。

## Grpc简介

Protobuf是一种google推出的非常流行的跨语言序列化/反序列化框架。在Protobuf2中就已经出现了用rpc定义服务的概念，但是一直缺少一种流行的rpc框架支持。当Http2推出之后，google将Http2和protobuf3结合，推出了Grpc。Grpc继承了Protobuf和Http2的优点，包括：
* 序列化反序列化性能好
* 强类型支持
* 向前/向后兼容
* 有代码生成机制，而且可以支持多语言
* 长连接、多路复用

同时Grpc还提供了简单地服务发现和负载均衡功能。虽然这并不是Grpc框架的重点，但是开发者可以非常容易的自己扩展Grpc这些功能，实现自己的策略或应用最新的相关方面技术，而不用像重型Rpc框架一样受制于框架本身是否支持。

## Grpc与Thrift对比

Thrift是Facebook推出的一种RPC框架，从性能上来讲远优于Grpc。但是在实际调研时发现有一个很麻烦的问题：Thrift的客户端是**线程不安全**的——这意味着在Spring中无法以单例形式注入到Bean中。解决方案有三种：
1.每次调用创建一个Thrift客户端。这不仅意味着额外的对象创建和垃圾回收开销，而且实际上相当于只使用了短链接，这是一个开发复杂度最低但是从性能上来讲最差的解决方案。
2.利用Pool，稍微复杂一点的解决方案，但是也非常成熟。但是问题在于一来要实现服务发现和负载均衡恐怕需要很多额外开发；二来恐怕要创建Pool数量\*服务端数量个客户端，内存开销会比较大。
3.使用异步框架如Netty，可以成功避免创建过多的客户端，但是仍要自己实现服务发现和负载均衡，相对复杂。实际上Facebook有一个基于Netty的Thrift客户端，叫Nifty，但是快四年没更新了。。。

>相比较而言Grpc就友好多了，本身有简单而且可扩展的服务发现和负载均衡功能，底层基于Netty所以线程安全，在不需要极限压榨性能的情况下是非常好的选择。当然如果需要极限压榨性能Thrift也未必够看。

# Grpc入门

## Grpc服务定义

Grpc中有一个特殊的关键字**stream**，表示可以以流式输入或输出多个protobuf对象。注意只有异步非阻塞的客户端支持以stream形式输入，同步阻塞客户端不支持以stream形式输入。

```
syntax = "proto3";  //GRPC必须使用proto3

option java_multiple_files = true;
option java_package = "cn.lmh.examples.grpc.proto";

service RouteGuide {
  // 输入一个坐标，返回坐标和时间(1:1)
  rpc getPoint(Point) returns (LocationNote) {}
  // 输入一个矩形，以stream形式返回一系列点(1:n)
  rpc listPoints(Rectangle) returns (stream Point) {}
  // 以stream形式输入一系列点，返回点的数量和总共花费的时间(m:1)
  rpc recordRoute(stream Point) returns (RouteSummary) {}
  // 以stream形式输入一系列点，以stream形式返回已输入点的数量和总共花费的时间(m:n)
  rpc getPointStream(stream Point) returns (stream RouteSummary) {}
}

message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
message Rectangle {
  Point lo = 1;
  Point hi = 2;
}
message LocationNote {
  Point location = 1;
  int64 timestamp = 2;
}
message RouteSummary {
  int32 point_count = 1;
  int64 elapsed_time = 2;
}
```

## 依赖和代码生成

由于protoc的Grpc插件需要自己编译，而且存在环境问题。推荐使用gradle或者maven的protobuf插件。实例项目使用了gradle，根目录build.gradle配置如下:

```
plugins {
	id 'java'
	id 'idea'
	id 'wrapper'
}

ext {
	groupId = 'cn.lmh.leviathan'
	proto = [
		version : "3.9.0",
		"grpc" :[
			version : "1.23.0"
		]
	]
}

allprojects{
	apply plugin: 'java'
	apply plugin: 'idea'

	sourceCompatibility=JavaVersion.VERSION_1_8
	targetCompatibility=JavaVersion.VERSION_1_8

	project.group = 'cn.lmh.examples'

	compileJava.options.encoding = 'UTF-8'
}

subprojects{
	repositories {
		mavenCentral()
		mavenLocal();
	};
	configurations {
		compile
	}

	dependencies {
		compile "io.grpc:grpc-netty-shaded:${proto.grpc.version}"
		compile "io.grpc:grpc-protobuf:${proto.grpc.version}"
		compile "io.grpc:grpc-stub:${proto.grpc.version}"

		testCompile group: 'junit', name: 'junit', version: '4.12'
	}
}
```

子项目build.gradle如下：

```
plugins{
    id 'com.google.protobuf' version '0.8.10'   //引入protobuf插件
}

sourceSets{
    main{
        proto {
            srcDir 'src/main/proto' //指定.proto文件所在的位置
        }
    }
}

protobuf {
    generatedFilesBaseDir = "$projectDir/src"   //生成文件的根目录

    protoc {
        artifact = "com.google.protobuf:protoc:${proto.version}"    //protoc的版本
    }

    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${proto.grpc.version}" //grpc的版本
        }
    }

    generateProtoTasks {
        all()*.plugins {
            grpc {
                outputSubDir = "java"   //grpc生成文件的子目录
            }
        }
    }
}
```

我们的入门子项目名称叫做starter，配置好build.gradle之后，执行gradlew :starter:generateProto就可以在src/main/java下生成对应的文件：

![grpc生成的目录结构](grpc-in-depth/starter-genenate-grpc.png)
## 服务端

>无论客户端以异步非阻塞还是同步阻塞形式调用，Grpc服务端的返回都是以异步形式返回。对于异步的请求或者返回，都需要实现Grpc的`io.grpc.stub.StreamObserver`接口。接口有三个方法：
+ `onNext`:表示接收/发送一个对象
+ `onError`:处理异常
+ `onCompleted`:表示请求或返回结束

>当请求来到服务端端时，会异步调用requestObserver的onNext方法，直到结束时调用requestObserver的onCompleted方法；服务端调用responseObserver的onNext把结果返回给客户端，直到调用responseObserver的onCompleted方法通知客户端返回结束。服务端代码如下：
```
import cn.lmh.examples.grpc.proto.*;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;
import java.io.IOException;
import java.util.concurrent.atomic.AtomicInteger;

public class RouteGuideServer {
    private final int port;
    private final Server server;

    public RouteGuideServer(int port) throws IOException {
        this.port = port;
        server = ServerBuilder.forPort(port).addService(new RouteGuideService())
                .build();
    }

    /** Start server. */
    public void start() throws IOException {
        server.start();
        System.out.println("Server started, listening on " + port);
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                RouteGuideServer.this.stop();
            }
        });
    }

    /** Stop server */
    public void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    /**
     * Await termination on the main thread since the grpc library uses daemon threads.
     */
    private void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }

    public static void main(String[] args) throws Exception {
        RouteGuideServer server = new RouteGuideServer(8980);
        server.start();
        server.blockUntilShutdown();
    }

    private static class RouteGuideService extends RouteGuideGrpc.RouteGuideImplBase {
        @Override
        public void getPoint(Point request, StreamObserver<LocationNote> responseObserver) {
            LocationNote value = LocationNote.newBuilder().setLocation(request).setTimestamp(System.nanoTime()).build();
            responseObserver.onNext(value);
            responseObserver.onCompleted();
        }

        @Override
        public void listPoints(Rectangle request, StreamObserver<Point> responseObserver) {
            int left = Math.min(request.getLo().getLongitude(), request.getHi().getLongitude());
            int right = Math.max(request.getLo().getLongitude(), request.getHi().getLongitude());
            int top = Math.max(request.getLo().getLatitude(), request.getHi().getLatitude());
            int bottom = Math.max(request.getLo().getLatitude(), request.getHi().getLatitude());
            for(int x = left; x <= right; x++){
                for(int y = top; y >= bottom; y--){
                    Point point = Point.newBuilder().setLongitude(x).setLatitude(y).build();
                    responseObserver.onNext(point);
                }
            }
            responseObserver.onCompleted();
        }

        @Override
        public StreamObserver<Point> recordRoute(StreamObserver<RouteSummary> responseObserver) {
            return new StreamObserver<Point>(){ //返回的是requestObserver
                AtomicInteger pointCount = new AtomicInteger(0);
                final long startTime = System.nanoTime();

                @Override
                public void onNext(Point value) {
                    int count = pointCount.incrementAndGet();
                }

                @Override
                public void onError(Throwable t) {}

                @Override
                public void onCompleted() {
                    RouteSummary result = RouteSummary.newBuilder().setElapsedTime(System.nanoTime() - startTime).setPointCount(pointCount.get()).build();
                    responseObserver.onNext(result);
                    responseObserver.onCompleted();
                }
            };
        }

        @Override
        public StreamObserver<Point> getPointStream(StreamObserver<RouteSummary> responseObserver) {
            return new StreamObserver<Point>(){ //返回的是requestObserver
                AtomicInteger pointCount = new AtomicInteger(0);
                final long startTime = System.nanoTime();

                @Override
                public void onNext(Point value) {
                    int count = pointCount.incrementAndGet();
                    RouteSummary result = RouteSummary.newBuilder().setElapsedTime(System.nanoTime() - startTime).setPointCount(count).build();
                    responseObserver.onNext(result);
                }

                @Override
                public void onError(Throwable t) {}

                @Override
                public void onCompleted() {
                    responseObserver.onCompleted();
                }
            };
        }
    }
}
```

## 客户端

Grpc的客户端有同步阻塞客户端（blockingStub)和异步非阻塞客户端(Stub）两种。同步客户端使用比较方便，但是性能较低，而且不支持stream形式的request;异步客户端性能较高，支持stream形式的request，但是如果想要以同步方式调用需要额外封装。本文将主要以异步为例。

### 异步转同步

由于Grpc的异步客户端性能较高且功能更完整，所以一般都会采用异步客户端。异步客户端接收到的服务端返回也是以`io.grpc.stub.StreamObserver`形式。由于客户端的调用可能是在异步进程中但更可能是在同步进程中，所以就存在一个如何把Grpc异步返回转为同步返回的过程。

一个比较常见的思路是写一个`io.grpc.stub.StreamObserver`实现，里面有一个内置变量保存异步返回的结果，再添加一个阻塞式的`get()`方法，直到返回完成才把所有结果返回。要知道返回是否完成，需要添加一个`Boolean`或者`AtomicBoolean`变量，初始化为`false`，调用`responseObserver.onCompleted()`方法时设置为`true`，这样就可以通过这个变量判断返回是否完成。阻塞`get()`方法最常见的思路是`get()`写一个`while`循环，直到变量值改为`true`才退出循环并返回结果。这种方式的优点是简单直接，任何语言都可以简单实现，缺点是由于使用循环可能CPU占用较高。另一个思路是在仅限于java之类多线程比较完善的语言，如在发现返回没有结束时，将线程挂起，`调用responseObserver.onCompleted()`方法在修改变量的同时唤醒线程。代码如下

```
import io.grpc.stub.StreamObserver;
import java.util.ArrayList;
import java.util.List;
public class CallableStreamObserver<T> implements StreamObserver<T> {
    List<T> values = new ArrayList<T>();
    boolean isCompleted = false;
    Throwable t = null;
    @Override
    public void onNext(T value) {
        this.values.add(value);
    }
    @Override
    public void onError(Throwable t) {
        this.isCompleted = true;
        this.t=t;
        notifyAll();
    }
    @Override
    public synchronized void onCompleted() {
        this.isCompleted = true;
        notifyAll();
    }
    public List<T> get() throws Throwable{
        while(!this.isCompleted){
            synchronized (this) {
                this.wait(60 * 1000);
            }
        }
        if(null != t){
            throw this.t;
        }else{
            return this.values;
        }
    }
}
```

### 客户端代码
```
import cn.lmh.examples.grpc.CallableStreamObserver;
import cn.lmh.examples.grpc.proto.*;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.stub.StreamObserver;

import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class RouteGuideClient {

    private final ManagedChannel channel;
    private final RouteGuideGrpc.RouteGuideBlockingStub blockingStub;
    private final RouteGuideGrpc.RouteGuideStub asyncStub;

    public RouteGuideClient(String host, int port) {
        this(ManagedChannelBuilder.forAddress(host, port).usePlaintext());
    }

    /** Construct client for accessing RouteGuide server using the existing channel. */
    public RouteGuideClient(ManagedChannelBuilder<?> channelBuilder) {
        channel = channelBuilder.build();
        blockingStub = RouteGuideGrpc.newBlockingStub(channel);
        asyncStub = RouteGuideGrpc.newStub(channel);
    }

    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

    public LocationNote getPoint(int lo, int lt, boolean blocking) throws Throwable {
        Point point = Point.newBuilder().setLongitude(lo).setLatitude(lt).build();
        if(blocking) {
            return blockingStub.getPoint(point);
        }else{
            CallableStreamObserver<LocationNote> responseObserver = new CallableStreamObserver<LocationNote>();
            asyncStub.getPoint(point, responseObserver);
            return responseObserver.get().get(0);
        }
    }

    public Iterator<Point> listPoints(int left, int top, int right, int bottom, boolean blocking) throws Throwable {
        Point hi = Point.newBuilder().setLongitude(left).setLatitude(top).build();
        Point lo = Point.newBuilder().setLongitude(right).setLatitude(bottom).build();
        Rectangle rec = Rectangle.newBuilder().setHi(hi).setLo(lo).build();
        if(blocking){
            return blockingStub.listPoints(rec);
        }else{
            CallableStreamObserver<Point> responseObserver = new CallableStreamObserver<Point>();
            asyncStub.listPoints(rec, responseObserver);
            return responseObserver.get().iterator();
        }
    }

    public RouteSummary recordRoute(Collection<Point> points) throws Throwable {
        CallableStreamObserver<RouteSummary> responseObserver = new CallableStreamObserver<RouteSummary>();
        StreamObserver<Point> requestObserver = asyncStub.recordRoute(responseObserver);
        points.stream().parallel().forEach(p -> requestObserver.onNext(p));
        requestObserver.onCompleted();
        return responseObserver.get().get(0);

    }

    public List<RouteSummary> getPointStream(Collection<Point> points) throws Throwable {
        CallableStreamObserver<RouteSummary> responseObserver = new CallableStreamObserver<RouteSummary>();
        StreamObserver<Point> requestObserver = asyncStub.getPointStream(responseObserver);
        points.stream().parallel().forEach(p -> requestObserver.onNext(p));
        requestObserver.onCompleted();
        return responseObserver.get();
    }
}
```

# GRPC代码详解

鉴于整个Grpc的代码还是比较多的，本文将专注于以下问题：
1.究竟有哪些可用的Server可以提供服务？
2.究竟应该在这些可用的服务端中选择哪个Server？
3.Client与Server之间是如何通信的？

## 发现服务端

前文的入门示例客户端中直接写了host和port，那么不免让人产生疑问，如果有多个可以提供服务的Server如何处理。如前文的客户端示例所示，客户端的创建依赖于`io.grpc.ManagedChannel`，而`io.grpc.ManagedChannel`依赖于`io.grpc.ManagedChannelBuilder`。在`io.grpc.ManagedChannelBuilder`中有两个方法,`forTarget`和`nameResolverFactory`，二者配合使用可以发现服务端。

### `forTarget`方法
感觉Grpc这里设计比较繁琐。`forTarget`方法的实际功能**是把一个url赋值给`io.grpc.ManagedChannelBuilder`内部的`target`变量**，但是过程比较绕。
```
/**
   * Creates a channel with a target string, which can be either a valid {@link
   * NameResolver}-compliant URI, or an authority string.
   *
   */
  public static ManagedChannelBuilder<?> forTarget(String target) {
    return ManagedChannelProvider.provider().builderForTarget(target);
  }
```
`io.grpc.ManagedChannelProvider.provider()`会返回一个`io.grpc.ManagedChannelProvider`实现，然后会调用这个实现的`buildForTarget`方法。有哪些`io.grpc.ManagedChannelProvider`实现是在`io.grpc.ManagedChannelProvider`中以硬编码实现的。
```
private static final class HardcodedClasses implements Iterable<Class<?>> {
    @Override
    public Iterator<Class<?>> iterator() {
      List<Class<?>> list = new ArrayList<>();
      try {
        list.add(Class.forName("io.grpc.okhttp.OkHttpChannelProvider"));
      } catch (ClassNotFoundException ex) {
        // ignore
      }
      try {
        list.add(Class.forName("io.grpc.netty.NettyChannelProvider"));
      } catch (ClassNotFoundException ex) {
        // ignore
      }
      return list.iterator();
    }
  }
```
实际上就根据依赖的jar包不同就只有两个实现，一个netty的，一个okhttp的。因为我们配置里配置的是netty实现,所以就只分析netty实现，okhttp里的实现也是类似的。`io.grpc.netty.NettyChannelProvider`的buildForTarget方法调用的是`io.grpc.netty.NettyChannelBuilder`的`forTarget`方法。
```
public NettyChannelBuilder builderForTarget(String target) {
    return NettyChannelBuilder.forTarget(target);
}
```
而`io.grpc.netty.NettyChannelBuilder`继承自`io.grpc.internal.AbstractManagedChannelImplBuilder`，`forTarget`方法实际上调用了父类的构造函数。
```
@CheckReturnValue
  NettyChannelBuilder(String target) {
    super(target);
  }

@CheckReturnValue
  public static NettyChannelBuilder forTarget(String target) {
    return new NettyChannelBuilder(target);
  }
```
`io.grpc.internal.AbstractManagedChannelImplBuilder`的构造函数主要作用是把参数赋值给`target`变量。
```
protected AbstractManagedChannelImplBuilder(String target) {
    this.target = Preconditions.checkNotNull(target, "target");
    this.directServerAddress = null;
  }
```
### `nameResolverFactory`方法
这个方法的实现也在`io.grpc.internal.AbstractManagedChannelImplBuilder`中，如果用户有自己的`io.grpc.NameResolve.Factory`实现的话会使用用户的`io.grpc.NameResolve.Factroy`实现，否则会使用`io.grpc.NameResolverRegistry`的默认实现。
```
final NameResolverRegistry nameResolverRegistry = NameResolverRegistry.getDefaultRegistry();

@Override
  public final T nameResolverFactory(NameResolver.Factory resolverFactory) {
    Preconditions.checkState(directServerAddress == null,
        "directServerAddress is set (%s), which forbids the use of NameResolverFactory",
        directServerAddress);
    if (resolverFactory != null) {
      this.nameResolverFactory = resolverFactory;
    } else {
      this.nameResolverFactory = nameResolverRegistry.asFactory();
    }
    return thisT();
  }
```
`io.grpc.NameResolverRegistry`会通过硬编码加载`io.grpc.NameResolverProvider`实现，并创建一个与之有关的`io.grpc.NameResolve.Factory`的实现。目前硬编码加载的`io.grpc.NameResolverProvider`实现只有`io.grpc.internal.DnsNameResolverProvider`一种。
```
private static NameResolverRegistry instance;
private final NameResolver.Factory factory = new NameResolverFactory();
@GuardedBy("this")
private final LinkedHashSet<NameResolverProvider> allProviders = new LinkedHashSet<>();

private synchronized void addProvider(NameResolverProvider provider) {
    checkArgument(provider.isAvailable(), "isAvailable() returned false");
    allProviders.add(provider);
  }
/**
   * Returns the default registry that loads providers via the Java service loader mechanism.
   */
  public static synchronized NameResolverRegistry getDefaultRegistry() {
    if (instance == null) {
      List<NameResolverProvider> providerList = ServiceProviders.loadAll(
          NameResolverProvider.class,
          getHardCodedClasses(),
          NameResolverProvider.class.getClassLoader(),
          new NameResolverPriorityAccessor());
      if (providerList.isEmpty()) {
        logger.warning("No NameResolverProviders found via ServiceLoader, including for DNS. This "
            + "is probably due to a broken build. If using ProGuard, check your configuration");
      }
      instance = new NameResolverRegistry();
      for (NameResolverProvider provider : providerList) {
        logger.fine("Service loader found " + provider);
        if (provider.isAvailable()) {
          instance.addProvider(provider);
        }
      }
      instance.refreshProviders();
    }
    return instance;
  }
  

  public NameResolver.Factory asFactory() {
    return factory;
  }

  @VisibleForTesting
  static List<Class<?>> getHardCodedClasses() {
    // Class.forName(String) is used to remove the need for ProGuard configuration. Note that
    // ProGuard does not detect usages of Class.forName(String, boolean, ClassLoader):
    // https://sourceforge.net/p/proguard/bugs/418/
    ArrayList<Class<?>> list = new ArrayList<>();
    try {
      list.add(Class.forName("io.grpc.internal.DnsNameResolverProvider"));
    } catch (ClassNotFoundException e) {
      logger.log(Level.FINE, "Unable to find DNS NameResolver", e);
    }
    return Collections.unmodifiableList(list);
  }

  private final class NameResolverFactory extends NameResolver.Factory {
    @Override
    @Nullable
    public NameResolver newNameResolver(URI targetUri, NameResolver.Args args) {
      List<NameResolverProvider> providers = providers();
      for (NameResolverProvider provider : providers) {
        NameResolver resolver = provider.newNameResolver(targetUri, args);
        if (resolver != null) {
          return resolver;
        }
      }
      return null;
    }

    @Override
    public String getDefaultScheme() {
      List<NameResolverProvider> providers = providers();
      if (providers.isEmpty()) {
        return "unknown";
      }
      return providers.get(0).getDefaultScheme();
    }
  }
```
### `ManagedChannel.builder()`方法

### 自定义`NameResolver.Factory`实现

### 服务健康状态检查

## 负载均衡

## 通信
