

# 文件IO
## 引言
- 相关的操作
    - 打开文件 
    - 读文件
    - 写文件
    - 定位文件
    - 关闭文件

- 不带缓冲IO, 即每个read和write都调用内核中的一个syscall

## 文件描述符
#### 概念
- 内核对每个已经打开的文件用一个<font color = yellow>非负整数</font>来引用
- 属于当前进程的资源, 所以不同进程之间可以有相同的fd, 但它们指向的实际文件可能不是同一个
- 所有的IO操作通过fd来引用
- 惯例:
    - 进程打开的一般情况下,默认打开3个文件描述符
        - STDIN_FILENO
        - STDOUT_FILENO
        - STDERR_FILENO
        - 分别代表<font color = yellow>标准输入, 标准输出, 标准错误</font>
        - 对应的整数是<font color = yellow>0, 1, 2</font>
        - 头文件`<unistd.h>`中定义


## open
#### 函数原型
```cpp
#include<fcntl.h>
int open(const char* path, int oflag, ... /* mode_t mode*/);

int openat(int fd, const char* path, int oflag, ... /* mode_t mode */);
```

#### 参数
- **path**
    - 文件路径
        - 可以是相对路径
        - 也可以是绝对路径 

- **oflag**

|选项|说明|
|:-|:-:|
|O_RDONLY|只读打开|
|O_WRONLY|只写打开|
|O_RDWR|读写打开|
|O_EXEC|只执行打开|
|O_APPEND|以追加形式打开, 原子性|
|O_CLOEXEC|把FD_CLOEXEC常量设置为<font color = yellow>文件描述符标志</font>|
|O_CREAT|打开文件时, 不存在则创建|
|O_DIRECTORY|若不是目录则erroor|
|O_EXCL|若同时指定O_CREAT时, 文件存在则error|
|O_NONBLOCK|若文件是管道,块等设备文件, 则IO时非阻塞|
|O_SYNC|write等待物理IO完成,包括文件亚数据(修改时间等)|
|O_TRUNC|若文件存在, 则清空文件从0开始写|
|O_DSYNC|同O_SYNC, 但不等待文件亚数据更新就返回|
|O_TRUNC|若文件存在, 则清空文件从0开始写|

#### 返回值
- <font color = green>成功时</font>返回的fd一定是最小的未用的整数
> 如当前进程打开的fd有
> - 0
> - 1
> - 2
> - 4
> 调用fd成功后, 返回的是3

- <font color = green>失败时</font>返回-1, 并设置errno


#### O_CREAT
- 若是新建文件, 则必须指定 mode参数, 它用来控制权限
- 一个<font color = red>普通文件</font>被创建后, 它的默认权限是:
    - `rw-rw-r--`即`0664`
- 一个<font color = red>目录</font>被创建后, 它的默认权限是:
    - `rwxr-xr-x` 即`0755`
> 可以试着在shell下自己的home目录下创建查看一下, unix创建时赋于权限的算法是:
> mode & ~umask

#### case在代码中创建文件的正确姿势
```cpp
#include<stdio.h>
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>



int main(int args, char** argv){
    umask(0);       // 清空umask

    int RWRWRW = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH;

    // 打开屏蔽位 组读写:IRGRP, IWGRP, 其他读写:IROTH IWOTH
    umask(S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);       //__code_1
    
    // 这样创建出来的 文件 f.pdf的 权限是 rw- --- ---
    // 因为上面的umask设置了 对应的屏蔽位
    // 若 将 __code_1注释了, 则 umask所有的位都取消了屏蔽, 此时创建出来的
    // f.pdf的权限就是 rw- rw- rw-
    open("./f.pdf", O_CREAT | O_RDWR, RWRWRW);
    return 0;
}
```
> 这里umask并不关心执行位x


#### case(判断文件是否存在)
- 不存在则创建, 整个过程是原子操作
```cpp
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<string.h>
#include<errno.h>
#include<iostream>

using namespace std;

int main(int args, char** argv){
    umask(0);       // 清空umask

    int RWRWRW = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH;
    
    // O_CREAT 和 O_EXCL 一起用
    // 若 f.pdf存在, 则报错, 若不存在则直接创建该文件
    auto result = open("./f.pdf", O_CREAT | O_EXCL | O_RDWR, RWRWRW);
    cout << "结果:" << result << endl;
    if(result == -1){
        cerr << result << "err:" << strerror(errno) << endl;
    }
    return 0;
}
```

