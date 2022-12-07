# 前置知识

## Serializable、Externalizable

[Java 序列化和反序列化详解完整版]: https://blog.csdn.net/huangtenglong/article/details/12777454

```java
public interface Externalizable extends java.io.Serializable {

    void writeExternal(ObjectOutput out);

    void readExternal(ObjectInput in);
}
```

## java反序列化

反序列化过程发生在java.io.ObjectInputStream#readOrdinaryObject方法内

### 整体流程

1、从字节流中获取类描述符。

2、通过类描述符获取类对象。

3、判断类对象是否可实例化，进而实例化类对象。

4、判断类对象的序列化接口类型（Serializable or Externalizable）进行赋值。

5、判断类对象是否有readReslove方法。

![1](http://xxlegend.com/images/SequenceDiagram1.png)  

### 步骤一：从输入流获取类描述符

```
ObjectStreamClass desc = readClassDesc(false);
```

 ![./](D:/Desktop/%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/Weblogic/Assets/2-Weblogic%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E7%9F%A5%E8%AF%86%E7%82%B9/1.png)

### 步骤二：通过类描述符获取类名

```
Class<?> cl = desc.forClass();
```

### 步骤三：判断类对象是否可实例化

```
Object obj;
obj = desc.isInstantiable() ? desc.newInstance() : null;
```

### 步骤四：判断类对象的序列化接口类型

```
if (desc.isExternalizable()) {
    readExternalData((Externalizable) obj, desc);
} else {
    readSerialData(obj, desc);
}
```

关于readObject、readResolve、readExternal等方法的调用

 ![](D:/Desktop/%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/Weblogic/Assets/2-Weblogic%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E7%9F%A5%E8%AF%86%E7%82%B9/2.png)

### 步骤五：判断是否存在readResolve方法

```
desc.hasReadResolveMethod()
```

### 代码实现

```java
private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_OBJECT) {
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);//获取类描述符
        desc.checkDeserialize();

        Class<?> cl = desc.forClass();//获取类对象
        if (cl == String.class || cl == Class.class
                || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
            //判断类是否可实例化，进而生成实例对象
            obj = desc.isInstantiable() ? desc.newInstance() : null;
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }
		
    	//判断类的序列化接口，调用不同的方法对实例对象赋值
        if (desc.isExternalizable()) {
            readExternalData((Externalizable) obj, desc);
        } else {
            readSerialData(obj, desc);
        }

        handles.finish(passHandle);
		
    	//判断类是否有readReslove方法，没有直接返回obj对象；有的话使用其内部方法生成rep对象并赋值给obj返回
        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
    }
```

# Weblogic反序列化过程

# Weblogic T3协议

# 参考

https://xz.aliyun.com/t/8443

