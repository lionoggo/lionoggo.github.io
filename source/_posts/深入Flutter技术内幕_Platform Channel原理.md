title: 深入Flutter技术内幕:Platform Channel设计与实现
date: 2019/2/9 21:20:50
toc  : true
tags: [Flutter,Platform Channel,Dart,Android]
categories: technology
description: 深入分析Flutter跨平台通信机制,掌握MethodChannel工作基本流程.

-----

# Platform Channel简介

在Flutter中,提供了三种Platform Channel用来支持和平台之间数据的传递:

- BasicMessageChannel: 支持字符串和半结构化的数据传递
- MethodChannel: 支持方法调用,既可以从Flutter发平台发起方法调用,也可以从平台代码向Flutter发起调用
- EventChannel: 支持数据流通信

这三种Platform Channel分别用不同的作用,但在设计上大同小异,他们都有以下三个成员变量:

- name:表示Channel名字,每个Channel使用唯一的name作为其唯一标志
- messager:信使,是消息的发送和接受工具
- codec: 表示消息的编解码器,目前有MethodCodec和MessageCodec两种类型

## Platform Channel基本结构

为了对这三种Platform Channel有个比较感性的认识,通过以下简化过的代码来熟悉它们的结构.比较熟悉的同学可以直接跳过此章节.

### BasicMessageChannel

```java
public final class BasicMessageChannel<T> {
    private final BinaryMessenger messenger;
    private final String name;
    private final MessageCodec<T> codec;
    
    public void setMessageHandler(final MessageHandler<T> handler) {
        messenger.setMessageHandler(name,
            handler == null ? null : new IncomingMessageHandler(handler));
    }
    
    public interface MessageHandler<T> {
        void onMessage(T message, Reply<T> reply);
    }
    
    private final class IncomingMessageHandler implements BinaryMessageHandler {
        private final MessageHandler<T> handler;

        private IncomingMessageHandler(MessageHandler<T> handler) {
            this.handler = handler;
        }

        @Override
        public void onMessage(ByteBuffer message, final BinaryReply callback) {
            ......
            handler.onMessage();
            ......
        }
    }         
}
```

### MethodChannel

```java
public final class MethodChannel {
    private final BinaryMessenger messenger;
    private final String name;
    private final MethodCodec codec;
    
    ......
    
    public void setMethodCallHandler(final @Nullable MethodCallHandler handler) {
        messenger.setMessageHandler(name,
            handler == null ? null : new IncomingMethodCallHandler(handler));
    }
    
    public interface Result {
        void success(@Nullable Object result);
        void error(String errorCode, @Nullable String errorMessage, @Nullable Object errorDetails);
        void notImplemented();
    }

    public interface MethodCallHandler {
        void onMethodCall(MethodCall call, Result result);
    }
    
    private final class IncomingMethodCallHandler implements BinaryMessageHandler {
        private final MethodCallHandler handler;

        IncomingMethodCallHandler(MethodCallHandler handler) {
            this.handler = handler;
        }

        @Override
        public void onMessage(ByteBuffer message, final BinaryReply reply) {
            ......
            handler.onMethodCall();
            ......
        }
    }     
}
```

### EventChannel

```java
public final class EventChannel {
    private final BinaryMessenger messenger;
    private final String name;
    private final MethodCodec codec;
    
    public void setStreamHandler(final StreamHandler handler) {
        messenger.setMessageHandler(name, handler == null ? null : new IncomingStreamRequestHandler(handler));
    }
    
    public interface StreamHandler {
        void onListen(Object arguments, EventSink events);
        void onCancel(Object arguments);
    }
    
    public interface EventSink {
        void success(Object event);
        void error(String errorCode, String errorMessage, Object errorDetails);
        void endOfStream();
    }

    private final class IncomingStreamRequestHandler implements BinaryMessageHandler {
        private final StreamHandler handler;
        private final AtomicReference<EventSink> activeSink = new AtomicReference<>(null);

        IncomingStreamRequestHandler(StreamHandler handler) {
            this.handler = handler;
        }

        @Override
        public void onMessage(ByteBuffer message, final BinaryReply reply) {
            ......
            handler.onCancel();
            ......
            handler.onListen();   
            ......
        }
    }
}
```

## name(Channel名称)

name是用于区分不同Platform Channel的唯一标志.在一个Flutter应用中,通常会存在多个Platform Channel,不同Channel之间通过name那么来区分.比如在使用MethodChannel平台发起方法调用时,需要为MethodChannel指定对应的name.

## messager(信使)

messager也称为信使,通俗来说信使就是现代的快递员,它负责把数据从Flutter搬运到JAndroid/IOS平台,或者从Android/IOS搬运到Flutter).对于Flutter中的三种Channel,尽管各自用途不同,但messager都是BinaryMessager.

