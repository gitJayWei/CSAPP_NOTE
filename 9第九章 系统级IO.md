## 9. 系统级I/O

- 输入/输出(I/O)就是在主存和外部设备之间复制数据的过程；(常见的外部设备有固态硬盘、终端和网络等)；
  - 输入指数据从设备流向主存，输出则相反；
- 各种编程语言提供的高级IO工具 (如C的printf/scanf，C++的<</>>) ，在Linux系统中最终都是通过**系统级的IO函数**实现的；



### 9.0. 前置知识

#### 9.0.0. 文件

##### 文件类型

- Linux有多种文件类型，具体包括：普通文件、目录、套接字、符号链接、块设备、字符设备、命名管道等；

  > 本章聚焦于**普通文件**和**目录文件**，套接字会在后面有关网络的章节讲解，其它暂不接触；

- 普通文件：又分为文本文件与二进制文件，不过对于内核来说，二者都是01序列，没有区别；

  > 文本文件就是能被逐字节解释为字符的文件，而二进制文件的字节不是可读字符，用文本编辑器打开是乱码 (比如音频文件，图片文件，目标文件等等)；

- 目录文件：也是存储在磁盘上的，只不过存的是**文件名到inode号的映射关系**；

  ~~~c
  // 假设目录/home/jay/里面有两个普通文件note.txt、image.png与一个目录文件projects/
  // 那目录文件/home/jay/实际存的是
  "."          ->  inode X (目录自己)
  ".."         ->  inode Y (父目录)
  “note.txt”   ->  inode 2019
  "image.png"  ->  inode 2017
  "projects"   ->  inode 2023
  ~~~

##### 访问文件

- Linux文件系统是一个以根目录 ‘ / ’ 开始的目录层级结构；

- 目录文件存储的文件名到inode号的映射：
  - 可以将多个文件名映射到同一个inode号；
  - 通过文件名访问文件时，无论是相对路径还是绝对路径，最终都能获取到该文件名对应的inode号；
- 磁盘被分成许多区域，其中两个区域为 inode table 与 data blocks；
  - inode table类似于一个大数组，该数组每项是一个inode structure结构体，包含文件元数据和指向文件具体位置的指针；(所以文件元数据和文件本身是分开的)
  - data blocks就是文件实际存放的区域；
- 所以拿到inode号后，将其作为索引，到inode table中拿到具体的inode structure，也就获得了文件的元数据和文件的位置；

> 所以删除文件并不需要擦除磁盘上的数据块，只需要删除目录文件中的对应关系，当没有文件名与该inode号对应时，相关机制会回收这些数据块；



#### 9.0.1. Unix IO

- Linux中，每个文件都被视为一个字节序列；所有的IO设备也被抽象为文件，这使得所有IO操作都可以通过统一的文件读写方式进行处理；这种设计允许内核提供一个简单的应用接口，即Unix IO，从而简化了IO管理；

  > 这种“一切皆文件”是通过虚拟文件系统 (VFS) 实现的

  - 进程打开一个文件，内核会返回一个文件描述符(非负整数)，进程通过该描述符来引用文件；每个进程都默认会打开三个标准文件：标准输入(描述符0)，标准输出(描述符1)，标准错误(描述符3)；
  - 每个打开的文件，内核会维护一个文件位置(初始值为0)，表示从文件开头的字节偏移量，可通过seek操作修改；
  - 当程序执行读操作，如果试图读超过文件大小的字节数，内核会返回一个EOF指示；





### 9.1. 核心操作

> 下文中使用的首字母大写的函数是对原系统函数封装后的安全版本，差异是出错时会自动处理；

#### 9.1.0. 打开关闭文件

##### open()

- 进程通过调用open函数来打开一个已有文件或创建一个新文件；

~~~c
int open(char* filename, int flags, mode_t mode);
// 成功返回当前进程没有使用的最小描述符，失败返回 -1
~~~

- 参数filename：文件名；

