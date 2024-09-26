# Qt-NetClient
## 客户端结构

1、Window系统（使用Qt跨平台框架，也可简单部署到其它系统）

2、Openssl安全加密传输

3、Asio库作为通信基础

4、短任务线程池。长任务多线程传输，避免UI界面冻结。

5、实现文件系统视图，可以进行层级化查看文件

6、界面与逻辑分离

## (一)SR_Tool类

​	该类是结合Asio库与Qt信号槽进行封装的用于方便与服务器进行报文交互的类。API包括：同步操作API、异步操作API. 已经信息槽，在触发事件后，将会发送相关信号，调用的类可以连接信息，触发相关处理函数。注意，该类只封装了方便ssl的操作，如果需要进行底层套接字操作，需要修改，或者使用gelssl.获得对象，再得到底层套接字，进行操作。

```
使用实例
string ip="127.0.0.1";
int port=8080;
SR_Tool tool(nullptr,ip,port);

asio::error_code ec;
tool.connect(ec);	//同步连接
tool.send(buf,ec);	//同步发送

//异步API
tool.async_connect();	//异步连接
tool.async_send(buf,len);	//异步发送
tool.SR_run();	//开始异步任务
tool.SR_stop();	//结束异步任务
```



```
typedef std::shared_ptr < ip::tcp::socket>socket_ptr;						 //定义为普通套接字共享指针共享指针
typedef std::shared_ptr < ssl::stream<ip::tcp::socket>>sslsock_ptr;			   //安全套接字共享指针

class SR_Tool  : public QObject,public std::enable_shared_from_this<SR_Tool>
{
	Q_OBJECT

public:
	SR_Tool(QObject* parent,const std::string &enpaddr,const int &enpPort);		//传递对端地址和端口进行构造
	~SR_Tool();

//同步处理接口，传递一个asio::error_code,同步操作后将结果存放在ec
public:
	void connect(asio::error_code& ec);			//同步连接
	void send(char* buf, size_t len,asio::error_code& ec);			//同步发送
	void recv(char* buf, size_t len, asio::error_code& ec);			//同步接收
	ssl::stream<ip::tcp::socket>* getSsl();							//返回ssl指针
	ip::tcp::endpoint GetEndPoint();								//返回对端对象
	bool sendPdu(const PDU& pud);									//将一个pdu发送到服务器
	bool recvRespon(ResponsePack& res);								//同步接受一个回复体
	bool recvFileInfo(FILEINFO& fileinfo);							//接受一条文件信息

//异步处理接口，异步事件完成会触发相关信号。连接信号，接收异步结果。也可以传递回调函数，在异步事件就绪时候，调用回调函数。调用异步任务后，需要SR_run()启动异步循环
public:
	void async_connect();							//异步连接服务器
	void async_send(char* buf, size_t len);			//异步发送数据
	void async_recv(char* buf, size_t len);			//异步接收len字节数据到buf

	void async_connect(std::function<void()>fun);	//传递函数对象，异步连接
	void async_send(char* buf, size_t len, std::function<void()>fun);	//传递函数对象，异步发送
	void async_recv(char* buf, size_t len, std::function<void()>fun);	//传递函数对象，异步接收

	void SR_run();									//开始异步任务
	void SR_stop();									//结束异步任务
	
//异步信号
signals:	
	void connected();	//连接成功
	void sendOK();		//发生成功
	void recvOK();		//接收成功
	void error(QString message);	//错误信号
private:
	//异步回调函数
	void connect_handler(const error_code& ec);		//异步连接成功后回调函数
	void send_handler(const error_code& ec, std::size_t bytes_transferred);		//异步发送成功回调函数
	void recv_handler(const error_code& ec, std::size_t bytes_transferred);		//异步接收成功回调函数
private:
	std::shared_ptr <io_context> m_Context;					//IO上下文
	std::shared_ptr<ssl::context>	m_sslContext;			//ssl安全连接上下文
	sslsock_ptr m_sslsock;									//sock安全套接字
	ip::tcp::endpoint m_ep;									//保存对端地址和端口信息，即服务器地址和端口
	std::atomic<bool>m_running=true;						//用于控制异步任务停止
};
```

##   

##  (二)UdTool类

