### `AOF`持久化

+ **简要：** 除了使用`RDB`持久化功能之外，还存在`AOF(Append only File)`持久化功能。与`RDB`持久化通过保存数据库库的键值对来记录数据库状态不同，`AOF`持久化时通过保存`Redis`服务器**所执行的写命令**来记录数据库状态的。

  ![image-20200925160233434](.\imges\image-20200925160233434.png)

![image-20200925160407470](.\imges\image-20200925160407470.png)

+ **服务器载入**

  ![image-20200925160623549](.\imges\image-20200925160623549.png)

### 二、`AOF`持久化的实现

​	`AOF`持久化功能的实现存在**3个步骤**：

+ 1、**命令追加**（append）
+ 2、**文件写入**
+ 3、**文件同步**（sync）

#### 2.1 命令追加

​		当 `AOF`持久化功能处于**打开**的状态时，服务器在**执行完成一个写命令**之后，会按照**协议格式**将被执行的**写命令追加**到服务器状态的**`aof_buf`缓冲区的末尾**：

![image-20200925162614371](.\imges\image-20200925162614371.png)

![image-20200925162731471](.\imges\image-20200925162731471.png)

![image-20200925162747254](.\imges\image-20200925162747254.png)

#### 2.2 `AOF`文件的写入与同步

​	`Redis`服务器进程就是一个**事件循环(loop)**，这个循环中的**文件事件**负责接收客户端的命令请求，以及向客户端发送命令回复，而**时间事件**则负责执行向`serverCron`函数这样需要定时运行的函数。

​	服务器在处理文件事件时可能会执行写命令，使得一些内容被**追加到`aof_buf`缓冲区**里面，当服务器**每次结束一个事件循环之前，都会调用`flushAppendOnlyFile`函数**，考虑是否需要把缓冲区中内容写入和保存到`AOF`文件中，伪代码的实现：

伪代码：

```python
def eventLoop():
	
	while True:
		
        #处理文件事件，接收命令请求以及发送命令回复
        #处理命令请求时可能会有新内容被追加到 aof_buf缓冲区中
        processFileEvents()
        
        #处理时间事件
        processTimeEvents()
        
        #考虑是否 把 aof_buf 的内容写入和保存都 AOF 文件里面
        flushAppendOnlyFile()
```

其中，`flushAppendOnlyFile`函数的行为由服务器配置的 `appendfsync`选项的值来决定，各个不同值产生的行为如下表：

![image-20200925165532174](.\imges\image-20200925165532174.png)

​	如果`appendfsync`没有设置值，那么默认的 值为`everysec`

##### 疑问：是如何提高效率？

​		为了提高文件的写入效率，当用户调用 `write`函数，将一些数据写入到文件的时候，操作系统通常会将**写入数据暂时存在一个内存缓冲区**中，得到缓冲区的空间**被填满或者指定时限**到了，才真正的将数据写入到磁盘里。

​		该做法虽然提高效率，但是如果你运行的计算机停机了，那保存在内存缓冲区的数据将会丢失。因此，系统**提供两个同步函数`fsync`和`fdarasync`**。它们可以强制让系统立即将缓冲区的数据写入到硬盘中，从而保证数据安全。

#### 2.3 `AOF`文件的载入与数据还原

​	`AOF`文件里面包含了**重建数据库状态的所需要的所有写命令**，所有服务器只要读入并重新执行一遍`AOF`文件保存的写命令，就可以还原服务器关闭之前的数据库状态。

​	`Redis`中读取`AOF`文件的**载入和数据还原存在 4个步骤**：

+ 1、会创建一个不带网络连接的**伪客户端**(fake client)，没有网络连接：这是因为`Redis`命令只能在客户端上下文中执行，另外`AOF`文件所需要的命令直接来源`AOF`文件而不是网络连接。

+ 2、从`AOF`文件中分析并读取出一条写命令。

+ 3、使用伪客户端**执行被读写的命令**。

+ 4、重复执行 2、3两个步骤，指定`AOF`文件中所有写命令被处理完毕为止。
  如下图：

  + ![image-20200927093035870](.\imges\image-20200927093035870.png)

  例子：

  ![image-20200927093145706](.\imges\image-20200927093145706.png)

#### 2.4 `AOF`重写

+ 为什么要有`AOF`重写？

  + `AOF`持久化是通过保存被执行的写命令来记录数据库状态的，随着服务器运行，那么`AOF`文件的**内容会越来越多，文件体积也就越来越大**，过大会导致服务器、宿主机造成影响，另外`AOF`文件过大，那么还原数据的时间花费也很多。

    因此，`Redis`**提供`AOF文件重写(rewrite)功能`**，创建**新的`AOF`文件**来替代现有的`AOF`文件，新旧文件保存的数据库状态相同，但是**新文件不包含任何浪费空间**的冗余命令。

##### 2.4.1 如何实现`AOF`重写

​	`AOF文件重写` ：在实际上，并**不需要**对现有的`AOF`文件进行任何读取、分析或者写入操作，这个功能是**通过读取服务器当前的数据库状态来实现的**。

+ 案例1

![image-20200930090426152](.\imges\image-20200930090426152.png)

![image-20200930090608451](.\imges\image-20200930090608451.png)

其它的对象也是如此，

**伪代码实现重写功能：**

