cdex001  https://github.com/AlienwareHe/RDex
在安卓9中为了减少内存的使用量，引入了cdex，magic为cdex001，这导致我们dump出来的dex不是一个正常的Dex结构，无法被jadx等反编译。



adb shell pm list packages 展示包
展示包所在的路径
adb shell pm path com.android.systemui


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