​	该类是负责上传和下载的类，也就是长任务类。使用方法，只需要传递服务器的IP和长任务端口，然后调用公有接口设置通信协议即可。该类需要配合多线程使用，因为下载上传属于长任务，不能与UI同线程，否则会导致UI冻结，影响用户体验。详细的多线程调用，请查看主程序DiskClient类源文件的上传和下载函数。

```
UdTool tool(ip,port);
tool.SetTranPdu(tranpdu);
tool.doWorkUP();	//执行上传
```



```
//负责上传和下载文件的类
class UdTool  : public QObject
{
	Q_OBJECT
public:
	UdTool(const QString& IP, const int& Port,QObject *parent=nullptr);
	~UdTool();
public:
	void SetTranPdu(const TranPdu& tranpdu);											//设置要上传和下载的文件名
	void SetControlAtomic(std::shared_ptr <std::atomic<int>>control,
		std::shared_ptr<std::condition_variable>cv, std::shared_ptr<std::mutex>mutex);	//设置传输控制变量
public slots:
	void doWorkUP();					//执行上传操作
	void doWorkDown();					//执行下载操作
signals:
	void workFinished();												//任务完成信号
	void sendItemData(TranPdu data,std::uint64_t fileid);				//发送添加文件视图信号
	void erroroccur(QString message);									//错误信号
	void sendProgress(qint64 bytes, qint64 total);						//发送进度信号
private:
	bool calculateSHA256();				//计算文件哈希值
	bool sendTranPdu();					//发生TranPdu
	bool recvRespon();					//接收服务器回复报文
	bool sendFile();					//发送文件
	bool recvFile();					//接受文件
	bool verifySHA256();				//哈希检查
	void removeFileIfExists();			//移除文件
private:
	SR_Tool m_srTool;		//发送接收工具
	TranPdu m_TranPdu;		//通信结构体
	QString m_filename;		//带路径的文件全名
	QFile   m_file;			//上传或者下载的文件保存数据对象

	ResponsePack m_res;		//服务端回复
	asio::error_code m_ec;	//错误码
	char buf[1024] = { 0 };	//收发数据缓冲区
	
	//上传和下载控制
	std::shared_ptr <std::atomic<int>>m_control;		//原子类型，工作控制.0为继续，1为暂停，2为结束
	std::shared_ptr<std::condition_variable> m_cv;		//共享条件变量
	std::shared_ptr<std::mutex>m_mutex;					//共享互斥锁
};
```

## 	(三)ShortTaskManage 短任务管理器

​	该类的主要功能是负责处理短任务，如创建文件夹、删除文件夹等。其使用方法需要配合任务队列来处理，再DiskClient类使用std::bind将其任务添加到任务队列去执行。

```
class ShortTaskManage  : public QObject
{
	Q_OBJECT

public:
	ShortTaskManage(std::shared_ptr<SR_Tool>srTool,QObject* parent=nullptr);
	~ShortTaskManage();			
public:
	void GetFileList();					//获取文件列表
	void MakeDir(quint64 parentid, QString newdirname);	//创建文件夹
	void del_File(quint64 fileid, QString filename);	//删除文件
signals:
	void GetFileListOK(std::shared_ptr<std::vector<FILEINFO>>vet);
	void error(QString message);
	void sig_createDirOK(quint64 newid,quint64 parentid,QString newdirname);
	void sig_delOK();	
private:
	std::shared_ptr<SR_Tool>m_srTool;
};
```

## (四)短任务队列

​	该任务主要处理短任务，后台启动一个或者多个线程，逐步从队列中取出任务进行处理，如果只有一个后台线程不需要注意线程并发问题。但是如果线程两条以上，就需要注意网络传输的收发问题，避免粘包问题。使用方法

```
m_task = std::make_shared<TaskQue>(1); 
m_taskManage = std::make_shared<ShortTaskManage>(m_srTool, this);//短任务管理器，用于处理短任务
m_task->AddTask(std::bind(&ShortTaskManage::MakeDir,m_taskManage.get(), parentID, newdirname));	//创建文件夹
```



```
class TaskQue {
public:
	explicit TaskQue(size_t threadCount = 1);
	TaskQue() = default;
	TaskQue(TaskQue&&) = default;
	~TaskQue();

	void Close();	//关闭队列
	template<class F>
	void AddTask(F&& task);
private:
	std::mutex m_mutex;
	std::condition_variable m_cv;
	std::queue<std::function<void()>>m_deq;		//任务队列，存储函数对象
	bool m_isClose = false;						//任务队列是否关闭

	std::vector<std::thread>m_workers;		//保存工作线程的容器
	void Worker();	//工作线程处理函数
};

//往任务池添加任务
template<class F>
inline void TaskQue::AddTask(F&& task)
{
	if (m_isClose == true)return;
	{
		std::lock_guard<std::mutex>locker(m_mutex);
		m_deq.emplace(std::forward<F>(task));
	}
	m_cv.notify_one();
}
```

