# databases_design
  网络聊天室
用户区：
	功能1：用户注册登录
	功能2：保存聊天记录
	功能3：账号注销
	功能4：修改密码
	*功能5：绑定手机号

​	功能6：设置头像

管理员区：
	功能1：查询所有账号信息
	功能2：删除某个账号



环境：操作系统：unbutu 18.0.4, 

​			mysql  Ver :14.14 Distrib 5.7.39, for Linux (x86_64)



数据库表

```
// 建立yourdb库
create database yourdb;

// 创建user表
USE yourdb;
CREATE TABLE user(
    username char(50) NULL,
    passwd char(50) NULL
)ENGINE=InnoDB;
```

注册功能实现：

​	总体思想：服务器端解析浏览器的post请求，并获取表单数据，解析出用户名和密码，并添加到数据库

第一步，先将数据库中的用户名和密码载入到服务器的map中来，map中的key为用户名，value为密码。

```c++
void http_conn::initmysql_result(connection_pool *connPool)
{
    //先从连接池中取一个连接
    MYSQL *mysql = NULL;
    connectionRAII mysqlcon(&mysql, connPool);

    //在user表中检索username，passwd数据，浏览器端输入
    if (mysql_query(mysql, "SELECT username,passwd FROM user"))
    {
        LOG_ERROR("SELECT error:%s\n", mysql_error(mysql));
    }

    //从表中检索完整的结果集
    MYSQL_RES *result = mysql_store_result(mysql);

    //返回结果集中的列数
    int num_fields = mysql_num_fields(result);

    //返回所有字段结构的数组
    MYSQL_FIELD *fields = mysql_fetch_fields(result);

    //从结果集中获取下一行，将对应的用户名和密码，存入map中
    while (MYSQL_ROW row = mysql_fetch_row(result))
    {
        string temp1(row[0]);
        string temp2(row[1]);
        users[temp1] = temp2;
    }
}
```

第二步：**提取用户名和密码**

```c
strcpy(m_real_file, doc_root);
    int len = strlen(doc_root);
    // printf("m_url:%s\n", m_url);
    const char *p = strrchr(m_url, '/');

    //处理cgi
    if (cgi == 1 && (*(p + 1) == '2' || *(p + 1) == '3' || *(p + 1) == '4' || *(p + 1) == '5'))
    {

        //根据标志判断是登录检测还是注册检测
        char flag = m_url[1];

        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/");
        strcat(m_url_real, m_url + 2);
        strncpy(m_real_file + len, m_url_real, FILENAME_LEN - len - 1);
        free(m_url_real);

        //将用户名和密码提取出来
        //user=123&passwd=123
        char name[100], password[100],newpasswd[100];
        int i;
        for (i = 5; m_string[i] != '&'; ++i)
            name[i - 5] = m_string[i];
        name[i - 5] = '\0';

        int j = 0;
        for (i = i + 10; m_string[i] != '\0'; ++i, ++j)
            password[j] = m_string[i];
        password[j] = '\0';
```

通过m_url定位/所在位置，根据/后的第一个字符判断是登录还是注册校验。

```
2: 登录校验
3：注册校验
4：注销校验
5：修改校验
```