- 参数flags：指明进程访问文件的方式；

  ~~~c
  // 下面的掩码是系统预定义的一系列二进制位不重叠常量，可用|运算符进行组合
  // 什么是二进制位不重叠常量？ 假设O_RDONLY是0001，O_WRONLY是0010，O_RDWR是0100...
  
  // O_RDONLY: 只读
  // O_WRONLY: 只写
  // O_RDWR: 可读可写
  // O_CREAT: 如果文件不存在，就创建一个新的空文件
  // O_TRUNC: 如果文件巳经存在，就清空它
  // O_APPEND: 每次写操作前，设置文件位置到文件结尾
  
  // eg.
  fd = Open("foo.txt", O_WRONLY|O_APPEND, 0); // fd即返回的描述符
  ~~~

- 参数mode：指定新文件的访问权限；

  - mode只会在创建新文件时设置权限 (即flags参数有O_CREAT)，打开已有文件时mode参数无效；
  
  ~~~c
  // 每个进程都有一个umask，用于屏蔽参数mode指定的部分权限(具体为：mode & ~umask)
  // 可通过umask函数主动设置umask，若不，系统会使用默认umask屏蔽组用户和其他用户的写权限
  
  // 权限位常量，也是一系列二进制位不重叠常量
  // S_IRUSR：使用者（拥有者）能够读这个文件
  // S_IWUSR：使用者（拥有者）能够写这个文件
  // S_IXUSR：使用者（拥有者）能够执行这个文件
  // S_IRGRP：拥有者所在组的成员能够读这个文件
  // S_IWGRP：拥有者所在组的成员能够写这个文件
  // S_IXGRP：拥有者所在组的成员能够执行这个文件
  // S_IROTH：其他人（任何人）能够读这个文件
  // S_IWOTH：其他人（任何人）能够写这个文件
  // S_IXOTH：其他人（任何人）能够执行这个文件
  
  // 缩写解释 R = Read（读权限）W = Write（写权限）X = eXecute（执行权限）
  // USR = User（文件所有者 / 使用者）GRP = Group（所有者所在的组）OTH = Others（其他用户，即任何人）
  
  #define DEF_MODE S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH // 所有用户都有读写权限
  #define DEF_UMASK S_IWGRP|S_IWOTH // 待屏蔽的权限(组用户和其他用户的写权限，其实就是默认会被屏蔽的权限)
  
  umask(DEF_UMASK); // 主动设置umask
  fd = Open("foo.txt", O_CREAT|O_TRUNC|O_WRONLY, DEF_MODE); 
  ~~~

##### close()

~~~c
// 进程通过close函数来关闭一个已打开的文件
int close(int fd); // 这里也可看出进程只需要通过描述符区分文件
~~~



#### 9.1.1. 读和写文件

##### read()与write()

~~~cpp
// 程序通过分别调用read和write函数来执行输入和输出

ssize_t read(int fd, void* buf, size_t n); 
// read 函数从描述符为 fd 的 当前文件位置 复制最多n个字节到内存位置buf
// 若成功返回读的字节数，若EOF返回0，若出错返回-1

ssize_t write(int fd, void* buf, size_t n); 
// write 函数从内存位置buf复制至多n个字节到描述符fd的当前文件位置
// 若成功返回写的字节数，若出错返回-1

// ssize_t被定义为long
~~~

- 示例

  ~~~c
  char c;
  while (Read(STDIN_FILENO, &c, 1) != 0) 
      Write(STDOUT_FILENO, &c, 1);
  
  // 作用：逐字节从标准输入复制到标准输出
  // 解释：STDIN_FILENO是标准输入文件的描述符，只要不EOF，就一直读取
  // 		STDOUT_FILENOs是标准输出文件的描述符
  // 		变量c相当于temp
  // 也说明两个函数都会更新 当前文件位置
  ~~~

- read和write返回**不足值**的情况：

  - 遇到EOF：当读取文件至末尾时，read返回实际读取的字节数 (称为不足值)，下一次调用则会返回0；
  - 终端读取：从终端读取时，read默认一次返回一行的文本内容，其返回值即为该文本行的大小；
  - 读写网络套接字：由于网络延迟或内部缓冲区限制，对网络套接字的读写操作也经常返回不足值；

  > 程序必须要能容忍不足值，而附1 RIO包提供的函数能自动处理不足值；
  
  

#### 9.1.2. 读取文件元数据

##### stat()与fstat()

