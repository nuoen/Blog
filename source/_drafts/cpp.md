开发规范

```

int AndroidBitmap_getInfo(AndroidBitmapInfo* bitmapInfo){

if(bitmapInfo == NULL){
    return -1;
}
return 0;
}

```

1.通过指针在方法里赋值
2.一定要考虑健壮性
3.利用返回值进行异常抛出
4.不要修改传入的指针，可以加const ，及 AndroidBitMapInfo* const bitmapInfo


二级指针
void set(char** str){
    *str = "test";
}

void main(){
    char* name = NULL;
    set(&name);

    printf("name = %s",name);
}
虽然NULL地址无法赋值，但是这里是没问题的
这里是没问题的，因为传入的是二级指针，所以
*str = "test",改变的是name这个指针变量的值


const 在c和cpp中的区别

二级指针的内存模式

指针数组：数组指针指向的数组元素的首地址
```c
struct FILE
{
    char* filename;
    int length;

}

void main(){
    char* name ="test";
    //二级指针，看成二维数组
    char** nameP = &name;

    //比如压缩文件，指定两个东西FILE ,一个是输入，一个是输出
    //定义一个FILE* 指针数组

    FILE* file[2]={{"in.mp4"},{"out.mp4"}};

    FILE** file1;//另外一种定义的方式（指针数组）（二级指针）其实是同一个概念，其实不是（栈中表现不同）
    //数组指针指向的是数组元素的首地址

    int number =100;
    int* numberP = &number;//一级指针，一级数组
    numberP++;
    printf("number is %d",*numberP);

}
```
//单独的来拿字符串数组来讲

```c
void main(){

    char* name[10] = {"aaa","bbbb","cccc"};//后面默认指向的是null指针 都是静态常量

    char** name = {"aaa","bbbb"};//报错
    //直接去赋值那么c和cpp的编译器会识别为二级指针，所以指针数组/二级指针不同


    char name[10][30] = {"aaa","bbbb","cccc"};//这几个都是从静态常量区copy到栈的buffer里面的
    //如果 name 用char** 去接 

    int len =4;
    char** params = malloc(sizeof(char*)*len);//开辟二维数组
    //开辟一维度数组
    for(int i=0;i<len;i++){
        params[i] = malloc(sizeof(char*)*100);
    }

    //释放数组
    for(int i=0;i<len;i++){
        if(params[i]!=NULL){
            free(params[i]);
            params[i]=NULL;
        }
    }
    if(params!=NULL){
        free(params);
        params=NULL;
    }
}
```
结构提赋值
在c中等于相当于是内容赋值操作
```c
typedef struct{
    char name[10];
    int age;
}Student;

Student stu1 = {"name",24};
Student stu2;
stu2 = stu1;// = 赋值的操作，java中stu2对象会变成stu1,但是c中不会，stu1 和 stu2是不同的结构体 

//但是如果结构体是
tydef struct {
    char* name;
    int age;
}Student;
Student stu1 = {"name",24};//会报错
//从内存角度来看？


void copyTo(Student* stu1,Student *stu2){
    *stu1 = *stu2;//指针的赋值运算是一个浅拷贝，即stu1 和 stu2的name变量不同，但是同时指向一个地址（即变量的值是同一个）因此当释放stu1.name后，stu1.name 值对应的那一块区域已经被释放了 ，如果继续释放stu2.name，就会出错误。
    
    }
```
cpp
```




```

