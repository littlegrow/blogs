---
title: okio 简单使用总结
date: 2019-9-11 19:00:00
description: "作为OkHttp的底层IO库，Okio确实比传统的java输入输出流读写更加方便高效。Okio补充了java.io和java.nio的不足，使访问、存储和处理数据更加容易，它起初只是作为OKHttp的一个组件，现在你可以独立的使用它来解决一些IO问题。"
categories:
- okio
---

### 概述

<br>

**基本特性**：

okio 是对 Java IO/NIO非常优秀的封装，“短小精焊”，不仅支持文件读写，也支持Socket读写；
不用区分字符流或者字节流，只有一个输入流Source和一个输出流Sink，使用简单；
提供丰富的API，支持各种类型数据读写；
优秀的缓冲机制和处理内存的技巧，使I/O在缓冲区得到更高的复用处理，从而尽量减少I/O的实际发生。

**核心基础类**：

**ByteString**：不可变的字节序列，可以很容易地将字节数组当作一个值来处理。
**Buffer**：可变的字节序列，可灵活访问，插入与移除，不需要自己去动手管理，内部维护 `Segment` 双项链表实现。

<br>

Okio自己的stream类型: `Source` 和 `Sink` ，分别类似于java的 `Inputstream` 和 `Outputstream`，提供简单的接口：


```
public interface Source : java.io.Closeable {
    public abstract fun close(): kotlin.Unit

    public abstract fun read(sink: okio.Buffer, byteCount: kotlin.Long): kotlin.Long

    public abstract fun timeout(): okio.Timeout
}

public interface Sink : java.io.Closeable, java.io.Flushable {
    public abstract fun close(): kotlin.Unit

    public abstract fun flush(): kotlin.Unit

    public abstract fun timeout(): okio.Timeout

    public abstract fun write(source: okio.Buffer, byteCount: kotlin.Long): kotlin.Unit
}
```

通过 `Bufferedsource` 和 `Bufferedsink` 提供了丰富的数据读写接口, 这两个接口可以满足你所需的一切。具体的实现通过 `RealBufferedSource` 和 `RealBufferedSink` 操作  `Buffer` 实现。

<br>

### okio读文件

<br>

```
    void readFromFile(String path) {
        try {
            Source source = Okio.source(new File(path));
            BufferedSource bufferedSource = Okio.buffer(source);

            String line;
            while ((line = bufferedSource.readUtf8Line()) != null) {
                System.out.println(line);
            }

            bufferedSource.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

`BufferedSource` 提供了丰富的API，可以很方便的读出不同编码格式的内容

<br>

### okio写文件

<br>

```
    void writeEnvToFile(String path) {
        try {
            Sink sink = Okio.sink(new File(path));
            BufferedSink bufferedSink = Okio.buffer(sink);

            for (Map.Entry<String, String> entry : System.getenv().entrySet()) {
                bufferedSink.writeUtf8(entry.getKey());
                bufferedSink.writeUtf8("=");
                bufferedSink.writeUtf8(entry.getValue());
                bufferedSink.writeUtf8("\n");
            }

            bufferedSink.flush();
            bufferedSink.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

`BufferedSource` 提供了丰富的API，可以很方便的写入不同编码格式的内容，与 `BufferedSource` 对应。

<br>

### 对象序列化与反序列化

<br>

支持将对象转化为16进制字符串或base64编码，同时支持将16进制或base64编码字符串转化为对象，可用于对象序列化传输。

工具类编写：

```
public class OkioSerializeUtils {
    /**
     * 对象序列化为16进制字符串
     */
    public static <T extends Serializable> String serializeHexString(T obj) {
        return serialize(obj).hex();
    }

    /**
     * 对象序列化为base64编码
     */
    public static <T extends Serializable> String serializeBase64(T obj) {
        return serialize(obj).base64();
    }


    /**
     * base64编码反序列化对象
     */
    public static <T extends Serializable> T deserializeBase64(String base64String) {
        return deserialize(ByteString.decodeBase64(base64String));
    }

    /**
     * 16进制字符串反序列化对象
     */
    public static <T extends Serializable> T deserializeHexString(String hexString) {
        return deserialize(ByteString.decodeHex(hexString));
    }

    /**
     * 序列化
     */
    private static <T extends Serializable> ByteString serialize(T obj) {
        Buffer buffer = new Buffer();
        try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(buffer.outputStream())) {
            objectOutputStream.writeObject(obj);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return buffer.readByteString();
    }

    /**
     * 反序列化
     */
    private static <T extends Serializable> T deserialize(ByteString byteString) {
        Buffer buffer = new Buffer();
        buffer.write(byteString);
        try (ObjectInputStream objectInputStream = new ObjectInputStream(buffer.inputStream())) {
            return (T) objectInputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

测试类：

```
public class SerializeTest {
    static class TestObj implements Serializable {
        private int a;
        private int b;

        TestObj(int a, int b) {
            this.a = a;
            this.b = b;
        }

        @Override
        public boolean equals(Object obj) {
            if (!(obj instanceof TestObj)) {
                return false;
            }
            TestObj testObj = (TestObj) obj;
            return testObj.a == a && testObj.b == b;
        }

        @Override
        public int hashCode() {
            int result = 17;
            result = 31 * result + a;
            return 31 * result + b;
        }

        @Override
        public String toString() {
            return String.format("[TestObj: a=%s, b=%s]", this.a, this.b);
        }
    }

    public static void main(String[] args) {
        TestObj testObj = new TestObj(1, 2);
        System.out.println("origin obj: " + testObj);
        System.out.println("serialize hexString: " + OkioSerializeUtils.serializeHexString(testObj));
        System.out.println("serialize base64: " + OkioSerializeUtils.serializeBase64(testObj));
        System.out.println("deserialize hexString: " + OkioSerializeUtils.deserializeHexString("aced00057372001a6f6b696f2e53657269616c697a655465737424546573744f626a8719308efba7bd66020002490001614900016278700000000100000002"));
        System.out.println("deserialize base64: " + OkioSerializeUtils.deserializeBase64("rO0ABXNyABpva2lvLlNlcmlhbGl6ZVRlc3QkVGVzdE9iaocZMI77p71mAgACSQABYUkAAWJ4cAAAAAEAAAAC"));
    }
}
```

输出结果:

```
origin obj: [TestObj: a=1, b=2]
serialize hexString: aced00057372001a6f6b696f2e53657269616c697a655465737424546573744f626a8719308efba7bd66020002490001614900016278700000000100000002
serialize base64: rO0ABXNyABpva2lvLlNlcmlhbGl6ZVRlc3QkVGVzdE9iaocZMI77p71mAgACSQABYUkAAWJ4cAAAAAEAAAAC
deserialize hexString: [TestObj: a=1, b=2]
deserialize base64: [TestObj: a=1, b=2]
```

<br><br><br><br>