#### openat
- 若path是绝对路径, fd被忽略
- 若path是文件, fd被忽略
- 若path是相对路径, 则相对的是fd(目录)
```cpp
// ./tmp/f.pdf
// ./tmp/dir/f.pfd

__auto fd = open("./tmp/dir");
openat(fd, "f.pdf", xxx);        // ./tmp/dir/f.pfd
```

## close
#### 函数原型
```cpp
#include<unistd.h>
int close(int fd);
```
> 返回-1,表示失败并设置errno, 返回0,表示成功
> 一个进程可以有不同的fd去关联同一个文件, 关闭当前的fd,并不会影响其他已经指向同一文件的fd, 但所有的fd都关闭后, 会释放该进程对这个文件所属的文件锁, 当一个进程终止时, 也会自动关闭fd

## lseek
#### 说明
- 每个fd都有自己的<font color = yellow>当前文件偏移量</font>
    - 它<font color = yellow>不是文件系统的指针偏移量</font>

#### 函数原型
```cpp
#include<unistd.h>
off_t lseek(int fd, off_t offset, int where);

// 成功, 返回 fd当前偏移量
// 失败, 返回-1, 并设置errno
```

#### 参数
- where可以是:
    - <font color = red>SEEK_SET</font>
        - 设置到 距离文件开头 offset 个字节处
    - <font color = red>SEEK_CUR</font>
        - 以距离当前位置为基点, offset 个字节处
    - <font color = red>SEEK_END</font>
        - 以文件结尾处, offset个字节处

#### case 检查文件能否设置偏移量
```cpp
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<string.h>
#include<errno.h>
#include<iostream>

using namespace std;

int main(int args, char** argv){
    if(lseek(STDIN_FILENO, 0, SEEK_CUR) == -1){
        cerr << "当前文件不支持偏移\n";
        return -1;
    }
    cout << "设置ok\n";
    return 0;
}
```

> 使用交互式的来测试
```shell
make main       # 编译
./main < /etc/passwd    # shell 重定向了stdin, 从文件中获取 
设置ok                   # 程序输出, 表示文件可以lseek

./main < ./FIFO         # 当前路径下有1个管道文件
当前文件不支持偏移         # 程序打印, 表示管道文件不可以lseek


cat /etc/passwd | ./main    # main的stdin从前面 cat的执行获取, 本质上还是标准的 stdin
当前文件不支持偏移           # 标准输入不支持设置lseek
```

#### case获取文件大小(字节)
```cpp
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<string.h>
#include<errno.h>
#include<iostream>

using namespace std;

int main(int args, char** argv){
    auto fd = open("./f.txt", O_RDONLY);
    if(fd == -1){
        cerr << strerror(errno) << endl;
        return -1;
    }

    auto count = lseek(fd, 0, SEEK_END);
    if(count == -1){
        cerr << strerror(errno) << endl;
        return -1;
    }

    cout << "文件所占的字节是:" << count << endl;

    return 0;
}
```
> 测试
```shell
// f.txt 的内容是 "我c"
l f.txt
-rw-rw-rw-  1 liubo  staff     5B  8 19 15:33 f.txt

./main 
文件所占的字节是:5
```

#### 空洞文件
- lseek的偏移量可以大于文件的当前长度, 这种情况下, 文件系统会构造成一个空洞, 空洞的位置全是0
- 空洞文件并不占据磁盘空间, 这由OS来处理不用关心
- 空洞文件的使用场景
    - 如下载一个4G电影, 预先分配一个4G的空间用来存, 然后分段多线程下载, 填写到对应的区段