```c
if (*(p + 1) == '3')
        {
            //如果是注册，先检测数据库中是否有重名的
            //没有重名的，进行增加数据
            char *sql_insert = (char *)malloc(sizeof(char) * 200);
            strcpy(sql_insert, "INSERT INTO user(username, passwd) VALUES(");
            strcat(sql_insert, "'");
            strcat(sql_insert, name);
            strcat(sql_insert, "', '");
            strcat(sql_insert, password);
            strcat(sql_insert, "')");

            if (users.find(name) == users.end())
            {

                m_lock.lock();
                int res = mysql_query(mysql, sql_insert);
                users.insert(pair<string, string>(name, password));
                m_lock.unlock();

                if (!res)
                    strcpy(m_url, "/log.html");
                else
                    strcpy(m_url, "/registerError.html");
            }
            else
                strcpy(m_url, "/registerError.html");
        }
        //如果是登录，直接判断
        //若浏览器端输入的用户名和密码在表中可以查找到，返回1，否则返回0
        else if (*(p + 1) == '2')
        {
            if (users.find(name) != users.end() && users[name] == password)
                strcpy(m_url, "/welcome.html");
            else
                strcpy(m_url, "/logError.html");
        }
		//如果是注销，判断正确与否然后删除
        else if (*(p + 1) == '4')
        {
            // (username, passwd) VALUES(
            char *sql_delete = (char *)malloc(sizeof(char) * 200);
            strcpy(sql_delete, "DELETE FROM user WHERE username=");
            strcat(sql_delete, "'");
            strcat(sql_delete, name);
            strcat(sql_delete, "'&& passwd='");
            strcat(sql_delete, password);
            strcat(sql_delete, "')");
            if (users.find(name) == users.end())
            {

                strcpy(m_url, "/nonuser.html");
            }
            else
            {
                m_lock.lock();
                int res = mysql_query(mysql, sql_delete);
                users.erase(name);
                m_lock.unlock();
                if (res)
                {
                    strcpy(m_url, "/logoff_success.html");
                }
                else
                {
                    strcpy(m_url, "/logoff_fail.html");
                }
            }
        }
        else if (*(p + 1) == '5')
        {
            int i;
            for (i = 5; m_string[i] != '&'; ++i)
                name[i - 5] = m_string[i];
            name[i - 5] = '\0';

            int j = 0;
            for (i = i + 10; m_string[i] != '&'; ++i, ++j)
                password[j] = m_string[i];
            password[j] = '\0';

            int k = 0;
            for (i = i + 10; m_string[i] != '\0'; ++i, ++k)
                newpasswd[k] = m_string[i];
            newpasswd[k] = '\0';

            // cout<<m_string<<name<<endl<<password<<endl<<newpasswd<<endl;
            // (username, passwd) VALUES(
            char *sql_update = (char *)malloc(sizeof(char) * 200);
            strcpy(sql_update, "UPDATE user SET passwd=");
            strcat(sql_update, "'");
            strcat(sql_update, newpasswd);
            strcat(sql_update, "'WHERE username='");
            strcat(sql_update, name);
            strcat(sql_update, "'&& passwd='");
            strcat(sql_update, password);
            strcat(sql_update, "'");
            if (users.find(name) == users.end())
            {

                strcpy(m_url, "/nonuser_change.html");
            }
            else
            {
                m_lock.lock();
                int res = mysql_query(mysql, sql_update);
                users.erase(name);
                users.insert(pair<string, string>(name, newpasswd));
                m_lock.unlock();
                if (!res)
                {
                    strcpy(m_url, "/change_success.html");
                }
                else
                {
                    strcpy(m_url, "/change_fail.html");
                }
            }
        }
```

### 页面跳转

通过m_url定位/所在位置，根据/后的第一个字符，使用分支语句实现页面跳转。具体的，

```c
 if (*(p + 1) == '0')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/register.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
        free(m_url_real);
    }
    else if (*(p + 1) == '1')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/log.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == 'p')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/picture.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '6')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/video.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '7')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/fans.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '7')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/fans.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '8')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/logoff.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '9')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/change.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else
        strncpy(m_real_file + len, m_url, FILENAME_LEN - len - 1);

    if (stat(m_real_file, &m_file_stat) < 0)
        return NO_RESOURCE;
    if (!(m_file_stat.st_mode & S_IROTH))
        return FORBIDDEN_REQUEST;
    if (S_ISDIR(m_file_stat.st_mode))
        return BAD_REQUEST;
    int fd = open(m_real_file, O_RDONLY);
    m_file_address = (char *)mmap(0, m_file_stat.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    close(fd);
    return FILE_REQUEST;
```

**单例模式创建**，结合代码描述连接池的单例实现。

**连接池代码实现**，结合代码对连接池的外部访问接口进行详解。

**RAII机制释放数据库连接**，描述连接释放的封装逻辑。

### 单例模式创建

使用局部静态变量懒汉模式创建连接池。

```
 1class connection_pool
 2{
 3public:
 4    //局部静态变量单例模式
 5    static connection_pool *GetInstance();
 6
 7private:
 8    connection_pool();
 9    ~connection_pool();
10}
11
12connection_pool *connection_pool::GetInstance()
13{
14    static connection_pool connPool;
15    return &connPool;
16}
```

### 连接池代码实现

连接池的定义中注释比较详细，这里仅对其实现进行解析。

连接池的功能主要有：初始化，获取连接、释放连接，销毁连接池。

