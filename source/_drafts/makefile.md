下面整理了一份较为完整的 Makefile 教程，从基础语法、变量、目标与规则，到一些高级用法和注意事项，帮助你快速入门 GNU Make 以及编写高效的 Makefile。

1. Makefile 基础

Makefile 是一个用来自动构建程序的脚本文件，主要由 目标(target)、依赖(dependency) 和 命令(command) 组成。Make 工具根据 Makefile 指定的规则自动判断哪些文件需要重新构建。

基本语法格式：

目标: 依赖1 依赖2 ...
	命令

	•	目标：通常是可执行文件、库或其他需要生成的文件，也可以是伪目标（例如 clean）。
	•	依赖：构成目标所需的文件或目标列表。如果依赖比目标更新，则目标需要重新生成。
	•	命令：实际执行的 shell 命令，用于生成目标。注意：命令行前必须以 Tab 字符开始（空格不行）。

示例：

hello: hello.c
	gcc -o hello hello.c

上述规则表示：当 hello 文件需要生成时，如果 hello.c 是比 hello 更新的（或 hello 不存在），就用 gcc -o hello hello.c 命令生成。

2. 变量的使用

Makefile 中可以定义变量，以便在多个地方复用相同的内容。

变量定义：
	•	使用等号 =（延迟展开）或 :=（立即展开）定义变量。

CC := gcc
CFLAGS := -Wall -O2
SRC := main.c utils.c
OBJ := main.o utils.o

变量引用：
	•	使用 $() 或 ${} 语法引用变量，它们是等价的。例如：
	•	$(CC) 或 ${CC}
	•	当变量名与其他字符紧挨在一起时，使用 ${} 更清晰：

${OBJ}_backup  # 表示变量 OBJ 的值后面接 _backup 字符串



示例：

all: myprog

myprog: $(OBJ)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

	•	$@：表示当前规则中的目标文件名。
	•	$<：表示当前规则中的第一个依赖文件。
	•	$^：表示当前规则中的所有依赖文件（去除重复项）。

3. 目标、依赖和命令规则

多个目标和依赖：
	•	一个目标可以有多个依赖文件：

myprog: main.o utils.o lib.o
	$(CC) -o myprog main.o utils.o lib.o



伪目标（Phony Targets）：
	•	伪目标不会生成实际文件，比如 clean、all、install 等，可以用 .PHONY 声明：

.PHONY: clean
clean:
	rm -f myprog *.o



自动化变量：
	•	自动变量使规则编写更简洁：

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

这里 % 表示通配符，匹配文件的相同部分。

4. 常用的函数和高级特性

GNU Make 提供了丰富的函数和高级特性，可以用来进行字符串操作、条件判断、循环等。

字符串替换函数 $(subst from,to,text)：

FILES := file1.c file2.c file3.c
OBJS := $(subst .c,.o,$(FILES))

条件判断：

ifeq ($(DEBUG),1)
    CFLAGS += -g
else
    CFLAGS += -O2
endif

包含其他 Makefile 文件：

include config.mk

模式规则（Pattern Rules）：
	•	用于定义通用构建规则：

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@



内置目标：
	•	GNU Make 内置了一些特殊目标，例如 .PHONY、.SUFFIXES、.DEFAULT 等，可用于调整行为。

5. 完整示例：编译一个小程序

下面给出一个简单示例 Makefile，假设项目包含源文件 main.c 和 utils.c，目标为生成可执行文件 myprog。

# Makefile 示例

# 定义变量
CC := gcc
CFLAGS := -Wall -O2
SRC := main.c utils.c
OBJ := $(SRC:.c=.o)    # 使用模式替换，将 .c 转为 .o
TARGET := myprog

# 默认目标
all: $(TARGET)

# 目标生成规则：链接目标文件生成可执行文件
$(TARGET): $(OBJ)
	$(CC) $(CFLAGS) -o $@ $^

# 模式规则：将 .c 文件编译为 .o 文件
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 清理构建生成的文件
.PHONY: clean
clean:
	rm -f $(TARGET) $(OBJ)

# 安装目标（可选）
.PHONY: install
install: $(TARGET)
	install -m 755 $(TARGET) /usr/local/bin/

说明：
	•	变量 SRC 定义了所有的源代码文件。
	•	变量 OBJ 利用模式替换，将每个 .c 文件自动转换成对应的 .o 文件。
	•	默认目标 all 依赖目标可执行文件 myprog。
	•	自动化变量 $@ 表示目标文件名字， $^ 表示所有依赖文件， $< 表示第一个依赖文件。
	•	.PHONY 用来声明伪目标 clean 和 install，确保这些目标总会执行对应命令，而不会与同名文件冲突。

6. 调试和调优

打印变量值：
	•	可使用 $(info ) 函数打印变量值：

$(info SRC = $(SRC))



使用 make -n 或 make --dry-run
	•	可模拟执行，查看 Makefile 各条命令，帮助调试构建过程。

启用详细输出：
	•	执行 make V=1 有时可帮助观察命令实际执行细节（具体与项目设定有关）。

7. 总结
	•	目标、依赖和命令构成了 Makefile 的基本单元。
	•	使用变量和自动变量简化编写工作，便于维护。
	•	利用模式规则、条件判断、内置函数等高级特性可以编写通用、灵活的构建脚本。
	•	记得在命令行前使用 Tab 缩进，否则会导致错误。

通过不断调试和修改，你可以编写出适应各种工程需求的 Makefile。希望这份教程能帮助你快速上手 GNU Make 工具！