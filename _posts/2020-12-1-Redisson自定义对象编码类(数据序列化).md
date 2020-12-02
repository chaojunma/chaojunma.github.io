---
layout: post
title: "Redisson自定义对象编码类(数据序列化)"
date: 2020-12-1
categories: 分布式
tags: Redisson Redis  
--- 

Redisson的对象编码类是用于将对象进行序列化和反序列化，以实现对该对象在Redis里的读取和存储。Redisson提供了以下几种的对象编码应用，以供大家选择：

编码类名称 |	说明
-|-|
org.redisson.codec.JsonJacksonCodec	Jackson |JSON 编码 默认编码|
org.redisson.codec.AvroJacksonCodec	Avro |一个二进制的JSON编码|
org.redisson.codec.SmileJacksonCodec	|Smile 另一个二进制的JSON编码|
org.redisson.codec.CborJacksonCodec	|CBOR 又一个二进制的JSON编码|
org.redisson.codec.MsgPackJacksonCodec	|MsgPack 再来一个二进制的JSON编码|
org.redisson.codec.IonJacksonCodec	|Amazon Ion 亚马逊的Ion编码，格式与JSON类似|
org.redisson.codec.KryoCodec	|Kryo 二进制对象序列化编码|
org.redisson.codec.SerializationCodec|	JDK序列化编码|
org.redisson.codec.FstCodec	|FST 10倍于JDK序列化性能而且100%兼容的编码|
org.redisson.codec.LZ4Codec	|LZ4 压缩型序列化对象编码|
org.redisson.codec.SnappyCodec	|Snappy 另一个压缩型序列化对象编码
org.redisson.client.codec.JsonJacksonMapCodec|	基于Jackson的映射类使用的编码。可用于避免序列化类的信息，以及用于解决使用byte[]遇到的问题。|
org.redisson.client.codec.StringCodec	|纯字符串编码（无转换）|
org.redisson.client.codec.LongCodec	|纯整长型数字编码（无转换）|
org.redisson.client.codec.ByteArrayCodec	|字节数组编码|
org.redisson.codec.CompositeCodec	|用来组合多种不同编码在一起|


除了以上提供的一些对象编码类，另外Redisson也支持自定义对象编码类用户数据的序列化和反序列化。Redisson自定义对象编码类需要继承`BaseCodec`类，具体实现如下：

***添加依赖***

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.60</version>
</dependency>
```

这里我们添加fastjson的依赖是需要fastjson实现序列化和返序列化

***创建对象编码类***

```java
public class FastjsonCodec extends BaseCodec {


    private final Encoder encoder = new Encoder() {
        @Override
        public ByteBuf encode(Object in) throws IOException {
            ByteBuf out = ByteBufAllocator.DEFAULT.buffer();
            try {
                ByteBufOutputStream os = new ByteBufOutputStream(out);
                JSON.writeJSONString(os, in, SerializerFeature.WriteClassName);
                return os.buffer();
            } catch (IOException e) {
                out.release();
                throw e;
            } catch (Exception e) {
                out.release();
                throw new IOException(e);
            }
        }
    };


    private final Decoder<Object> decoder = new Decoder<Object>() {

        @Override
        public Object decode(ByteBuf buf, State state) throws IOException {
            ParserConfig.getGlobalInstance().addAccept("com.mk.provider.config.");
            return JSON.parseObject(new ByteBufInputStream(buf), Object.class);
        }
    };

    @Override
    public Decoder<Object> getValueDecoder() {
        return decoder;
    }

    @Override
    public Encoder getValueEncoder() {
        return encoder;
    }
}
```

注意：以上代码中有`ParserConfig.getGlobalInstance().addAccept("com.mk.provider.config.")`这样一行，这里很重要，这里需要添加待序列化对象的扫描包，如果不添加扫描包，无法实现序列化和反序列化。另外待序列化的对象必须有一个默认的构造器，否则也无法实现序列化和反序列化。

***测试代码***

```java
public class Test1 {

    public static void main(String[] args) {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        RedissonClient client = Redisson.create(config);
        RMap<String, City> map = client.getMap("antMap", new FastjsonCodec());
        City city = City.builder().name("北京").build();
        map.computeIfAbsent("bj", k -> city);
        City city1 = map.get("bj");
        System.out.println(JSONObject.toJSONString(city1));
    }

}
```

由于Redisson默认使用的JsonJacksonCodec默认编码，当我们需要使用自定义编码时，我们可以通过类似 client.getMap("antMap", `new FastjsonCodec()`)这样使用自定义编码。执行测试方法，控制台返回如下信息：

```
{"name":"北京"}
```

此时说明自定义编码对于对象的序列化和反序列化正常。