## (五)窗体类

### 1、用户信息

（用来在主程序显示用户信息）

```
class UserInfoWidget : public QDialog
{
	Q_OBJECT

public:
	UserInfoWidget(UserInfo* info,QWidget *parent = nullptr);
	~UserInfoWidget();

private:
	Ui::UserInfoWidgetClass *ui;
};

```

### 2、下载上传进度条

​	·可以与UdTool配合实习下载控制，控制暂停，继续结束。

```
UdTool* worker = new UdTool(CurServerIP,CurLPort,nullptr);
worker->SetTranPdu(pdu);

TProgress* progress = new TProgress(this);
progress->setFilename(1, filename);
ui->TranList->addWidget(progress);
worker->SetControlAtomic(progress->getAtomic(),progress->getCv(),progress->getMutex());
```



```
class TProgress : public QDialog
{
	Q_OBJECT

public:
	TProgress(QWidget *parent = nullptr);
	~TProgress();
	void setFilename(const int& option, const QString& filename);		
	std::shared_ptr <std::atomic<int>> getAtomic();
	std::shared_ptr<std::condition_variable>getCv();
	std::shared_ptr<std::mutex>getMutex();
public slots:
	void do_updateProgress(qint64 bytes, qint64 total);		//更新下载和上传进度
	void do_finished();				//进度达到100%槽函数
private slots:
	void on_btnstop_clicked();		//暂停键按下
	void on_btncontine_clicked();	//继续键按下
	void on_btnclose_clicked();		//结束键按下
private:
	Ui::TProgressClass ui;
	QString m_filename;			//文件名
	int m_option;				//区分上传还是下载，1上传，2下载

	//下载控制
	std::shared_ptr <std::atomic<int>>m_control;							//原子类型控制量
	std::shared_ptr<std::condition_variable> m_cv;		//共享条件变量
	std::shared_ptr<std::mutex>m_mutex;					//共享互斥锁

};
```

### 3、登陆与注册界面

​	在主程序类创建，然后在调用，详细需要看DiskClient的bool startClient(); 函数，里面实现了异步登陆。

```
class Login : public QDialog
{
	Q_OBJECT

public:
	Login(std::shared_ptr<SR_Tool>srTool,UserInfo *info,QWidget* parent = nullptr);
	~Login();
private:
	void Tologin();		//执行登陆操作
	void sign();		//执行注册操作
	void connection();	//执行连接操作
private slots:
	void on_btnLogin_clicked();		//登陆按钮槽函数
	void on_btnSign_clicked();		//注册按钮槽函数
	void do_Connected();	//连接成功会执行
	void do_Send();			//发送成功后执行
	void do_recved();		//接收成功后执行
	void do_error(QString message);		//出错就执行
	void do_ArrToInfo();				//将接收到的字节数据转换到用户信息结构体
private:
	Ui::LoginClass *ui;
	std::shared_ptr<SR_Tool>m_srTool;		//用来进行通信
	bool m_loginOrSign=true;				//区别是注册操作还是登陆操作
	bool m_connected = false;				//是否已经连接到服务器

	char m_buf[1024] = { 0 };				//保存接收到数据的缓冲区
	char m_recvbuf[1024] = { 0 };		
	char m_infobuf[1024] = { 0 };	//保存接收到的用户信息缓冲区

	QString m_user;					//用户名
	QString m_pwd;					//密码
	UserInfo* m_info = nullptr;		//保存服务器发送回来的用户信息
};
```

### 4、文件视图系统

​	使用QWidgetlist配合自定义数据结构实现的文件系统视图。 核心在弄懂ItemDate结构。 该结构是指一个项，项又分为两类：文件项（表示一个文件，如MP3文件，子指针为空）、文件夹项（用来表示一个文件夹，子指针不为空，保存子文件夹和子文件项）

