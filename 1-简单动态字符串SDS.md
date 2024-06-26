### 一、简单动态字符串SDS

+ **简要**：Redis底层默认使用SDS字符串。没有直接用到C语言字符串(以空字符结尾的--C字符串   在redis中没有修改字符串的会用到，比如打印日志)。其它关于被修改的值，就会是使用 `SDS` 
  + 补充：key 是字符串，值也是。倘如值是列表对象(其他也是一样)，那么该列表里面元素的值也是字符串，这些都是 `SDS`

+ 其他：`SDS`还用作缓冲区(buffer)：`AOF`缓冲区、客户状态的输入缓冲区
+ **`SDS`的定义**
  + ![image-20200911103336817](./imges/image-20200911103336817.png)
  + 

![image-20200911103513874](.\imges\image-20200911103513874.png)



### 二、**`SDS`与 C 字符串的区别**

+ 获取字符串长度 **复杂度** 不同

  + C 字符串本身不记录长度信息，C得长度用遍历。**复杂度：O(N)**
  + `SDS`字符串本身记录着 长度属性`len`。 **复杂度：O(1)**
    + `SDS`中关于长度得工作，是由 `SDS`中得`API`在执行时自动完成得，不需要对`SDS`的`len`属性做任何手动修改长度工作。即使很长，也不会因为取`len`的值造成系统性能任何影响。

+ **杜绝缓冲区溢出**

  + C容易造成缓冲区溢出。因为C 不记录长度。如果进行字符串拼接,没有分配足够空间，那么就用可能造成缓冲区 溢出问题--导致内容被修改，丢失等
    + ![image-20200911105813847](.\imges\image-20200911105813847.png)
  + `SDS`完全杜绝溢出：由于`SDS`的`API`中存在一个 `sdscat`拼接函数。该函数在拼接前会 先检查给定 `SDS`的空间是否足够，如果不够，该函数会先扩展`SDS`空间，然后在进行拼接。
    + ![image-20200911105658123](.\imges\image-20200911105658123.png)

+ **内存重分配次数**(修改字符串时引起)

  + C 语言中由于没有对字符串长度的记录，底层实现列表长度都是 N + 1 ,每次进行增加减少时，都需要对这个操作的字符串进行**内存重分配操作**。

  + `SDS`中则不同，因为在buf中的长度不一定时 N + 1；可能包含未使用的字节，由free记录者。在扩展`SDS`空间时，`SDS API`会先检查未使用空间够不够，不够需要重新分配空间，但是`redis`的内存分配策略非常银兴。

    + **空间预分配** --->减少连续执行字符串增长操作需要的内存重分配次数。

      ​	用于优化 `SDS`的字符串**增长操作**，当需要对空间进行扩展时，不会仅仅扩展必要的空间，还会为`SDS`分配额外的未使用空间。

      ​	下图为主要公式：

      ​				![image-20200911131325264](.\imges\image-20200911131325264.png)

      ​	

    + **惰性空间释放**

      ​	用于 优化 `SDS`的字符串**缩短操作**，当字符串缩短，程序不会立即收回多出来的字节，而是使用 `free` 属性将这些空闲记录起来，并等待使用。 --->  **避免了缩短带来的内存重分配的操作**，并为将来可能有的增长操作提供 优化。

      ​	注意：`SDS`中，也存在相应的`API`,当我们需要这些`free`记录的空余空间时，会真正的释放这些未使用的空间，，我们不用去担心**惰性空间释放策略**会造成内存浪费。

+ **二进制安全**

  + C 语言中，字符串的字符必须符合某种编码(比如 ASCII)，并且除了末尾之外，字符串里面不能包含 空字符。只能保存一些文本数据。
  + `Redis`能过适用于不用的适用场景，`SDS`的`API`都是二进制安全的，以处理二进制的方式存在在`buf`数组里的数据，不会对数据做任何的限制，过滤。

+ **兼容部分C字符串函数**

  + 虽然`SDS`中存的欧式二进制数据，但是一样遵循C字符串以空字符结尾的惯例：这些`API`总会将 `SDS`保存的数据末尾设置为 空字符，并且总会在`buf`数组分配空间时**多分配一个字节**来容纳这个 空字符，，这样的**目的**：保存文本数据的`SDS`可以重用一部分`<string.h>库定义的函数`

### 三、总结

虽然`Redis`是通过`C语言`来编写的，但是由于`Redis`利用自己的 `SDS`简单动态字符串，使得其于C字符串有明显的区别。

![image-20200911134102386](.\imges\image-20200911134102386.png)

+ `SDS API`

  ![image-20200911134340926](.\imges\image-20200911134340926.png)
  
  ![image-20200911134357452](.\imges\image-20200911134357452.png)