#### **初始化**

值得注意的是，销毁连接池没有直接被外部调用，而是通过RAII机制来完成自动释放；使用信号量实现多线程争夺连接的同步机制，这里将信号量初始化为数据库的连接总数。

```c
 1connection_pool::connection_pool()
 2{
 3    this->CurConn = 0;
 4    this->FreeConn = 0;
 5}
 6
 7//RAII机制销毁连接池
 8connection_pool::~connection_pool()
 9{
10    DestroyPool();
11}
12
13//构造初始化
14void connection_pool::init(string url, string User, string PassWord, string DBName, int Port, unsigned int MaxConn)
15{
16    //初始化数据库信息
17    this->url = url;
18    this->Port = Port;
19    this->User = User;
20    this->PassWord = PassWord;
21    this->DatabaseName = DBName;
22
23    //创建MaxConn条数据库连接
24    for (int i = 0; i < MaxConn; i++)
25    {
26        MYSQL *con = NULL;
27        con = mysql_init(con);
28
29        if (con == NULL)
30        {
31            cout << "Error:" << mysql_error(con);
32            exit(1);
33        }
34        con = mysql_real_connect(con, url.c_str(), User.c_str(), PassWord.c_str(), DBName.c_str(), Port, NULL, 0);
35
36        if (con == NULL)
37        {
38            cout << "Error: " << mysql_error(con);
39            exit(1);
40        }
41
42        //更新连接池和空闲连接数量
43        connList.push_back(con);
44        ++FreeConn;
45    }
46
47    //将信号量初始化为最大连接次数
48    reserve = sem(FreeConn);
49
50    this->MaxConn = FreeConn;
51}
```

#### **获取、释放连接**

当线程数量大于数据库连接数量时，使用信号量进行同步，每次取出连接，信号量原子减1，释放连接原子加1，若连接池内没有连接了，则阻塞等待。

另外，由于多线程操作连接池，会造成竞争，这里使用互斥锁完成同步，具体的同步机制均使用lock.h中封装好的类。

```c
 1//当有请求时，从数据库连接池中返回一个可用连接，更新使用和空闲连接数
 2MYSQL *connection_pool::GetConnection()
 3{
 4    MYSQL *con = NULL;
 5
 6    if (0 == connList.size())
 7        return NULL;
 8
 9    //取出连接，信号量原子减1，为0则等待
10    reserve.wait();
11
12    lock.lock();
13
14    con = connList.front();
15    connList.pop_front();
16
17    //这里的两个变量，并没有用到，非常鸡肋...
18    --FreeConn;
19    ++CurConn;
20
21    lock.unlock();
22    return con;
23}
24
25//释放当前使用的连接
26bool connection_pool::ReleaseConnection(MYSQL *con)
27{
28    if (NULL == con)
29        return false;
30
31    lock.lock();
32
33    connList.push_back(con);
34    ++FreeConn;
35    --CurConn;
36
37    lock.unlock();
38
39    //释放连接原子加1
40    reserve.post();
41    return true;
42}
```

#### **销毁连接池**

通过迭代器遍历连接池链表，关闭对应数据库连接，清空链表并重置空闲连接和现有连接数量。

```
 1//销毁数据库连接池
 2void connection_pool::DestroyPool()
 3{
 4    lock.lock();
 5    if (connList.size() > 0)
 6    {
 7        //通过迭代器遍历，关闭数据库连接
 8        list<MYSQL *>::iterator it;
 9        for (it = connList.begin(); it != connList.end(); ++it)
10        {
11            MYSQL *con = *it;
12            mysql_close(con);
13        }
14        CurConn = 0;
15        FreeConn = 0;
16
17        //清空list
18        connList.clear();
19
20        lock.unlock();
21    }
22
23    lock.unlock();
24}
```

### RAII机制释放数据库连接

将数据库连接的获取与释放通过RAII机制封装，避免手动释放。

#### **定义**

这里需要注意的是，在获取连接时，通过有参构造对传入的参数进行修改。其中数据库连接本身是指针类型，所以参数需要通过双指针才能对其进行修改。

```c
 1class connectionRAII{
 2
 3public:
 4    //双指针对MYSQL *con修改
 5    connectionRAII(MYSQL **con, connection_pool *connPool);
 6    ~connectionRAII();
 7
 8private:
 9    MYSQL *conRAII;
10    connection_pool *poolRAII;
11};
```