```python
def aof_rewrite(new_aof_file_name):
    
    #创建 新AOF文件
    f = create_file(new_aof_file_name)
    
    #遍历数据库
    for db in redisServer.db:
        
        # 忽略空数据库
        if db.is_emtry():
            continue
        
        #不为空，需要选择该数据库，新文件需要写入select命令+id
        f.write_command("SELECT" + db.id)
        
       	#遍历该数据库的所有键
        for key in db:
            #忽略过期的键
            if key.is_expired():
                continue
            #判断key的类型，进行对应类型的命令重写
            if key.type == String:
                rewrite_string(key)
            elif key.type == List:
                rewrite_list(key)
            elif key.type == Hash:
                rewrite_hash(key)
            elif key.type == Set:
                rewrite_set(key)
            elif key.type == SortedSet:
                rewrite_sorted_set(key)
            #r如果键带有过期时间，那么过期时间也是要重写的：
            if key.have_expire_time():
                rewirte_expire_time(key)
        #写入完毕，关闭文件
        f.close()
    
    def rewirte_string(key):
        #使用get命令获取字符串的值
        value = GET(key)
        f.wirte_command(SET,key,value) #使用SET命令写入
        
    def rewirte_list(key):
        #使用LRANGE命令获取列表的值
        item1,item2...itemN = LRANGE(key,0,-1)
        #使用RPUSH写入文件
        f.wirte_command(RPUSH,key,item1,item2...itemN) 
        
    def rewirte_hash(key):
        #使用hgetall命令获取全部哈希的值
        field1,value1,field2,value2...fieldN,valueN = HGETALL(key)
        #使用HMSET命令写入
        f.wirte_command(HMSET,key,field1,value1,field2,value2...fieldN,valueN) 
        
    def rewirte_set(key):
        #使用smembers命令获取所有的集合键的所有元素
        elem1,elem2...elemN = SMEMBERS(key)
        #使用SADD写入
        f.wirte_command(SADD,key,elem1,elem2...elemN) 
        
    def rewirte_sorted_set(key):
        #使用zrange命令获取有序集合的键包含所有元素
        member1,score1,member2,score2... = ZRANGE(key)
        #使用ZADD写入
        f.wirte_command(ZADD,key,member1，score1，member2,score2...) 
        
    def rewirte_expire_time(key):
        #获取毫秒精度的键过期时间戳
        timestamp = get_expire_time_in_unixstamp(key)
		#使用 pexpireat命令重写键的过期时间
        f.wirte_command(PEXPIREAT,key,timestamp) 

```

![image-20200930094004896](.\imges\image-20200930094004896.png)



##### 2.4.2 `AOF`后台重写

​	`Redis`中`AOF`重写的`aof_rewrite`函数会进大量的写入操作，当如果调用该函数的线程将被长时间阻塞，服务器使用单个线程来处理命令请求，如果调用该函数的话，那么服务器将长时间内无法处理客户端发来的命令请求。

​	因此，`Redis`不希望这样，将`AOF重写`程序放到子进程里执行，这样可以达到两个目的：

+ 1、子进程执行重写操作，服务器进程继续处理客户端的命令请求
+ 2、子进程带有服务器进程的数据副本，使用子进程而不是线程，可以避免使用锁的情况下，保证数据的安全性。

使用子进程也存在一个问题需要解决：

​		**问题：**子进程在冲洗期间，服务器还在处理客户端的命令请求，而新的命令可能对现有的数据库状态造成修改，那么会使得`AOF`重写保存的文件和当前服务器数据库状态不一致。

![image-20200930101421061](.\imges\image-20200930101421061.png)

​		**解决：**因此，`Redis`服务器设置了一个`AOF重写缓冲区`，这个缓冲区子啊服务器创建子进程之后开始使用，当服务器执行完一个命令之后，它会同时将这个写命令发送给`AOF缓冲区`和`AOF重写缓冲区`：
​		所以，在子进程进行`AOF重写期间`，服务器进程需要**执行3步**：
​				1、执行客户端发送的命令
​				2、将执行后的写命令追加到`AOF缓冲区`
​				3、将执行后的写命令追加到`AOF重写缓冲区`

​				![image-20200930101925736](.\imges\image-20200930101925736.png)

​		这样一来，可以保证：

​				1、`AOF缓冲区`的内容会定期被写入和同步到`AOF文件`，对现有的`AOF`文件的处理工作会如常进行。

​				2、从创建子进程开始，服务器所有执行写命令都会被记录到`AOF重写缓存区`里面。



**`AOF重写完成之后`：**

​	子进程完成重写操作之后，会向父进程发送一个信号，父进程接收到信号之后，会调用一个**信号处理函数**，执行工作如下：

​		1、将`AOF重写缓冲区`中所有内容写入到新`AOF文件中`，这时新`AOF`文件所保存的数据库状态将和服务器数据库状态一致的。

​		2、对新的`AOF文件`进行改名，原子地覆盖现有`AOF`文件，完成新旧两个`AOF`文件的替换。

所以：在整个`AOF`重写的过程中，只有信号处理函数会使服务器进程操作阻塞，在其它`AOF`重写都不会阻塞父进程，也让`AOF重写`对服务器性能影响降到了最低。

![image-20200930104048981](.\imges\image-20200930104048981.png)

### 总结

![image-20200930104329082](.\imges\image-20200930104329082.png)