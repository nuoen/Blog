netd 解析 
system/netd/server/main.cpp

初始化NetlinkManager
NetlinkManager *nm = NetlinkManager::Instance();
    NetlinkManager *NetlinkManager::Instance() {
    if (!sInstance)
        sInstance = new NetlinkManager();
    return sInstance;
}

    NetlinkManager::NetlinkManager() {
        mBroadcaster = nullptr;
    }
开始
if (nm->start()) {
    ALOGE("Unable to start NetlinkManager (%s)", strerror(errno));
    exit(1);
}
int NetlinkManager::start() {
    if ((mUeventHandler = setupSocket(&mUeventSock, NETLINK_KOBJECT_UEVENT,
         0xffffffff, NetlinkListener::NETLINK_FORMAT_ASCII, false)) == nullptr) {
        return -1;
    }

    if ((mRouteHandler = setupSocket(&mRouteSock, NETLINK_ROUTE,
                                     RTMGRP_LINK |
                                     RTMGRP_IPV4_IFADDR |
                                     RTMGRP_IPV6_IFADDR |
                                     RTMGRP_IPV6_ROUTE |
                                     (1 << (RTNLGRP_ND_USEROPT - 1)),
         NetlinkListener::NETLINK_FORMAT_BINARY, false)) == nullptr) {
        return -1;
    }

    if ((mQuotaHandler = setupSocket(&mQuotaSock, NETLINK_NFLOG,
            NFLOG_QUOTA_GROUP, NetlinkListener::NETLINK_FORMAT_BINARY, false)) == nullptr) {
        ALOGW("Unable to open qlog quota socket, check if xt_quota2 can send via UeventHandler");
        // TODO: return -1 once the emulator gets a new kernel.
    }

    if ((mStrictHandler = setupSocket(&mStrictSock, NETLINK_NETFILTER,
            0, NetlinkListener::NETLINK_FORMAT_BINARY_UNICAST, true)) == nullptr) {
        ALOGE("Unable to open strict socket");
        // TODO: return -1 once the emulator gets a new kernel.
    }

    return 0;
}
NetlinkHandler *NetlinkManager::setupSocket(int *sock, int netlinkFamily,
    int groups, int format, bool configNflog){
    //初始化sock 地址
    struct sockaddr_nl nladdr;
    int sz = 64 * 1024;
    int on = 1;

    memset(&nladdr, 0, sizeof(nladdr));
    nladdr.nl_family = AF_NETLINK;
    // Kernel will assign a unique nl_pid if set to zero.
    nladdr.nl_pid = 0;
    nladdr.nl_groups = groups;
    // 创建socket
    if ((*sock = socket(PF_NETLINK, SOCK_DGRAM | SOCK_CLOEXEC, netlinkFamily)) < 0) {
    ALOGE("Unable to create netlink socket for family %d: %s", netlinkFamily, strerror(errno));
    return nullptr;}
    '''
    // 绑定sock
    if (bind(*sock, (struct sockaddr *) &nladdr, sizeof(nladdr)) < 0) {
    ALOGE("Unable to bind netlink socket: %s", strerror(errno));
    close(*sock);
    return nullptr;}
    // 
    NetlinkHandler *handler = new NetlinkHandler(this, *sock, format);
    if (handler->start()) {
        ALOGE("Unable to start NetlinkHandler: %s", strerror(errno));
        close(*sock);
        return nullptr;
    }
}

NetlinkHandler 继承了SocketListener
pthread_mutex_t         mClientsLock;

void SocketListener::init(const char *socketName, int socketFd, bool listen, bool useCmdNum) {
    mListen = listen;
    mSocketName = socketName;
    mSock = socketFd;
    mUseCmdNum = useCmdNum;
    pthread_mutex_init(&mClientsLock, nullptr);
}
对于mClientsLock 声明后直接能用：
在C++中，当你声明一个变量，包括一个类的实例时，不管其构造函数在声明点是否已经被明确调用，该变量都会在内存中被分配空间，并且其地址可以被取用和使用。
当你声明pthread_mutex_t mClientsLock;时，这条语句做了两件事：

内存分配：它为类pthread_mutex_t的一个实例分配内存空间。这意味着有一个特定的内存位置被设置来存储属于mClientsLock的所有数据。

构造函数调用：它自动调用了默认构造函数A::A()，这就是为什么你会看到控制台上打印出"the A() call"。这是对象初始化过程的一部分。