```
struct ItemDate {
    quint64 ID = 0;			//文件id,对应服务器数据库的文件id
    QString filename;		//文件名
    QString filetype;		//文件类型，如d文件夹，mp3等
    quint64 Grade = 0;      //文件距离根文件夹的距离，用来排序。构建文件系统
    quint64 parentID = 0;	//父项ID
    quint64 filesize = 0;	//文件大小
    QString filedate;	    //文件日期

    QListWidget* parent = nullptr;      //父文件夹（前驱文件夹）指针
    QListWidget* child = nullptr;       //本文件夹指针，即保存该文件夹中的项的指针
    ItemDate() = default;
};

class FileSystemView : public QWidget
{
	Q_OBJECT

public:
	FileSystemView(QWidget *parent = nullptr);
	~FileSystemView();

    void InitView(std::vector<FILEINFO > &vet);
	quint64 GetCurDirID();												//返回当前目录ID
	void addNewFileItem(TranPdu newItem,std::uint64_t fileid);		    //往文件视图中添加一个新文件项
	void addNewDirItem(quint64 newid, quint64 parentid, QString newdirname);	//往文件视图添加一个新文件夹项
private slots:
	void do_itemDoubleClicked(QListWidgetItem* item);					//项目被双击
	void do_itemClicked(QListWidgetItem* item);							//项目被单击
	void do_createMenu(const QPoint& pos);								//创建鼠标右键菜单
	void do_displayItemData(QListWidgetItem* item);						//鼠标滑进后显示项目信息

	void on_actback_triggered();										//返回行为被按下
	void on_actRefresh_triggered();										//刷新按键按下
	void on_actdown_triggered();										//下载按钮按下
	void on_actupfile_triggered();										//上传按钮按下
	void on_actcreateDir_triggered();									//创建文件夹按钮
	void on_actdelete_triggered();										//删除按钮按下
signals:
	void sig_RefreshView();												//刷新视图信号
	void sig_down(ItemDate data);										//下载信号
	void sig_up();														//上传信号
	void sig_createdir(quint64 parentID,QString newdirname);			//创建文件夹信号
	void sig_deleteFile(quint64 fileif, QString filename);				//删除文件夹信号
private:
	void initIcon(QListWidgetItem* item, QString filetype);
	void initChildListWidget(QListWidget* List);
	quint64 initDirByteSize(QListWidgetItem* item);						//递归更新文件夹空间信息
private:
	Ui::FileSystemViewClass ui;
    QListWidgetItem* currentItem = nullptr; //当前项
	QListWidgetItem* curDirItem = nullptr;	//当前文件夹项

	QListWidget* rootWidegt = nullptr;		//根文件夹界面指针
	QMap<uint64_t, QListWidgetItem*>Map;	//文件夹项ID对应文件夹项指针映射
};

```

## (六)、主程序类

​	该类是整个程序的中心类，所有操作都需要通过它为中转或者它调用执行。main函数也是创建该类执行。

```
int main()
{
	QApplication a(argc, argv);
    DiskClient client;
    if (client.startClient()==true)
    {   
        client.show();
        return a.exec();
    }
    else
        return 0;
    return a.exec();
}
```



```
class DiskClient : public QMainWindow
{
    Q_OBJECT

public:
    DiskClient(QWidget* parent = nullptr);
    ~DiskClient();
    bool startClient(); 
    
private:
    void initClient();
    void initUI();
    void RequestCD();  
    bool connectMainServer();
private slots:
    void widgetChaned();
    void on_BtnInfo_clicked();

    void do_createDir(quint64 parentID, QString newdirname);        
    void do_getFileinfo(std::shared_ptr<std::vector<FILEINFO>>vet);
    void do_delFile(quint64 fileif, QString filename);
    void do_btnPuts_clicked();
    void do_Gets_clicked(ItemDate item); 
    void do_error(QString message);                                    
private:
    Ui::DiskClientClass* ui;
    std::shared_ptr<UserInfo>m_userinfo;            //用户信息结构体
    std::shared_ptr<SR_Tool>m_srTool;               //发送和接受数据工具
    std::shared_ptr<TaskQue>m_task;                 //任务队列
    std::shared_ptr<ShortTaskManage>m_taskManage;   //短任务管理器

    FileSystemView* m_FileView = nullptr;             //文件系统视图

    QString CurServerIP;          //连接主服务器后，当前分配的服务器
    int CurSPort=0;               //当前分配的端口
    int CurLPort=0;               //当前分配的长任务端口
};