- 元数据就是关于文件的信息，比如文件的大小，访问权限，类型等；

  ~~~c
  // 程序可以通过stat和fstat函数获取文件的元数据
  
  int stat(const char *filename, struct stat *buf);
  // 这个函数接收一个文件名，然后把元数据填充到struct stat类型的buf结构体中
  
  int fstat(int fd, struct stat *buf);
  // 与stat函数功能类似，不过它接收的是文件描述符fd ，而不是文件名
  ~~~

  - struct stat结构体成员：

    ~~~c
    st_dev     // 表示文件所在的设备
    st_ino     // 标识文件在文件系统中的唯一索引
    st_mode    // 记录文件的访问权限和文件类型 (见下方注释)
    st_nlink   // 文件的硬链接数
    st_uid     // 文件所有者的用户 ID
    st_gid     // 文件所有者的组 ID
    st_rdev    // 若为设备文件，标识设备类型
    st_size    // 文件的字节大小
    st_blksize // 文件系统 I/O 的块大小
    st_blocks  // 文件占用的块数量
    st_atime   // 文件最后一次被访问的时间
    st_mtime   // 文件最后一次被修改的时间
    st_ctime   // 文件状态最后一次变更的时间
        
    // 注：st_mode也是用不同二进制位记录文件类型和访问权限
    // 提取访问权限：由于其记录访问权限的二进制位与前面的权限位常量是对应的，所以st_mode&S_IRUSR为真就表示该文件使用者可读，st_mode&S_IWUSR为真就表示该文件使用者可写等等，以此类推
    // 那又怎么从st_mode中提取文件类型呢？Linux定义了一些宏谓词，用于提取st_mode中文件类型相关位(所谓宏谓词就是预定义的位运算工具)
    // S_ISREG(st_mode)：判断是否是普通文件；S_ISDIR(st_mode)：判断是否是目录文件；S_ISSOCK(st_mode)：判断是否是网络套接字；等等
    ~~~
    

- 示例代码

  ~~~c
  int main (int argc, char **argv)
  {
      struct stat stat; // struct stat结构体，用于存储文件元数据
      char *type, *readok; // 记录文件类型和文件是否可读
  
      Stat(argv[1], &stat); // 获取文件元数据
  
      // 判断文件类型
      if (S_ISREG(stat.st_mode))      type = "regular";
      else if (S_ISDIR(stat.st_mode)) type = "directory";
      else                            type = "other";
  
      // 判断该文件使用者是否可读
      if ((stat.st_mode & S_IRUSR))  readok = "yes";
      else                           readok = "no";
  
      printf("type: %s, read: %s\n", type, readok); // 给出结果
      exit(0);
  }
  ~~~
  
  

#### 9.1.3. 读取目录内容

##### readdir系列函数

- ~~~c
  DIR *opendir(const char *name); 
  // 参数name: 路径(单个文件名就是相对路径)
  // 返回值DIR*: 指向目录流的指针
  ~~~

  - 目录流：将磁盘上的目录文件封装成一个可以顺序读取的流；它会记录你读取到的位置；它也会缓存一批目录项到内存；

- ~~~c
  struct dirent *readdir(DIR *dirp);
  // 参数drop: 目录流指针，来源于opendir的返回值
  // 返回值 struct dirent*: 返回流dirp中下一个目录项的指针，若没有了就返回NULL
  
  // 值得注意的是，如果出错了readdir也是返回NULL，但是会设置errno
  // 所以分辨 流结束or错误，就是检查readdir调用后errno有没有被修改
  ~~~

  - struct dirent结构体：

    ~~~c
    struct dirent {
    	ino_t d_ino;		// 文件的inode号
        char d_name[256];	// 文件名
    }
    ~~~

- ~~~c
  int closedir(DIR *dirp);
  // 关闭流dirp并释放其所有的资源
  ~~~

- 示例程序：

  ~~~c
  int main(int argc, char **argv)
  {
      DIR *streamp;       // 目录流指针，指向打开的目录
      struct dirent *dep; // 目录项结构体指针，存储单个目录项信息
  
      streamp = Opendir(argv[1]); // 打开指定的目录
  
      errno = 0; // 重置全局错误码，避免之前的错误干扰
      
      // 循环读取目录项：readdir返回NULL时结束（正常结束或出错）
      while ((dep = Readdir(streamp)) != NULL)
          printf("%s\n", dep->d_name); // 输出当前目录项的文件名
      
      // 检查循环结束的原因：是流结束，还是出错了
      if (errno != 0)
          unix_error("Readdir error"); // 出错则输出错误信息
      
      Closedir(streamp); // 关闭目录流，释放资源
      exit(0);           // 程序退出
  }
  ~~~