#### **实现**

不直接调用获取和释放连接的接口，将其封装起来，通过RAII机制进行获取和释放。

```c
 1connectionRAII::connectionRAII(MYSQL **SQL, connection_pool *connPool){
 2    *SQL = connPool->GetConnection();
 3
 4    conRAII = *SQL;
 5    poolRAII = connPool;
 6}
 7
 8connectionRAII::~connectionRAII(){
 9    poolRAII->ReleaseConnection(conRAII);
10}
```

前端代码实现（页面效果展示)

![image-20220813172118668](C:\Users\19252\AppData\Roaming\Typora\typora-user-images\image-20220813172118668.png)



前端代码实现

```html
<!DOCTYPE html>
<html lang="en">

    <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <!-- 引入css文件 -->
        <link rel="stylesheet" href="./login.css">
        <!-- 引入jquery -->
        <script src="http://code.jquery.com/jquery-latest.js"></script>
        <title>登录</title>
    </head>
    <body>
        <!-- 最外层的大盒子 -->
       <div class="box">
        <!-- 滑动盒子 -->
        <div class="pre-box">
            <h1>WELCOME</h1>
            <p>JOIN US!</p>
            <div class="img-box">
                <img src="./img/waoku.jpg" alt="">
            </div>
        </div>
        <!-- 注册盒子 -->
        <div class="register-form">
            <!-- 标题盒子 -->
            <div class="title-box">
                <h1>注册</h1>
            </div>
            <!-- 输入框盒子 -->
            <form action="3CGISQL.cgi" method="post">
            <div class="input-box">
                <input type="text" name="user" placeholder="用户名" required="required">
                <input type="password" name="password" placeholder="登录密码" required="required">
            </div>
            <!-- 按钮盒子 -->
            <div class="btn-box">
                <button type="submit">注册</button>
            </form>
                <!-- 绑定点击事件 -->
                <p onclick="mySwitch()">已有账号?去登录</p>
            </div>
        </div>
        <!-- 登录盒子 -->
        <div class="login-form">
            <!-- 标题盒子 -->
            <div class="title-box">
                <h1>登录</h1>
            </div>
            <!-- 前後端表单数据 -->
            <!-- 输入框盒子 -->
            <form action="2CGISQL.cgi" method="post">
            <div class="input-box">
                <input type="text" name="user" placeholder="用户名" required="required">
                <input type="password" name="password" placeholder="登录密码" required="required">
            </div>
            <!-- 按钮盒子 -->
            <div class="btn-box">
                <button type="submit">登录</button>
            </form>
                <!-- 绑定点击事件 -->
                <p onclick="mySwitch()">没有账号?去注册</p>
            </div>
        </div>
       </div>
       <script>
            // 滑动的状态
             let flag=true
             const mySwitch=()=>{
                if(flag){
                    // 获取到滑动盒子的dom元素并修改它移动的位置
                    $(".pre-box").css("transform","translateX(100%)")
                    // 获取到滑动盒子的dom元素并修改它的背景颜色
                    $(".pre-box").css("background-color","#c9e0ed")
                    //修改图片的路径
                    $("img").attr("src","./img/wuwu.jpeg")
                    
                }
                else {
                    $(".pre-box").css("transform","translateX(0%)")
                    $(".pre-box").css("background-color","#edd4dc")
                    $("img").attr("src","./img/waoku.jpg")
                }
                flag=!flag
             }
       </script>
       <script>
            const bubleCreate=()=>{
                // 获取body元素
                const body = document.body
                // 创建泡泡元素
                const buble = document.createElement('span')
                // 设置泡泡半径
                let r = Math.random()*5 + 25 //半径大小为25~30
                // 设置泡泡的宽高
                buble.style.width=r+'px'
                buble.style.height=r+'px'
                // 设置泡泡的随机起点
                buble.style.left=Math.random()*innerWidth+'px'
                // 为body添加buble元素
                body.append(buble)
                // 4s清除一次泡泡
                setTimeout(()=>{
                    buble.remove()
                },4000)
            }
            // 每200ms生成一个泡泡
            setInterval(() => {
                bubleCreate()
            }, 200);
        </script>
    </body>
</html>
```