#### lseek的offset
- offset偏移的是字节, 若读取的内容中有中文字符, 要特别注意
```cpp
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<string.h>
#include<errno.h>
#include<iostream>

using namespace std;

#define BUF_SIZE (1024)

int main(int args, char** argv){

    if(args != 2){
        cerr << "usage: main file-path\n";
        return -1;
    }

    __auto_type fd = open(argv[1], O_RDONLY);
    if(fd == -1){
        cerr << "open fail() \n";
        return -1;
   }

   __auto_type lres = lseek(fd, 1, SEEK_CUR);
   char buf[BUF_SIZE] = {'0'};
   __auto_type r_res = read(fd, buf, BUF_SIZE);
    cout << buf << endl;
   return 0;
}

/**
    读取的文本的内容是:
        "我是一个中国人"
    程序运行时,在terminer中输出的结果是:
        ��是一个中国人
    
    
    可以发现, 当读取的是中文时, seek设置的偏移1个字节是有问题的
    因为 1个中文字符占据的空间不只1个字节, 这种情况下要考虑当前读取文件
    的编码, 这时候就特别复杂了
*/
```



## read
#### 函数原型
```cpp
#include<unistd.h>
ssize_t read(int fd, void* buf, size_t nbytes);

// 成功返回 实际写入的字节数目
// 失败返回 -1, 并设置errno
// 读取到文件未尾时, 返回0
```

#### case读取次数的一个小细节
```cpp
#include<unistd.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<string.h>
#include<errno.h>
#include<iostream>
using namespace std;
int main(int args, char** argv){

    if(args != 2){
        cerr << "usage: main file-path\n";
        return -1;
    }

    __auto_type fd = open(argv[1], O_RDONLY);
    if(fd == -1){
        cerr << "open fail() \n";
        return -1;
   }

   char buf[6] = {0};
   int count = 0;
   while(1){
       count = read(fd, buf, 5);        // __code_1

       if(count == -1){
           cerr << "read error\n";
           return -1;
       }
        cout << "ok\n";
        cout << count << endl;

        if(count == 0)                  // __code_2
            break;
   }
   return 0;
}
```

> 测试的读取次数:(文件的大小是5个字节)

|`__code_1`设置的值|读取的次数|
|:-:|:-:|
|5|2次|
|4|3次|
|6|2次|

> 即unix对文件读取完毕时, 作单独的一次返回, 需要指出的是在linux中, f.pdf文件中的数据实际是6个字节, 最后会多出一个换行, 所以同样的设置为5时, 读取次数是3次, 第2次读取的是换行,第3次是完毕
> `__code_2`这里不能使用`if(count == EOF)`来当作读取结束, 函数原型已经说得很清楚了, 读取完毕后, 返回的是0, <font color = green>EOF是标准IO库中使用的</font>


## write
#### 函数原型
```cpp
#include<unistd.h>
ssize_t write(int fd, const void* buf, size_t nbytes);
// 成功返回已写的字节数, 并且 == nbytes
// 失败返回 -1, 并设置errno,  写入的字节 != nbytes也出错
```

#### append模式
- 当文件已append模式写时, lseek则失去作用, 每次写时追加到文件末尾(原子写入)
```cpp
#include<iostream>
#include<cstdlib>
#include<cstring>
#include<unistd.h>
#include<fcntl.h>
#include<cerrno>

using namespace std;

int main(int args, char** argv){
    if(args < 4){
        cerr << "usage ./append <file_path> <world> <world>\n";
        exit(1);
    }
   auto fd = open(argv[1], O_RDWR | O_APPEND | O_TRUNC | O_CREAT, S_IWUSR | S_IRUSR);
   if(fd < 0){
       cerr << "open():" << strerror(errno) << endl;
       exit(1);
   }

   auto log_str = argv[2];
   if(write(fd, log_str, __builtin_strlen(log_str)) < 0){
       cerr << "write err:" << strerror(errno) << endl;
       exit(1);
   }
   if(lseek(fd, SEEK_SET, 0) < 0){
        cerr << "seek err:" << strerror(errno) << endl;
        exit(1);
   }

   log_str = argv[3];
   if(write(fd, log_str, __builtin_strlen(log_str)) < 0){
       cerr << "write err:" << strerror(errno) << endl;
       exit(1);
   }
}
```
> 测试
```shell
make append     # 编译
./append b.txt "abc" "hello world"
cat b.txt
abchello world  # 代码中输出abc后, 调整了lseek, 但没有效果, "hello world"还是被追加在了"abc"后面
```