### 9.2. 其它

#### 9.2.0. 共享文件

##### 内核如何表示打开的文件

- Linux系统中，内核通过三个数据结构来表示打开的文件；

1. 描述符表 (descriptor table)

   - 每个进程都有独立的描述符表，索引为进程打开的文件描述符；
   - 每个打开的描述符表项指向文件表中的一个表项；
1. 文件表 (file table)

   - 用于表示系统中打开的文件的集合，所有进程共享这张表；

   - 表项组成：

     - 当前文件位置：记录文件读写操作的当前位置；

     - 引用计数：表示指向该表项的 描述符表项 的数量；引用计数为零时，系统会删除该文件表表项；

     - 指向 v - node表中对应表项的指针；
1.  v - node表
   - 所有进程共享；
   - 每个表项包含stat结构的大多数信息，例如st_mode和st_size；

##### 不同的文件共享情况

1. 无文件共享：

   <img src="./assets/9.8.2. 无文件共享.png" alt="9.8.2. 无文件共享示例" style="zoom: 40%;" />

   - 描述符1和4分别索引描述符表中的两个表项，这两个表项又分别指向文件表中的两个表项，这两个文件表表项又分别指向v - node 表的两个表项，它们之间没有共享关系；

1. 多个描述符引用同一个文件：

   <img src="./assets/9.8.2. 多个描述符引用同一个文件.png" alt="9.8.2. 多个描述符引用同一个文件.png" style="zoom:50%;" />

   - 如果用同一个filename调用open函数两次就会出现上图情况，即多个描述符引用了同一个文件；虽然描述符不同，但是文件表表项都指向了v - node表的同一个表项，说明文件元数据相同；值得一提的是由于描述符不同导致对不同描述符的读操作可以从文件的不同当前文件位置获取数据；

1. 父子进程共享文件

   <img src="./assets/9.8.2. 父子进程共享文件.png" alt="9.8.2. 父子进程共享文件.png" style="zoom:50%;" />

   - 假设在fork之前，父进程就有打开的文件(如图10 - 12)，而fork后(图10 -14)，子进程会有一个父进程描述符表的副本，并且相同的描述符指向的文件表表项也是相同的，这意味着父子进程相同的描述符共享相同的当前文件位置等；只有父子进程都关闭了对应的描述符，使得文件表表项的引用计数归零，内核才会删除相应的文件表表项；



#### 9.2.1. IO重定向

- LInux shell有I/O重定向操作符

  ~~~cpp
  linux> ls > foo.txt
  ~~~

  - shell将加载和执行ls程序，将标准输出重定向到磁盘文件foo.txt；

- dup2函数实现I/O重定向

  ~~~cpp
  #include <unistd.h>
  int dup2(int oldfd, int newfd); // 成功返回描述符，错误返回-1
  ~~~

  - dup2函数将oldfd索引的描述符表表项复制到newfd索引的，直接覆盖；如果newfd已经打开了，dup2函数会在复制oldfd之前关闭newfd；(注意是谁复制给谁)；

  - 假设在调用dup2(4, 1);之前，系统状态为图10 - 12，此时文件A和B的索引计数都为1；下图为调用了dup2(4, 1);后的系统状态，现在两个描述符都引用了文件B，文件B的引用计数为2，而文件A已被内核关闭；从此以后，所有写到标准输出的数据都将被重定向到文件B；

    <img src="./assets/9.9. dup2函数实现重定向.png" alt="9.9. dup2函数实现重定向.png" style="zoom: 33%;" />





### 附1 RIO包

- 类似于高级语言中的IO语句，RIO(Robust I/O)包中的函数也是基于Unix IO，而且在健壮性上做了提升，比如它会自动处理不足值；RIO提供了两类函数：
  - 无缓冲的输入输出函数：这些函数直接在内存和文件之间传送数据，没有应用级缓冲；
  - 带缓冲的输入函数：使用缓冲区来提高数据读取效率；

