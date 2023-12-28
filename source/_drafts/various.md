cdex001  https://github.com/AlienwareHe/RDex
在安卓9中为了减少内存的使用量，引入了cdex，magic为cdex001，这导致我们dump出来的dex不是一个正常的Dex结构，无法被jadx等反编译。


c:内存申请

栈：占最大内存中的2M,开辟内存的方式是静态内存开辟，比如：函数内声明变量，函数方法结束后会自动回收
堆：占最大内存中的80%，动态内存开辟，不会自动回收,必须手动回收。malloc()开辟，free()释放

利用realloc扩展内存，如果原来的内存不足以向下扩展，会重新找一块足够的内存，因此，指针指向的地址会变动。
动态申请内存时有可能会失败，所以会返回null,如果是扩展内存返回null,那么需要把malloc的内存释放掉
释放时，也务必进行空指针判断，释放完最好把指针置空
'''
//动态开辟内存
void dyMemoryTest(int num,int newNum){
    int* arrPoint= (int*)malloc(10*1024*1024*sizeof(int)); //40M

    //必须手动释放
    if(arrPoint){
        free(arrPoint);  
    }
    int* arrPoint2=(int*)malloc(num*sizeof(int));
    if(arrPoint2){
        for(int i=0;i<num;i++){
            *(arrPoint2+i)=i;
        }
    }
    int* arrPoint3 =(int *)realloc(arrPoint2,(num+newNum)*sizeof(int));
    if(arrPoint3){
        for(int i=num;i<num+newNum;i++){
            *(arrPoint3+i)=i;
        }
        for(int i=0;i<num+newNum;i++){
            LOGD("the number is %d ,the value is %d",i,*(arrPoint3+i));
        }
        //成功了，只释放旧指针就行
        free(arrPoint3);
        arrPoint3= nullptr;
    }else{
        //没有成功，释放旧指针
        if(arrPoint2){
            free(arrPoint2);
        }
    }
}
'''
adb shell pm list packages 展示包
展示包所在的路径
adb shell pm path com.android.systemui


c语言结构体大小计算规则：
1、结构体变量的首地址，必须是结构体变量中的“最大基本数据类型成员所占字节数”的整数倍。（对齐）
2、结构体变量中的每个成员相对于结构体首地址的偏移量，都是该成员基本数据类型所占字节的整数倍。（对齐）
3、结构体变量的总大小，为结构体变量中“最大基本数据类型成员所占字节数”的整数倍（补齐）

--
typedef struct Work_{
    int num;

}Work_;

如果不对自己起别名，直接用
Work_ work=new Work_(); 会报错，必须用 struct Work_ work=new Work_();
--