当我们创建一个Channel时,并为其设置消息处理器时,最终会为该Channel绑定一个BinaryMessagerHandler.并以Channel的name为key,保存在Map结构中.当接受到发送消息后,会根据消息中携带的channel名称取出对应BinaryMessagerHandler,并交由其处理.在Android平台中,BinaryMessenger是一个接口,其实现类是FlutterNativeView.在后续[MethodChannel](#BinaryMessenger)调用原理中,会进一步分析.

## Codec(编解码器)

在Platform Channel中,messager携带的数据需要在Dart层,Native层以及Android/IOS平台中传输,需要考虑一种与平台无关的数据协议,且又能支持图片/文件等资源,因此官方最终采用了二进制字节流作为数据传输协议:发送方需要把数据编码成二进制数据,接受方再把数据解码成原始数据.而负责编解码操作的就是Codec.

在Flutter中有两种Codec:

- MethodCodec: 用于对MethodCall编解码
- MessageCodec: 用于对Message进行编解码

### MessageCodec

MessageCodec用于二进制数据与基础数据之间的编解码,其中BasicMessageChannel中采用的就是该Codec.以Android平台为例,MessageCodec定义如下:

```java
public interface MessageCodec<T> {
    // 将指定的类型message编码为二进制数据ByteBuffer
    ByteBuffer encodeMessage(T message);
    // 将二进制数据ByteBuffer解码成指定类型
    T decodeMessage(ByteBuffer message);
}
```

MessageCodec被设计为一个泛型接口,用于实现二进制数据ByteBuffer和不同类型数据之间的转换.(在IOS中,可参考FlutterMessageCodec协议,其原理基本一致).

在Flutter中,目前MessageCodec有多种实现:

![image-20190209205941331](https://ws3.sinaimg.cn/large/006tNc79ly1g00gywsky2j32dx0u0wxl.jpg)

#### BinaryCodec

用于二进制数据和二进制数据之间的编解码,在实现上什么也没有做,只是原封不动的将二进制数据返回而已.

#### StringCodec

用于字符串与二进制数据之间的编解码,对于字符串采用UTF-8编码格式.

#### JSONMessageCodec

用于数据类型与二进制数据之间的编解码,支持基础数据类型(boolean,char,double,float,int,long,short,String)以及List,Map.在Android端使用[JSONUtil](https://sourcegraph.com/github.com/flutter/engine@053f7a8/-/blob/shell/platform/android/io/flutter/plugin/common/JSONUtil.java?utm_source=share#L25)和StringCodec作为序列化和反序列化工具.

#### StandardMessageCodec

用于数据类型和二进制数据之间的编解码,它也是BasicMessageChannel中默认使用的编解码器,支持基础数据类型(boolean,char,double,float,int,long,short,String),List,Map以及二进制数据,更多参见:[深入编解码器原理](#深入编解码器原理)

### MethodCodec

MethodCodec用于二进制数据与方法调用(MethodCall)和返回结果之间的编解码.MethodChannel和EventChannel所使用的编解码器均为MethodCodec.以Android平台为例,MethodCodec定义如下:

```java
public interface MethodCodec {
    // 将methodCall编码为二进制ByteBuffer
    ByteBuffer encodeMethodCall(MethodCall methodCall);

    // 将二进制methodCall解码为MethodCall
    MethodCall decodeMethodCall(ByteBuffer methodCall);

    // 将正常响应结果result编码为二进制ByteBuffer
    ByteBuffer encodeSuccessEnvelope(Object result);

    // 将错误响应提示编码为二进制ByteBuffer
    ByteBuffer encodeErrorEnvelope(String errorCode, String errorMessage, Object errorDetails);

    // 将二进制数据ByteBuffer解码成Object
    Object decodeEnvelope(ByteBuffer envelope);
}
```

一个MethodCall对象代表一次从Flutter端发起的方法调用,对于方法调用而言,涉及方法名,方法参数以及方法返回结果,因此和MessageCodec相比,MethodCodec中多了两个处理调用结果的方法:

- 方法调用成功,使用`encodeSuccessEnvelope()`编码result
- 方法调用失败,使用`encodeErrorEnvelope()`编码errorCode,errorMessage,errorDetail

> `decodeEnvelope()`方法则用于解码平台代码调用Dart中方法的结果.比如Android平台通过MethodChannel调用Flutter中的方法,且获取其返回结果

在Flutter中,目前MethodCodec有两种实现:

![image-20190209174504635](https://ws3.sinaimg.cn/large/006tNc79ly1g00bceb3z0j316c0nan01.jpg)

####  JSONMethodCodec

JSONMethodCodec编解码器依赖于JSONMessageCodec.在将MethodCall对象进行编码时,会首先将该对象转成JSON对象:{"method":method,"args":args},比如当前想要调用某个Channel的`setVolum(5)`,其对应的MethodCall被被转成`{"method":"setVolum","args":{"volum":5}}`,接下来使用JSONMessageCodec将其编码为二进制数据.

#### StandardMethodCodec

StandardMethodCodec的编解码器依赖于StandardMessageCodec,它也是MethodCodec的默认实现.当其编码在将MethodCall对象进行编码时,会将MethoCall对象的method和args依次使用StandardMessageCodec进行编码,然后写成二进制数据.

### 深入编解码器原理

在学习编写Platform Channel过程中,Flutter官方介绍了如何借助MethodChannel编写获取Android/IOS平台上的[电量插件](https://flutterchina.club/platform-channels/).在Android平台中电量返回值是java.lang.Integer类型,而IOS平台中电量返回值是NSNumber类型,而在Dart中,该返回值类型是dart语言中的int类型.下面以Android平台为例,通过分析StandardMethodCodec实现来了解其原理.

上面我们已经说过StandardMethodCodec中使用标准的二进制消息编码器StandardMessageCodec,首先来看一下StandardMethodCodec的定义:

```java
public final class StandardMethodCodec implements MethodCodec {
    public static final StandardMethodCodec INSTANCE = new StandardMethodCodec(StandardMessageCodec.INSTANCE);
    private final StandardMessageCodec messageCodec;
    
    public StandardMethodCodec(StandardMessageCodec messageCodec) {
      this.messageCodec = messageCodec;
    }
    
    .......
    
}
```

现在重点分析StandardMessageCodec:

```java
public class StandardMessageCodec implements MessageCodec<Object> {
    public static final StandardMessageCodec INSTANCE = new StandardMessageCodec();
    
    // 字节序判断,LITTLE_ENDIAN为true表示是小端模式,否则为大端模式.在对字节流写入和读取时,
    // 需要根据字节序来决定读取和写入的顺序
    private static final boolean LITTLE_ENDIAN = ByteOrder.nativeOrder() == ByteOrder.LITTLE_ENDIAN;
    private static final Charset UTF8 = Charset.forName("UTF8");
    // 下面14个常量值用来标志不同类型的数据,比如0表示NULL,3表示int类型.在向字节流写入指定类型
    // 的数据时,需要首先写入类型标志,然后紧跟着写入具体的值;在从字节流读取数据时,首先读取类型标
    // 志,然后读取具体的数值.
    private static final byte NULL = 0;
    private static final byte TRUE = 1;
    private static final byte FALSE = 2;
    private static final byte INT = 3;
    private static final byte LONG = 4;
    private static final byte BIGINT = 5;
    private static final byte DOUBLE = 6;
    private static final byte STRING = 7;
    private static final byte BYTE_ARRAY = 8;
    private static final byte INT_ARRAY = 9;
    private static final byte LONG_ARRAY = 10;
    private static final byte DOUBLE_ARRAY = 11;
    private static final byte LIST = 12;
    private static final byte MAP = 13;


    @Override
    public ByteBuffer encodeMessage(Object message) {
        if (message == null) {
            return null;
        }
        final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
        // 将message写入二进制字节流中
        writeValue(stream, message);
        final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
        buffer.put(stream.buffer(), 0, stream.size());
        return buffer;
    }
    
    @Override
    public Object decodeMessage(ByteBuffer message) {
        if (message == null) {
            return null;
        }
        message.order(ByteOrder.nativeOrder());
        // 从二进制字节流message中读取数据
        final Object value = readValue(message);
        if (message.hasRemaining()) {
            throw new IllegalArgumentException("Message corrupted");
        }
        return value;
    }
    
    // 根据数据类型,先向stream中写入类型标志值,及上述提到的14个常量值,然后将具体的
    // value值转成byte继续写入到stream
    protected void writeValue(ByteArrayOutputStream stream, Object value) {
        if (value == null) {
            stream.write(NULL);
        } else if (value == Boolean.TRUE) {
            stream.write(TRUE);
        } else if (value == Boolean.FALSE) {
            stream.write(FALSE);
        } else if (value instanceof Number) {
            if (value instanceof Integer || value instanceof Short || value instanceof Byte) {         // 1.写入类型标志值
                stream.write(INT);
                // value转为byte,继续写入到stream中
                writeInt(stream, ((Number) value).intValue());
            }
            .......
        }else if (value instanceof String) {
            stream.write(STRING);
            writeBytes(stream, ((String) value).getBytes(UTF8));
        }
        .......
    }
    
    // 根据字节序决定写入顺序
    protected static final void writeInt(ByteArrayOutputStream stream, int value) {
        if (LITTLE_ENDIAN) {
            stream.write(value);
            stream.write(value >>> 8);
            stream.write(value >>> 16);
            stream.write(value >>> 24);
        } else {
            stream.write(value >>> 24);
            stream.write(value >>> 16);
            stream.write(value >>> 8);
            stream.write(value);
        }
    }
    
    // writeValue()方法反向过程,原理一致
    protected final Object readValue(ByteBuffer buffer) {
        .......
    }
    
    static final class ExposedByteArrayOutputStream extends ByteArrayOutputStream {
        byte[] buffer() {
            return buf;
        }
    }
}
```

在StandardMessageCodec中最重要的两个方法是`writeValue()`和`readValue()`.前者用于将value值写入到字节输出流ByteArrayOutputStream中,后者从字节缓冲数组中读取.在Android返回电量的过程中,假设电量值为100,该值转换成二进制数据流程为:首先向字节流stream中写入表示int类型的标志值3,再将100转为4个byte,继续写入到字节流stream中.当Dart中接受到该二进制数据后,先读取第一个byte值,根据此值得知后面需要读取一个int类型的数据,随后读取后面4个byte,并将其转为dart类型中int类型.

![image-20190209194123776](https://ws1.sinaimg.cn/large/006tNc79ly1g00epfqv0vj31zk0pyae9.jpg)



# Handler

Flutter中定义了一套Handler用于处理经过Codec解码后消息.在使用Platform Channel时,需要为其设置对应的Handler,实际上就是为其注册一个对应BinaryMessageHandler,二进制数据会被BinaryMessageHanler进行处理,首先使用Codec进行解码操作,然后再分发给具体Handler进行处理.与三种Platform Channel相对应,Flutter中也定义了三种Handler:

- MessageHandler: 用于处理字符串或者半结构化消息,定义在BasicMessageChannel中.
- MethodCallHandler: 用于处理方法调用,定义在MethodChannel中.
- StreamHandler: 用于事件流通信,定义在EventChannel中.

## MessageHandler

用于处理字符串或者半结构化的消息,其定义如下:

```java
public final class BasicMessageChannel<T> {
    ......
        
    public interface MessageHandler<T> {
        void onMessage(T message, Reply<T> reply);
    }
    
    ......
}
```

`onMessage()`用于处理来自Flutter中的消息,该接受两个参数:T类型的消息以及用于异步返回T类型的result

## MethodCallHandler

 MethodCallHandler用于处理方法的调用,其定义如下:

```java
public final class MethodChannel {
    ......
        
    public interface MethodCallHandler {
        void onMethodCall(MethodCall call, Result result);
    }
    
    ......
}
```

`onMessage()`用于处理来自Flutter中消息,该方法接受两个参数:用于方法调用MethodCall对象以及用于方法返回Result类型的对象.

## StreamHandler

 StreamHandler用于事件流的通信,通常是用于平台主动向Flutter发送事件通知,其定义如下:

```java
public final class EventChannel {
    ......
        
    public interface StreamHandler {

        void onListen(Object arguments, EventSink events);

        void onCancel(Object arguments);
    }
    
    ......
}
```

在StreamHandler存在两个方法:`onListen()`和`onCancel()`.当Flutter端开始监听平台事件时,会向平台发起一次MethodCall,其中方法名为listen,也就是最终会调用StreamHandler中的`onListen()`方法.该方法中接受两个参数,其中EventSink类型的参数可用于向Flutter发送事件(实际上还是通过BinaryMessager).当Flutter开始停止监听平台事件时,会再向平台发起一次MethodCall,其中方法名为cancel,最终会调用StreamHandler的`onCancel()`方法,在该方法中通常需要销毁一些无用的资源.关于StreamHandler原理,会另开一文.

# MethodChannel调用原理

MethodChannel用于实现Flutter和Android/IOS平台间的方法调用.下文将以从Flutter调用Android平台代码为主线进行分析,主要涉及方法调用及方法调用结果返回两个过程.

## 方法调用

### Dart -> Native

MethodChannel定义如下:

```dart
class MethodChannel {
  // 构造方法,通常我们只需要指定该channel的name,  
  const MethodChannel(this.name, [this.codec = const StandardMethodCodec()]);
  // name作为通道的唯一标志付,用于区分不同的通道调用
  final String name;
  // 用于方法调用过程的编码 
  final MethodCodec codec;
  
  // 用于发起异步平台方法调用,需要指定方法名,以及可选方法参数  
  Future<dynamic> invokeMethod(String method, [dynamic arguments]) async {
    assert(method != null);
    // 将一次方法调用中需要的方法名和方法参数封装为MethodCall对象,然后使用MethodCodec对该
    //  对象进行进行编码操作,最后通过BinaryMessages中的send方法发起调用 
    final dynamic result = await BinaryMessages.send(
      name,
      codec.encodeMethodCall(MethodCall(method, arguments)),
    );
    if (result == null)
      throw MissingPluginException('No implementation found for method $method on channel $name');
    return codec.decodeEnvelope(result);
  }
}
```

在使用MethodChannel时,需要我们根据Channel名称来创建MethodChannel对象.Channel名称作为MethodChannel的唯一标识符,用于区分不同的MethodChannel对象.创建MethodChannel对象的一般方式如下:

```dart
final MethodChannel _channel = new MethodChannel('flutter.io/player')
```

拿到MethodChannel对象后,通过调用其`invokeMethod()`方法用于向平台发起一次调用.在`invokeMethod()`方法中会将一次方法调中的方法名method和方法参数arguments封装为MethodCall对象,然后使用MethodCodec对其进行二进制编码,最后通过BinaryMessages的send()发起平台方法调用请求.

```dart
class BinaryMessages {
   ...... 
   static Future<ByteData> send(String channel, ByteData message) {
    final _MessageHandler handler = _mockHandlers[channel];
    // 在没有设置Mock Handler的情况下,继续调用_sendPlatformMessage()   
    if (handler != null)
      return handler(message);
    return _sendPlatformMessage(channel, message);
  } 
 
   static Future<ByteData> _sendPlatformMessage(String channel, ByteData message) {
    final Completer<ByteData> completer = Completer<ByteData>();   
    ui.window.sendPlatformMessage(channel, message, (ByteData reply) {
      try {
        completer.complete(reply);
      } catch (exception, stack) {
        FlutterError.reportError(FlutterErrorDetails(
          exception: exception,
          stack: stack,
          library: 'services library',
          context: 'during a platform message response callback',
        ));
      }
    });
    return completer.future;
  } 
   ......  
}

```

BinaryMessages类中提供了用于发送和接受平台插件的二进制消息.

```dart
class Window{
    ......
    void sendPlatformMessage(String name,
                           ByteData data,
                           PlatformMessageResponseCallback callback) {
    final String error =
        _sendPlatformMessage(name, _zonedPlatformMessageResponseCallback(callback), data);
    if (error != null)
      throw new Exception(error);
  }
  // 和Java类似,Dart中同样提供了Native方法用于调用底层C++/C代码的能力  
  String _sendPlatformMessage(String name,
                              PlatformMessageResponseCallback callback,
                              ByteData data) native 'Window_sendPlatformMessage';
   ....... 
}
```

 上述过程最终会调用到ui.Window的`_sendPlatformMessage()`方法,该方法是一个native方法,这与Java中JNI技术非常类似.

![image-20190209112548889](https://ws2.sinaimg.cn/large/006tNc79ly1g000dx3s0fj31rl0u078i.jpg)

在调用该Native方法中,我们向native层发送了三个参数：

- name: String类型,代表Channel名称
- data: ByteData类型,代表之前封装的二进制数据
- callback: Function类型,用于结果回调

`_sendPlatformMessage()`具体实现在[Window.cc](https://sourcegraph.com/github.com/flutter/engine@053f7a8/-/blob/lib/ui/window/window.cc?utm_source=share#L336:14)中:

```c++
void Window::RegisterNatives(tonic::DartLibraryNatives* natives) {
  natives->Register({
      {"Window_defaultRouteName", DefaultRouteName, 1, true},
      {"Window_scheduleFrame", ScheduleFrame, 1, true},
      {"Window_sendPlatformMessage", _SendPlatformMessage, 4, true},
      {"Window_respondToPlatformMessage", _RespondToPlatformMessage, 3, true},
      {"Window_render", Render, 2, true},
      {"Window_updateSemantics", UpdateSemantics, 2, true},
      {"Window_setIsolateDebugName", SetIsolateDebugName, 2, true},
      {"Window_reportUnhandledException", ReportUnhandledException, 2, true},
  });
}
```

```c++
void _SendPlatformMessage(Dart_NativeArguments args) {
  // 最终调用SendPlatformMessage函数  
  tonic::DartCallStatic(&SendPlatformMessage, args);
}
```

[SendPlatformMessage()](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/lib/ui/window/window.cc?utm_source=share#L84:13)函数定义如下:

```c++
Dart_Handle SendPlatformMessage(Dart_Handle window,
                                const std::string& name,
                                Dart_Handle callback,
                                const tonic::DartByteData& data) {
  UIDartState* dart_state = UIDartState::Current();
  // 1.只能在main iolate调用平台方法
  if (!dart_state->window()) {
    // Must release the TypedData buffer before allocating other Dart objects.
    data.Release();
    return tonic::ToDart(
        "Platform messages can only be sent from the main isolate");
  }
  // 此处response的作用?	
  fml::RefPtr<PlatformMessageResponse> response;
  if (!Dart_IsNull(callback)) {
    response = fml::MakeRefCounted<PlatformMessageResponseDart>(
        tonic::DartPersistentValue(dart_state, callback),
        dart_state->GetTaskRunners().GetUITaskRunner());
  }
  // 2.核心方法调用 
  if (Dart_IsNull(data.dart_handle())) {
    dart_state->window()->client()->HandlePlatformMessage(
        fml::MakeRefCounted<PlatformMessage>(name, response));
  } else {
    const uint8_t* buffer = static_cast<const uint8_t*>(data.data());
		
    dart_state->window()->client()->HandlePlatformMessage(
        fml::MakeRefCounted<PlatformMessage>(
            name, std::vector<uint8_t>(buffer, buffer + data.length_in_bytes()),
            response));
  }

  return Dart_Null();
}
```

在上方代码中首先判断是否在main isolate中进行平台方法调用,如果不是返回错误信息;接下来就是执行关键的方法:`dart_state->window()->client()->HandlePlatformMessage()`.其中`HandlePlatformMessage()`是[WindowClient](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/lib/ui/window/window.h?utm_source=share#L39:7)中虚方法:

```c++
class WindowClient {
 public:
  virtual std::string DefaultRouteName() = 0;
  virtual void ScheduleFrame() = 0;
  virtual void Render(Scene* scene) = 0;
  virtual void UpdateSemantics(SemanticsUpdate* update) = 0;
  virtual void HandlePlatformMessage(fml::RefPtr<PlatformMessage> message) = 0;
  virtual FontCollection& GetFontCollection() = 0;
  virtual void UpdateIsolateDescription(const std::string isolate_name,
                                        int64_t isolate_port) = 0;

 protected:
  virtual ~WindowClient();
};
```

而[RuntimeController](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/runtime/runtime_controller.cc?utm_source=share#L272:6)是WindowClient中的唯一继承类,其头文件[runtime_controller.h](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/runtime/runtime_controller.h?utm_source=share#L137:33),为了方便起见,我们只看其部分定义:

```c++
class RuntimeController final : public WindowClient {
    ......
        
    private:
      RuntimeDelegate& client_;
    
      .......
      void HandlePlatformMessage(fml::RefPtr<PlatformMessage> message) override;
      ......

}
```

知道其定义之后,现在就可以看起具体在[runtime_controller.cc](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/runtime/runtime_controller.cc?utm_source=share#L272:25)的实现了:

```c++
void RuntimeController::HandlePlatformMessage(
    fml::RefPtr<PlatformMessage> message) {
  client_.HandlePlatformMessage(std::move(message));
}
```

在运行过程中,不同的平台有运行机制不同,需要不同的处理策略,因此RuntimeController中相关的方法实现都被委托到了不同的平台实现类RuntimeDelegate中,即上述代码中`client_`.

RuntimeDelegate的实现为Engine,其头文件[Engine.h](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/common/engine.h?utm_source=share)定义如下:

```c++
class Engine final : public blink::RuntimeDelegate {
    ........
}
```

对应真实的实现在[Engine.cc](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/common/engine.cc?utm_source=share#L414:14)中,直接来看其`HandlePlatformMessage()`的实现

```c++
void Engine::HandlePlatformMessage(
    fml::RefPtr<blink::PlatformMessage> message) {
  // kAssetChannel值为flutter/assets  
  if (message->channel() == kAssetChannel) {
    HandleAssetPlatformMessage(std::move(message));
  } else {
    delegate_.OnEngineHandlePlatformMessage(std::move(message));
  }
}
```

在上述代码过程中,Engine在处理message时,如果该message值等于kAssetChannel,即flutter/assets,表示当前操作想要获取资源,因此会调用`HandleAssetPlatformMessage()`来走获取资源的逻辑;否则调用`delegate_.OnEngineHandlePlatformMessage()`方法.这里delegate的具体实现为[Shell](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/common/shell.cc?utm_source=share#L723):

```c++
void Shell::OnEngineHandlePlatformMessage(
    fml::RefPtr<blink::PlatformMessage> message) {
  FML_DCHECK(is_setup_);
  FML_DCHECK(task_runners_.GetUITaskRunner()->RunsTasksOnCurrentThread());

  // kSkiaChannel值为flutter/skia  
  if (message->channel() == kSkiaChannel) {
    HandleEngineSkiaMessage(std::move(message));
    return;
  }
  // 其他情况下,向PlatformTaskRunner中添加Task
  task_runners_.GetPlatformTaskRunner()->PostTask(
      [view = platform_view_->GetWeakPtr(), message = std::move(message)]() {
        if (view) {
          view->HandlePlatformMessage(std::move(message));
        }
      });
}
```

`OnEngineHandlePlatformMessage`在接收到消息后,首先判断要调用Channel是否是flutter/skia,如果是则调用`HandleEngineSkiaMessage()`进行处理后返回,否则向PlatformTaskRunner添加一个Task,在该Task中会调用PlatformView的`HandlePlatformMessage()`方法.根据运行平台不同PlatformView有不同的实现,对于Android平台而言,其具体实现是PlatformViewAndroid;对于IOS平台而言,其实现是PlatformViewIOS.

![image-20190208112830510](https://ws2.sinaimg.cn/large/006tNc79ly1fzyuucjellj31020jogmp.jpg)



>  Task中的代码执行在Platform Task Runner中,而之前的代码运行在UI Task Runner中

以PlatformViewAndroid为例,来了解`HandlePlatformMessage()`[方法的实现](https://sourcegraph.com/github.com/flutter/engine@053f7a8/-/blob/shell/platform/android/platform_view_android.cc?utm_source=share#L142:27):

```c++
// |shell::PlatformView|
void PlatformViewAndroid::HandlePlatformMessage(
    fml::RefPtr<blink::PlatformMessage> message) {
  JNIEnv* env = fml::jni::AttachCurrentThread();
  fml::jni::ScopedJavaLocalRef<jobject> view = java_object_.get(env);
  if (view.is_null())
    return;
  // response_id在Flutter调用平台代码时,会传到平台代码中,后续平台代码需要回传数据时
  // 需要用到它
  int response_id = 0;
  // 如果message中有response(response类型为PlatformMessageResponseDart),则需要对
  // response_id进行自增  
  if (auto response = message->response()) {
    response_id = next_response_id_++;
    // pending_responses是一个Map结构  
    pending_responses_[response_id] = response;
  }
  auto java_channel = fml::jni::StringToJavaString(env, message->channel());
  if (message->hasData()) { 
    fml::jni::ScopedJavaLocalRef<jbyteArray> message_array(
        env, env->NewByteArray(message->data().size()));
    env->SetByteArrayRegion(
        message_array.obj(), 0, message->data().size(),
        reinterpret_cast<const jbyte*>(message->data().data()));
    message = nullptr;

    // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
    FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(),
                                     message_array.obj(), response_id);
  } else {  
    message = nullptr;
    // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
    FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(),
                                     nullptr, response_id);
  }
}
```

在上述代码中,当该方法接受PlatformMessage类型的消息时,如果消息中有response,则对response_id自增,并以response_ide为key,response为value存放在变量`pending_responses_`中(pending_responsed是一个Map结构).

> 正常情况下,每次从Flutter调用Channel代码,都会生成对应的response_id和response.同时该response_id会被传到平台代码中,当平台代码需要为此次调用返回数据时,需要同时回传该response_id.

接着将消息中的channel和data数据转成Java可识别的数据,并连同response_id一同作为`FlutterViewHandlePlatformMessage()`方法的参数,最终通过JNI调用的方式传递到Java层.

简单的分析下该过程,首先来看`FlutterViewHandlePlatformMessage()`在[platform_android_jni.cc](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/platform_view_android_jni.cc?utm_source=share#L70:6)中的实现:

```c
static jmethodID g_handle_platform_message_method = nullptr;
void FlutterViewHandlePlatformMessage(JNIEnv* env,
                                      jobject obj,
                                      jstring channel,
                                      jobject message,
                                      jint responseId) {
  env->CallVoidMethod(obj, g_handle_platform_message_method, channel, message,
                      responseId);
  FML_CHECK(CheckException(env));
}
```

在上述方法中,最终将调用g_handle_platform_message_method中指向Java层的方法.其中g_handle_platform_message_method在[RegisterApi](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/platform_view_android_jni.cc?utm_source=share#L541:6)中被初始化:

```c
bool PlatformViewAndroid::Register(JNIEnv* env) {
  ......  
  g_flutter_jni_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/embedding/engine/FlutterJNI"));
    
  ......
      
  return RegisterApi(env);
}

bool RegisterApi(JNIEnv* env) {
  ......  
  g_handle_platform_message_method =
      env->GetMethodID(g_flutter_jni_class->obj(), "handlePlatformMessage",
                       "(Ljava/lang/String;[BI)V");
  ......  
}
```

不难看出g_flutter_jni_class指向FlutterJNI.java类,g_handle_platform_message_method指向FlutterJN.javaI中的`handlePlatformMessage()`方法.

现在我们知道从Flutter发一次Channel调用,需要经过`Dart -> Native -> JNI -> Java`几个层次,并把三个参数channel,message,responseId传给Java层.

![image-20190209113644507](https://ws2.sinaimg.cn/large/006tNc79ly1g000p630c4j31xj0u0wn3.jpg)

### Native -> Java

通过上述分析,我们知道`FlutterViewHandlePlatformMessage()`实际上是通过JNI的方式最终调用了FlutterJNI.java中的`handlePlatformMessage()`方法,该方法接受三个来自Native层的参数:

- channel: String类型,表示Channel名称.
- message: 字节数组,表示方法调用中的数据,如方法名和参数.
- replyId: int类型,在将此次调用的响应数据从Java层写回到Native层时用到

```java
public class FlutterJNI {
  private PlatformMessageHandler platformMessageHandler;
    
      @UiThread
  public void setPlatformMessageHandler(@Nullable PlatformMessageHandler platformMessageHandler) {
    this.platformMessageHandler = platformMessageHandler;
  }
    
      // Called by native.
  @SuppressWarnings("unused")
  private void handlePlatformMessage(final String channel, byte[] message, final int replyId) {
    if (platformMessageHandler != null) {
      platformMessageHandler.handleMessageFromDart(channel, message, replyId);
    }
  }
}
```

FlutterJNI类定义了Java层和Flutter C/C++引擎之间的相关接口.此类目前处于实验性质,随着后续的发展可能会被不断的重构和优化,不保证一直存在,不建议开发者调用该类.

为了建立Android应用和Flutter C/C++引擎的连接,需要创建FlutterJNI实例,然后将其attach到Native,常见的使用方法如下:

```java
// 1.创建FlutterJNI实例
FlutterJNI flutterJNI = new FlutterJNI();
// 2.建立和Native层的连接
flutterJNI.attachToNative();

......
    
// 3.断开和Native层的连接,并释放资源
flutterJNI.detachFromNativeAndReleaseResources();
```

重新回到FlutterJNI中`handlePlatformMessage()`,在该方法中首先判断platformMessageHandler是否为null,不为null,则调用其`handleMessageFromDart()`方法.其中platformMessageHandler需要通过FlutterJNI中的`setPlatformMessageHandler()`方法来设置.那该方法被调用的时机是在什么时候呢?直接来看FlutterNativeView的[构造函数](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/io/flutter/view/FlutterNativeView.java?utm_source=share#L39:12):

```java
public class FlutterNativeView implements BinaryMessenger {

    private final Map<String, BinaryMessageHandler> mMessageHandlers;
    private int mNextReplyId = 1;
    private final Map<Integer, BinaryReply> mPendingReplies = new HashMap<>();

    private final FlutterPluginRegistry mPluginRegistry;
    private FlutterView mFlutterView;
    private FlutterJNI mFlutterJNI;
    private final Context mContext;
    private boolean applicationIsRunning;

    public FlutterNativeView(Context context, boolean isBackgroundView) {
        mContext = context;
        mPluginRegistry = new FlutterPluginRegistry(this, context);
        // 创建FlutterJNI实例
        mFlutterJNI = new FlutterJNI();
        mFlutterJNI.setRenderSurface(new RenderSurfaceImpl());
        // 将PlatformMessageHandlerImpl实例赋值给FlutterJNI中的platformMessageHandler属性
        mFlutterJNI.setPlatformMessageHandler(new PlatformMessageHandlerImpl());
        mFlutterJNI.addEngineLifecycleListener(new EngineLifecycleListenerImpl());
        attach(this, isBackgroundView);
        assertAttached();
        mMessageHandlers = new HashMap<>();
    }
    
    .......
}

```

在FlutterNativeView的构造函数中,首先创建FlutterJNI实例mFlutterJNI,然后调用`setPlatformMessageHandler()`并把PlatformMessageHandlerImpl实例作为参数传入.因此在FlutterJNI的`handlePlatformMessage()`方法中,最终调用PlatformMessageHandlerImpl实例的`handleMessageFromDart()`来处理来自Flutter中的消息:

```java
public class FlutterNativeView implements BinaryMessenger {
        private final Map<String, BinaryMessageHandler> mMessageHandlers;
    
    	......
            
        private final class PlatformMessageHandlerImpl implements PlatformMessageHandler {
        // Called by native to send us a platform message.
        public void handleMessageFromDart(final String channel, byte[] message, final int replyId) {
			// 1.根据channel名称获取对应的BinaryMessageHandler对象.每个Channel对应一个
            // Handler对象
            BinaryMessageHandler handler = mMessageHandlers.get(channel);
            if (handler != null) {
                try {
                    // 2.将字节数组对象封装为ByteBuffer对象
                    final ByteBuffer buffer = (message == null ? null : ByteBuffer.wrap(message));
                    // 3.调用handler对象的onMessage()方法来分发消息
                    handler.onMessage(buffer, new BinaryReply() {
                        private final AtomicBoolean done = new AtomicBoolean(false);
                        @Override
                        public void reply(ByteBuffer reply) {
                            // 4.根据reply的情况,调用FlutterJNI中invokePlatformMessageXXX()方法将响应数据发送给Flutter层
                            if (reply == null) {
                                mFlutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
                            } else {
                                mFlutterJNI.invokePlatformMessageResponseCallback(replyId, reply, reply.position());
                            }
                        }
                    });
                } catch (Exception exception) {
                    mFlutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
                }
                return;
            }
            mFlutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
        }
}
```

在FlutterNativeView中存在Map类型的成员变量<span id="BinaryMessenger">mMessageHandler</span>:以Channel名称作为key,以BinaryMessageHandler类型为value.在`handleMessageFromDart()`方法中,首先根据Channel名称从mMessageHandlers取出对应的二进制消息处理器BinaryMessageHandler,然后将字节数组message封装为ByteBuffer对象,然后调用BinaryMessageHandler实例的`onMessage()`方法处理ByteBuffer,并进行响应.

BinaryReply是一个接口,主要用来将ByteBuffer类型的响应数据reply从Java层写回到Flutter层.根据reply是否为null,调用FlutterJNI实例不同的方法:reply为null时,调用`invokePlatformMessageEmptyResponseCallback()`;否则调用`invokePlatformMessageResponseCallback()`.关于具体的实现流程,稍后再述.

现在来看下BinaryMessageHandler是如何添加到mMessageHandler中:

```java
public class FlutterNativeView implements BinaryMessenger {
    private final Map<String, BinaryMessageHandler> mMessageHandlers;
    
    ......
        
    @Override
    public void setMessageHandler(String channel, BinaryMessageHandler handler) {
        if (handler == null) {
            mMessageHandlers.remove(channel);
        } else {
            mMessageHandlers.put(channel, handler);
        }
    }
    .......
}
```

`setMessageHandler()`方法接受两个参数:channel作为Channel的名称,用来区分不同的Channel;handler是该Channel对应的二进制消息处理器.在该方法中,会根据handler是否为null来决定对mMessageHandlers是进行添加还是删除操作.那该方法什么时候回被调用呢?要想弄明白这个问题,需要了解编写平台Channel的过程.以官方获取Android平台电量的平台Channel为例:

```java
public class MainActivity extends FlutterActivity {
    // 1.定义Channel的名称,该名称作为Channel的唯一标识符
    private static final String CHANNEL = "samples.flutter.io/battery";

    @Override
    public void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);
		// 2.创建MethodChannel对象channel
        MethodChannel channel = new MethodChannel(getFlutterView(), CHANNEL);
        // 3.调用MethodChannel实例的setMethodCallHandler()方法为当前channel设置Handler
        channel.setMethodCallHandler(
                new MethodCallHandler() {
                    @Override
                    public void onMethodCall(MethodCall call, Result result) {
                        // TODO
                    }
                });
    }
}
```

在上述代码中,演示了编写平台代码的基本过程:首先创建MethodChanel实例,然后设置MethodCallHandler.

```java
public final class MethodChannel {
    // 二进制信使
    private final BinaryMessenger messenger;
    // Channel名称
    private final String name;
    // 方法编码
    private final MethodCodec codec;

    public MethodChannel(BinaryMessenger messenger, String name) {
        this(messenger, name, StandardMethodCodec.INSTANCE);
    }
        
    public MethodChannel(BinaryMessenger messenger, String name, MethodCodec codec) {
        assert messenger != null;
        assert name != null;
        assert codec != null;
        this.messenger = messenger;
        this.name = name;
        this.codec = codec;
    }    
    
    ......
        
    public void setMethodCallHandler(final @Nullable MethodCallHandler handler) {
        messenger.setMessageHandler(name,
            handler == null ? null : new IncomingMethodCallHandler(handler));
    }
    ......
}
```

在创建MethodChannel过程中需要指定三个参数:

- name: Channel名称,用作唯一标识符.
- codec: 用于方法编码的MethodCodec,.默认情况下,codec被指定为StandartMethodCodec.INSTANCE.
- messager: 用于消息发送的BinaryMessager,

创建完MethodChannel后,接下来需要调用`setMethodCallHandler()`设置用于处理方法调用MethodCallHandler.在该方法参数handler不为null的情况下,会将该handler包装为IncomingMethodCallHandler实例,然后调用BinaryMessager实例的`setMessageHanlder()`方法将I该ncomingMethodCallHandler实例保存在FlutterNativeView中的mMessageHandlers中.

简单来说就是在编写平台Channel时,需要创建对应的MethodChannel实例,并调用其`setMethodCallHandler()`将MethodCallHandler实例保存起来.

```java
public final class MethodChannel {

    private final class IncomingMethodCallHandler implements BinaryMessageHandler {
        private final MethodCallHandler handler;

        IncomingMethodCallHandler(MethodCallHandler handler) {
            this.handler = handler;
        }

        @Override
        public void onMessage(ByteBuffer message, final BinaryReply reply) {
            // 1.使用codec对来自Flutter方法调用数据进行解码,并将其封装为MethodCall对象.
            // MethodCall中包含两部分数据:method表示要调用的方法;arguments表示方法所需参数
            final MethodCall call = codec.decodeMethodCall(message);
            try {
                // 2.调用自定义MethodCallHandler中的onMethodCall方法继续处理方法调用
                handler.onMethodCall(call, new Result() {
                    @Override
                    public void success(Object result) {
                        // 调用成功时,需要回传数据给Flutter层时,使用codec对回传数据result
                        // 进行编码
                        reply.reply(codec.encodeSuccessEnvelope(result));
                    }

                    @Override
                    public void error(String errorCode, String errorMessage, Object errorDetails) {
                        // 调用失败时,需要回传错误数据给Flutter层时,使用codec对errorCode,
                        // errorMessage,errorDetails进行编码
                        reply.reply(codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
                    }

                    @Override
                    public void notImplemented() {
                        // 方法没有实现时,调用该方法后,flutter将会受到相应的错误消息
                        reply.reply(null);
                    }
                });
            } catch (RuntimeException e) {
                Log.e(TAG + name, "Failed to handle method call", e);
                reply.reply(codec.encodeErrorEnvelope("error", e.getMessage(), null));
            }
        }
    }
}
```

在上述代码中,首先使用codec对来自Flutter层的二进制数据进行解码,并将其封装为MethodCall对象,然后调用MethodCallHandler的`onMethodCall()`方法.

![image-20190209115925942](https://ws3.sinaimg.cn/large/006tNc79ly1g001crq3m8j31p90u0gr6.jpg)

.MethodCall中包含两部分数据:method部分表示要调用的方法;arguments表示被调用方法所需要的参数.

```java
public final class MethodChannel {
    .......
    public interface MethodCallHandler {
      void onMethodCall(MethodCall call, Result result);
    }
    .......
}
```

接下来便是调用自定义MethodCallHandler中的`onMethodCall()`方法,该方法接受两个参数:

- call: MethodCall类型,它包含方法调用所需的信息

- result: Result类型,用于处理方法调用结果

Result是一个回调接口,其定义如下:

```java
public final class MethodChannel {
    .......
    public interface Result {
        // 方法调用处理成功时,调用success()
        void success(@Nullable Object result);
        // 方法调用处理失败时,调用error()
        void error(String errorCode, @Nullable String errorMessage, @Nullable Object errorDetails);
        // 方法调用遇到为定义的方法时,调用notImplemented()
        void notImplemented();
    }
    .......
}
```

## 方法返回

### Java -> Native

回到IncomingMethodCallHandler中的`onMessage()`中,我们看到在调用MethodCallHandler的`onMethodCall()`时,以匿名内部的形式实现了Result接口,而且在实现中又调用reply实例的`reply()`方法来把响应数据写会到Flutter层.reply是BinaryReply接口类型,其具体实现之前已经说过(即[PlatformMessageHandlerImpl](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/io/flutter/view/FlutterNativeView.java?utm_source=share#L182:21),在此就不做重复了.

当数据需要写回时,数据首先通过codec被编码成ByteBuffer类型,然后调用reply的`reply()`方法.在`reply()`方法中,对于非null类型的ByteBuffer,会调用FlutterJNI中的`invokePlatformMessageResponseCallback()`

```java
public class FlutterJNI {
  private Long nativePlatformViewId;
    
  ......  
  @UiThread
  public void invokePlatformMessageResponseCallback(int responseId, ByteBuffer message, int position) {
    // 1.检查FlutterJNI是否已经attach到Native层,如若没有则抛出异常  
    ensureAttachedToNative();
    // 2.继续调用nativeInvokePlatformMessageResponseCallback()  
    nativeInvokePlatformMessageResponseCallback(
        nativePlatformViewId,
        responseId,
        message,
        position
    );
  }
    
  private native void nativeInvokePlatformMessageResponseCallback(
     long nativePlatformViewId,
     int responseId,
     ByteBuffer message,
     int position
  );   
    
  ......  
      
  private void ensureAttachedToNative() {
    // FlutterJNI attach到Native层后,会返回一个long类型的值用来初始化nativePlatformViewId  
    if (nativePlatformViewId == null) {
      throw new RuntimeException("Cannot execute operation because FlutterJNI is not attached to native.");
    }
  }

}
```

在上述`invokePlatformMessageResponseCallback()`方法中,首先检查当前FlutterJNI实例是否已经attach到Native层,然后调用Native方法`nativeInvokePlatformMessageResponseCallback()`向JNI层写入数据,该Native方法Native层的实现在[platform_view_android.cc](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/platform_view_android_jni.cc?utm_source=share#L583:20),其中参数responseId是之前Native层生成,又传至Java层的,现在有又需要将它传至Native层:

```c
bool RegisterApi(JNIEnv* env) {
    static const JNINativeMethod flutter_jni_methods[] = {
        ......
         {
          .name = "nativeInvokePlatformMessageResponseCallback",
          .signature = "(JILjava/nio/ByteBuffer;I)V",
          .fnPtr = reinterpret_cast<void*>(&shell::InvokePlatformMessageResponseCallback),
         },
        .......
    }
    .......
}
```

通过上述代码定义不难看出其最终[实现](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/shell/platform/android/platform_view_android.cc?utm_source=share#L108)如下:

```c++
void PlatformViewAndroid::InvokePlatformMessageResponseCallback(
    JNIEnv* env,
    jint response_id,
    jobject java_response_data,
    jint java_response_position) {
  if (!response_id)
    return;
  // 1.通过response_id从pending_responses_中取出response  
  auto it = pending_responses_.find(response_id);
  if (it == pending_responses_.end())
    return;
  // 2.GetDirectBufferAddress函数返回一个指向被传入的ByteBuffer对象的地址指针  
  uint8_t* response_data =
      static_cast<uint8_t*>(env->GetDirectBufferAddress(java_response_data));
  std::vector<uint8_t> response = std::vector<uint8_t>(
      response_data, response_data + java_response_position);
  auto message_response = std::move(it->second);
  // 3.从pending_responses_中移除该response  
  pending_responses_.erase(it);
  // 4.调用response的Complete()方法将二进制结果返回
  message_response->Complete(
      std::make_unique<fml::DataMapping>(std::move(response)));
}
```

在上述代码中,首先以response_id为key,从`pending_responsed_`取出对应response.然后通过GetDirectBufferAddress函数获取二进制响应数据java_response_data对象的指针,最后调用reponse的`Complete()`方法将二进制结果返回.

### Native -> Dart

上文提到response是PlatformMessageResponseDart类型,简单看一下其`Complete()`[方法实现](https://sourcegraph.com/github.com/flutter/engine@053f7a8fa3bb4c50c28451f5403f176846692ce0/-/blob/lib/ui/window/platform_message_response_dart.cc?utm_source=share#L63:6):

```c++
void PlatformMessageResponseDart::Complete(std::unique_ptr<fml::Mapping> data) {
  if (callback_.is_empty())
    return;
  FML_DCHECK(!is_complete_);
  is_complete_ = true;
  ui_task_runner_->PostTask(fml::MakeCopyable(
      [callback = std::move(callback_), data = std::move(data)]() mutable {
        std::shared_ptr<tonic::DartState> dart_state =
            callback.dart_state().lock();
        if (!dart_state)
          return;
        tonic::DartState::Scope scope(dart_state);
		// 将Native层的二进制数据data转为Dart中的二进制数据byte_buffer
        Dart_Handle byte_buffer = WrapByteData(std::move(data));
        tonic::DartInvoke(callback.Release(), {byte_buffer});
      }));
}

```

在上述代码中,向ui_task_runner_添加了一个新的Task,在该Task中首先将Native层二进制数据转为Dart中的二进制数据,然后调用Dart中的callback将数据返回到Dart层.Dart层在接受到数据后,使用MethodCodec进行解码数据并将其返回到业务代码.

## 小结

上述以Dart作为发起方,完整梳理一次从Dart向Android平台发起方法调用(MethodCall)流程.实际MethodCall也支持以Android平台为发起方,通过MethodChannel调用Dart中的方法,对此就不做说明了.=

# 总结

目前为止,关于Platform Channel基本原理已经讲解完成,最后通过对MethodChannel工作原理进行分析来加深理解.对于IOS开发者而言,其原理实现类似.

总体而言,Flutter中实现跨平台通信的机制简单却高效,总体难度不大.最后祝大家新年快乐,人人都能拐卖小学妹😈(Flutter).