##### RIO的无缓冲输入输出函数

- 函数声明

  ~~~cpp
  ssize_t rio_readn(int fd, void *usrbuf, size_t n);
  	// 成功返回传送的字节数，EOF返回0，出错返回-1
  ssize_t rio writen(int fd, void *usrbuf, size_t n);
  	// 成功返回传送的字节数，出错返回-1
  ~~~

  - 对同一个描述符，可以任意交替地调用rio_readn和rio_writen

- rio_readn函数

  函数功能：从文件描述符fd指向的文件中读取最多n个字节到usrbuf指向的内存区域

  ~~~cpp
  ssize_t rio_readn(int fd, void *usrbuf, size_t n) {
      size_t nleft = n; // 记录还需要读取的字节数，初始化为n
      ssize_t nread;    // 用于存储每次实际读取的字节数
      char *bufp = usrbuf; // 指向用户提供的用于存储数据的缓冲区
  
      while (nleft > 0) { // 只要还有字节需要读取
          if ((nread = read(fd, bufp, nleft)) < 0) { // 调用read系统调用读取数据
              if (errno == EINTR) { 
                  // 如果是因为信号处理程序中断导致读取失败
                  // 则将nread设为0，准备重新调用read
                  nread = 0; 
              } else {
                  // 其他错误情况，直接返回-1表示读取失败
                  return -1; 
              }
          } else if (nread == 0) {
              // 遇到文件结束符（EOF），跳出循环
              break; 
          }
          nleft -= nread; // 更新剩余需要读取的字节数
          bufp += nread;  // 移动缓冲区指针到已读数据之后
      }
      return (n - nleft); // 返回实际成功读取的字节数
  }
  ~~~

- rio_writen函数

  函数功能：将usrbuf指向的内存区域中的n个字节数据写入到文件描述符fd指向的文件中

  ~~~cpp
  ssize_t rio_writen(int fd, void *usrbuf, size_t n) {
      size_t nleft = n; // 记录还需要写入的字节数，初始化为n
      ssize_t nwritten; // 用于存储每次实际写入的字节数
      char *bufp = usrbuf; // 指向用户提供的要写入数据的缓冲区
  
      while (nleft > 0) { // 只要还有字节需要写入
          if ((nwritten = write(fd, bufp, nleft)) <= 0) { // 调用write系统调用写入数据
              if (errno == EINTR) { 
                  // 如果是因为信号处理程序中断导致写入失败
                  // 则将nwritten设为0，准备重新调用write
                  nwritten = 0; 
              } else {
                  // 其他错误情况，直接返回-1表示写入失败
                  return -1; 
              }
          }
          nleft -= nwritten; // 更新剩余需要写入的字节数
          bufp += nwritten;  // 移动缓冲区指针到已写字节之后
      }
      return n; // 返回要写入的字节数，表示写入成功
  }
  ~~~

  - 每打开一个描述符，都会调用一次rio_readinitb函数，将描述符fd和地址rp处的一个类型为rio_t的读缓冲区联系起来；
  - rio_readlineb函数从文件rp读出下一个文本行(包括结尾的换行符)，将它复制到内存位置usrbuf，并且用NULL来结束这个文本行；rio_readlineb函数最多读maclen-1个字节(结尾要填充一个NULL)；超过maxlen-1的文本行会被截断(结尾一样填充一个NULL)；
  - rio_readnb函数从文件rp最多读n个字节到内存位置usrbuf；对同一个描述符，rio_readlineb和rio_readnb的调用可以任意交叉进行；然而，带缓冲区的函数的调用不应该和无缓冲的rio_readnb函数交叉进行；

- 这两个函数都通过循环调用`read`或`write`系统调用进行数据的读写操作，并且都对系统调用因信号中断（`EINTR` 错误 ）的情况进行了特殊处理，手动重启相应的系统调用，以增强函数在不同环境下的健壮性和可移植性。`rio_readn` 函数主要用于读取文件数据到内存，`rio_writen` 函数主要用于将内存中的数据写入文件；

##### RIO的带缓冲的输入函数

