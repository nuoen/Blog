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

字符串强化
```
   char* p="hello";
   char buffer[]="hello";
   printf(p);
   printf(buffer);
```
汇编：
```
main:                                   // @main
	.cfi_startproc
// %bb.0:
	sub	sp, sp, #64                     // =64
	stp	x29, x30, [sp, #48]             // 16-byte Folded Spill
	add	x29, sp, #48                    // =48
	.cfi_def_cfa w29, 16
	.cfi_offset w30, -8
	.cfi_offset w29, -16
	mov	w8, wzr
	str	w8, [sp, #20]                   // 4-byte Folded Spill
	stur	wzr, [x29, #-4]
	adrp	x8, .L.str
	add	x8, x8, :lo12:.L.str
	stur	x8, [x29, #-16]
	adrp	x8, .L__const.main.buffer
	add	x8, x8, :lo12:.L__const.main.buffer
	ldr	w9, [x8]
	add	x10, sp, #24                    // =24
	str	x10, [sp, #8]                   // 8-byte Folded Spill
	str	w9, [sp, #24]
	ldrh	w8, [x8, #4]
	strh	w8, [sp, #28]
	ldur	x0, [x29, #-16]
	bl	printf
	ldr	x0, [sp, #8]                    // 8-byte Folded Reload
	bl	printf
	ldr	w0, [sp, #20]                   // 4-byte Folded Reload
	ldp	x29, x30, [sp, #48]             // 16-byte Folded Reload
	add	sp, sp, #64                     // =64
	ret
.Lfunc_end2:
	.size	main, .Lfunc_end2-main
	.cfi_endproc
                                        // -- End function
	.type	.L.str,@object                  // @.str
	.section	.rodata.str1.1,"aMS",@progbits,1
.L.str:
	.asciz	"hello"
	.size	.L.str, 6

	.type	.L__const.main.buffer,@object   // @__const.main.buffer
.L__const.main.buffer:
	.asciz	"hello"
	.size	.L__const.main.buffer, 6
```
(lldb) register read sp
      sp = 0x0000007ffffffd80
(lldb) memoey read 0x0000007ffffffd80
error: 'memoey' is not a valid command.
(lldb) memory read 0x0000007ffffffd80
0x7ffffffd80: 00 00 00 00 00 00 00 00 98 fd ff ff 7f 00 00 00  ................
0x7ffffffd90: 00 00 00 00 00 00 00 00 68 65 6c 6c 6f 00 00 00  ........hello...
(lldb) memory read -c 0x60  0x0000007ffffffd80
0x7ffffffd80: 00 00 00 00 00 00 00 00 98 fd ff ff 7f 00 00 00  ................
0x7ffffffd90: 00 00 00 00 00 00 00 00 68 65 6c 6c 6f 00 00 00  ........hello...
0x7ffffffda0: 78 55 55 55 55 00 00 00 00 00 00 00 00 00 00 00  xUUUU...........
0x7ffffffdb0: c0 fd ff ff 7f 00 00 00 70 63 aa f4 7f 00 00 00  ........pc......
0x7ffffffdc0: 10 fe ff ff 7f 00 00 00 f4 66 55 55 55 00 00 00  .........fUUU...
0x7ffffffdd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................


c++