- rio_t结构体

  ~~~cpp
  // 定义读缓冲区大小为8192字节
  #define RIO_BUFSIZE 8192 
  
  // 定义rio_t结构体来管理读缓冲区
  typedef struct 
  {
      int rio_fd;        // 与该内部缓冲区关联的文件描述符
      int rio_cnt;       // 内部缓冲区中未读的字节数
      char *rio_bufptr;  // 内部缓冲区中下一个未读字节的指针
      char rio_buf[RIO_BUFSIZE]; // 内部缓冲区数组
  } rio_t;
  ~~~

- rio_readinitb

  初始化读缓冲区，将打开的文件描述符与缓冲区关联起来

  ~~~cpp
  void rio_readinitb(rio_t *rp, int fd) 
  {
      rp->rio_fd = fd;  // 将传入的文件描述符赋值给结构体成员
      rp->rio_cnt = 0;  // 初始化时，缓冲区中未读字节数为0
      rp->rio_bufptr = rp->rio_buf; // 初始化下一个未读字节指针，指向缓冲区起始位置
  }
  ~~~

- rio_readlineb

  从与rio_t结构体rp关联的读缓冲区中读取一行文本，存储到usrbuf指向的内存区域(最多读maxlen-1个字节，预留一个字节用于存储字符串结束符'\0')；

  ~~~cpp
  ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen) {
      int n, rc; 
      // n用于记录已经读取的字节数（不包含结束符'\0'），rc用于存储rio_read函数的返回值
      char c, *bufp = usrbuf; 
      // c用于暂存每次从缓冲区读取的单个字符，bufp指向用于存储文本的内存位置
  
      // 循环，在已读取字节数小于maxlen时，持续尝试读取
      for (n = 1; n < maxlen; n++) { 
          // 从读缓冲区中读取一个字符到c中
          if ((rc = rio_read(rp, &c, 1)) == 1) { 
              // 将读取到的字符存储到bufp指向的位置，并移动bufp指针
              *bufp++ = c; 
              // 检查字符是否为换行符'\n'，如果是
              if (c == '\n') { 
                  n++; // 包含换行符，n自增
                  break; // 跳出循环，已读取到一行文本
              }
          } 
          // 如果遇到文件结束符（EOF）
          else if (rc == 0) { 
              // 若此时还未读取到有效数据（n == 1）
              if (n == 1) 
                  return 0; // 返回0，表示EOF且无有效数据读取
              else 
                  break; // 若已读取到部分数据，跳出循环
          } 
          // 如果读取出错
          else 
              return -1; // 返回-1，表示读取错误
      }
      *bufp = 0; // 在存储文本的末尾添加字符串结束符'\0'
      return n - 1; // 返回实际读取的字节数（不包含结束符）
  }
  ~~~

- rio_readnb

  从与rio_t结构体rp关联的读缓冲区中读取n个字节的数据，存储到usrbuf指向的内存区域

  ~~~cpp
  ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n) {
      size_t nleft = n; 
      // nleft用于记录还需要读取的字节数，初始化为n
      ssize_t nread; 
      // nread用于存储每次实际读取的字节数
      char *bufp = usrbuf; 
      // bufp指向用于存储读取数据的内存位置
  
      // 循环，在还有字节需要读取（nleft大于0）时，持续尝试读取
      while (nleft > 0) { 
          // 从读缓冲区中读取数据
          if ((nread = rio_read(rp, bufp, nleft)) < 0) 
              return -1; // 读取出错，直接返回-1
          // 如果遇到文件结束符（EOF）
          else if (nread == 0) 
              break; // 跳出循环
          nleft -= nread; // 更新剩余需要读取的字节数
          bufp += nread; // 移动bufp指针到已读数据之后
      }
      return (n - nleft); // 返回已经成功读取的字节数
  }
  ~~~

  - 是`rio_readn`带缓冲区的版本；

- rio_read

  ~~~cpp
  static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n) {
      int cnt;
      // 当缓冲区中未读字节数小于等于0时，即缓冲区为空或已读完
      while (rp->rio_cnt <= 0) { 
          // 调用read函数从文件（通过文件描述符rp->rio_fd）读取数据到缓冲区rp->rio_buf
          rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf)); 
          if (rp->rio_cnt < 0) { 
              // 如果读取错误，且不是因为信号处理程序中断（EINTR）导致的错误
              if (errno != EINTR) 
                  return -1; // 直接返回-1表示读取失败
          } 
          // 如果读取到文件结束符（EOF），即读取字节数为0
          else if (rp->rio_cnt == 0) 
              return 0; // 返回0表示到达文件末尾
          else 
              // 重置缓冲区指针，使其指向缓冲区起始位置，准备后续读取
              rp->rio_bufptr = rp->rio_buf; 
      }
  
      // 确定要从缓冲区复制到用户缓冲区的字节数，取n和缓冲区未读字节数rp->rio_cnt中的较小值
      cnt = n; 
      if (rp->rio_cnt < n) 
          cnt = rp->rio_cnt; 
      // 将数据从读缓冲区（从rp->rio_bufptr位置开始）复制到用户缓冲区usrbuf
      memcpy(usrbuf, rp->rio_bufptr, cnt); 
      // 移动缓冲区指针，跳过已复制的字节数
      rp->rio_bufptr += cnt; 
      // 更新缓冲区中剩余未读字节数
      rp->rio_cnt -= cnt; 
      return cnt; // 返回实际复制到用户缓冲区的字节数
  }
  ~~~

##### 示例

- 一次一行地从标准输入复制一个文本文件到标准输出

  ~~~cpp
  #include "csapp.h" // 本书配套头文件
  
  int main(int argc, char **argv) {
      int n;
      rio_t rio; // 读缓冲区
      char buf[MAXLINE]; // 从标准输入到标准输出的temp
  
      Rio_readinitb(&rio, STDIN_FILENO); // 从标准输入初始化缓冲区
      while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
        	// 从读缓冲区中读取一行到内存位置buf(最多MAXLINE-1)
      {
          Rio_writen(STDOUT_FILENO, buf, n); // 将文本行写入到标准输出
      }
      return 0;
  }
  ~~~







### 附2 标准I/O库概述

- C语言定义的标准I/O库(libc)，提供了一组高级输入输出函数，是在Unix I/O的基础上进行了更高级、更便捷的封装；

  - 文件操作类：`fopen`用于打开文件，`fclose`用于关闭文件 。比如`fopen("test.txt", "r");` 可打开名为`test.txt`的文件用于读取；
  - 字节读写类：`fread`用于从文件读取字节数据，`fwrite`用于向文件写入字节数据 。例如从文件读取数据到数组`char buffer[100]; fread(buffer, 1, 100, file);` （`file`为已打开的文件指针 ）；
  - 字符串读写类：`fgets`用于从文件读取字符串，`fputs`用于向文件写入字符串 。像`fgets(buffer, 100, file);` 从文件读字符串到`buffer` ，最多读 99 个字符（留 1 个给结束符 ）；
  - 格式化 I/O 类：`scanf`用于按指定格式从标准输入读取数据，`printf`用于按指定格式向标准输出打印数据 。如`printf("Hello, %d", num);` 可按格式输出；

- “流”的概念：

  标准 I/O 库把打开的文件抽象为 “流”。对于程序员来说，流是指向`FILE`类型结构的指针 。每个符合 ANSI C 标准的程序开始运行时，都默认有三个预打开的流：

  - **`stdin`**：对应标准输入，文件描述符为 0 ，用于从外部获取输入数据，比如从键盘读入用户输入；
  - **`stdout`**：对应标准输出，文件描述符为 1 ，用于向外部输出程序处理结果，比如在屏幕上显示文本；
  - **`stderr`**：对应标准错误，文件描述符为 2 ，用于输出程序运行中的错误信息；

- 流缓冲区作用：

  `FILE`类型的流是对文件描述符和流缓冲区的抽象。流缓冲区和 RIO 读缓冲区目的类似，都是为减少开销较高的 Linux I/O 系统调用次数 。以`getc`函数为例：

  - 首次调用`getc`时，标准 I/O 库会调用一次`read`系统调用，从文件读取一定量数据填充流缓冲区；
  - 然后从缓冲区取出第一个字节返回给应用程序。只要缓冲区还有未读字节，后续对`getc`的调用就直接从缓冲区获取数据，无需再次触发`read`系统调用，从而提高了 I/O 操作效率 。 总之，标准 I/O 库通过流和缓冲区机制，为程序员提供了更便捷、高效的文件读写等 I/O 操作方式，简化了 Unix I/O 操作的复杂性；