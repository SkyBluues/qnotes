# makefile笔记
### make概述
make的基本模型如下：
```makefile
targets: objects
    commands
```
简单的说就是我们要有一个目标 *target*（也可以是多个），生成这个目标需要依赖于多个对象 *object*，生成这个目标的方法就是 *command*。这里的*object*也可以有自己的依赖，这样层层递归，直到最底层，make会根据这个递归关系层层查找，然后从最底层的目标开始生成，然后层层往上生成最上层的目标（终极目标）。make会检测这些目标所依赖的对象是否被改动过，如果被改动，就运行命令*command*重新生成*target*。make其他的操作都是在这样的模式下进行的。make不限于做代码编译，凡是符合上面的模式的工作都可以做。



### Tips
- **@:** makefile中可以添加shell命名，如果在shell命令之前加上 *@* ，会让make不输出这条命令，只输出命令执行的结果。

- **cut:** shell 命令 cut，通过参数设置，能够显示文本的指定列/字符。
  显示文件 *aaa.txt* 的第二列，以空格为分隔符：
  ```bash
  cut -f2 -d' ' aaa.txt
  ```
  参数的用法：*-f-4*表示显示1到4列；
  如果显示指定字符：

  ```bash
  cut -c2 aaa.txt
  ```
  表示显示每行的第二个字符，*-c*参数的运用和*-f*一样。

- **wildcard:** makefile中的命令，用于获取当前目录下所有文件的文件路径（包括文件名）。用法：
  ```shell
  SOURCES = $(wildcard *.c)
  ```

- **notdir:** makefile中的命令。用于删除路径，保留文件名。（也就是把最后一个/之前的内容删除）。用法：
  ```shell
  FILES =$(notdir $(SOURCES))
  ```

- **patsubst:** makefile中的命令。用于替换操作。例如：
  ```shell
  OBJS = $(patsubst %.c，%.o，$(SOURCES))
  ```
  表示将*SOURCES*中所有文件名的 *.c* 换成 *.o*。还有另一种写法：
  ```shell
  OBJS = $($(SOURCES): %.c = %.o)
  ```

- **g++ -MP:** *g++* 的编译选项 *-MP*。以下是 *g++* 文档的说明：This option instructs CPP to add a phony target for each dependency other than the main file, causing each to depend on nothing.  These dummy rules work around errors make gives if you remove header files without updating the Makefile to match. This is typical output:
  ```makefile
  test.o: test.c test.h    
  test.h:
  ```
  这就是为了当删除了头文件 *test.h* 能够触发 *test.o* 重新编译。

- **g++ -MMD:** *g++* 的编译选项。 首先是 *-MM*, 它会在编译阶段打印出编译对象的依赖（除了系统文件）；*-MMD*, 将产生的依赖关系写入 **.d* 文件。通过 *g++* 的这种机制，能够自动发现文件之间的依赖关系，某个编译对象依赖的文件发生变化时，触发重新编译。

- 在makefile中，如果存在两个相同的目标，会将其合并，并选择较新的命令执行。

- 在make执行过程中，如果include的文件发生变化，会让make重新读取整个makefile。这也是make能够根据 *g++* 生成的依赖自动编译的原因。



### 一些特殊变量

**$^**
所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，那个这个变量
会去除重复的依赖目标，只保留一份。

**$@**
表示规则中的目标文件集。在模式规则中，如果有多个目标，那么，"$@"就是匹配于
目标中模式定义的集合

**$?**
所有比目标新的依赖目标的集合。以空格分隔。 

**$<**
依赖目标中的第一个目标名字。如果依赖目标是以模式（即"%"）定义的，那么"$<"将
是符合模式的一系列的文件集。注意，其是一个一个取出来的。



### make中的赋值

- ?= 变量如果没有被赋值过，就赋予右边的值
- := 覆盖之前的值
- = 最基本的赋值
- += 添加等号后面的值
- =和:=的区别：=是将makefile最终展开后才确定值，:=是立即能确定值



### 很好的解释了自动生成依赖的文章 

http://docs.linuxtone.org/ebooks/C&CPP/c/ch22s04.html



### 一个makefile的例子

本例是别人放在github上的一个项目，通过解释makefile文件了解其工作原理。
假设我们有如下工作目录：
+-- inclue
|   +-- qvector.h
+-- src
|   +-- main.cpp
|   +-- qvector.cpp
+-- makefile

其中makefile是这样写的：

```makefile
#################################################################
# 这部分都是定义一些变量，便于后续的操作
#################################################################
CXX ?= g++         
SRC_PATH = src      
BUILD_PATH = build       
BIN_PATH = $(BUILD_PATH)/bin     
BIN_NAME = runner  
SRC_EXT = cpp       

#################################################################
# 进行一些路径准备
# SOURCEs保存了当前目录下所有以 cpp 结尾的文件的路径
# OBJECTS通过模式匹配生成了每个源文件对应的目标文件路径
# DEPS是依赖文件的路径
#################################################################
SOURCES = $(shell find $(SRC_PATH) -name '*.$(SRC_EXT)' | sort -k 1nr | cut -f2-) 
OBJECTS = $(SOURCES:$(SRC_PATH)/%.$(SRC_EXT)=$(BUILD_PATH)/%.o)   
DEPS = $(OBJECTS:.o=.d)      

#################################################################
# 编译选项的配置
#################################################################
COMPILE_FLAGS = -std=c++11 -Wall -Wextra -g  # -Wall-> display warning, -Wextra->display other warning
INCLUDES = -I include/ -I /usr/local/include        
LIBS =         

#################################################################
# 定义了几个伪目标
# 这里的伪命令是层层递归调用的：
# default_target -> release -> dirs -> dirs -> all ->  $(BIN_PATH)/$(BIN_NAME)
# -> (BUILD_PATH)/%.o
#################################################################
# 在makefile中，如果make没有指定目标，第一个出现的目标就是终极目标，所以这里是default_target。
# default_target依赖于release，然后去检测release
.PHONY: default_target      
default_target: release  

# release依赖于dirs，当依赖执行完成后，会执行它的命令 make all
.PHONY: release
release: export CXXFLAGS := $(CXXFLAGS) $(COMPILE_FLAGS)
release: dirs               
	@$(MAKE) all        

# dirs用于创建两个目录
.PHONY: dirs
dirs:                                 
	@echo "Creating directories"
	@mkdir -p $(dir $(OBJECTS))
	@mkdir -p $(BIN_PATH)

# 用于清理文件的伪命令
.PHONY: clean
clean:
	@echo "Deleting $(BIN_NAME) symlink"
	@$(RM) $(BIN_NAME)
	@echo "Deleting directories"
	@$(RM) -r $(BUILD_PATH)
	@$(RM) -r $(BIN_PATH)

# all目标依赖于$(BIN_PATH)/$(BIN_NAME)，当依赖解决后，执行命令，产生一个软链接
.PHONY: all                                             
all: $(BIN_PATH)/$(BIN_NAME)        
	@echo "Making symlink: $(BIN_NAME) -> $<"
	@$(RM) $(BIN_NAME)
	@ln -s $(BIN_PATH)/$(BIN_NAME) $(BIN_NAME)

# $(BIN_PATH)/$(BIN_NAME)依赖于$(OBJECTS)，当所有的对象都更新完成后，执行命令，链接生成
# 可执行文件。
$(BIN_PATH)/$(BIN_NAME): $(OBJECTS)    
	@echo "Linking: $@"
	$(CXX) $(OBJECTS) -o $@

# 加载依赖文件，第一次不存在。
-include $(DEPS)            

# 每个目标对象对应一个源文件，且会生成对应的依赖文件，这里的规则会和include的规则进行合并
$(BUILD_PATH)/%.o: $(SRC_PATH)/%.$(SRC_EXT)         
	@echo "Compiling: $< -> $@"
	$(CXX) $(CXXFLAGS) $(INCLUDES) -MP -MMD -c $< -o $@
```



### 自己的通用模板

```makefile
EXE_NAME:=main
SRC_PATH:=.
BUILD_PATH:=./build
BIN_PATH:=$(BUILD_PATH)/bin
EXE:=$(BIN_PATH)/$(EXE_NAME)

SOURCES:=$(shell find $(SRC_PATH) -name '*.cpp')
OBJS:=$(addprefix $(BUILD_PATH)/,$(SOURCES:%.cpp=%.o))
DEPS:=$(addprefix $(BUILD_PATH)/,$(SOURCES:%.cpp=%.d))
OBJS_DIR:=$(addprefix $(BUILD_PATH)/,$(shell dirname $(SOURCES) | sort | uniq))

LIBS:=
CPPFLAGS+= -g -Wall -std=c++11 -I ./include
CC:=g++

all:$(EXE)
$(shell mkdir -p $(OBJS_DIR))
$(shell mkdir -p $(BIN_PATH))
-include $(DEPS)

$(EXE): $(OBJS)
	$(CC) $(CPPFLAGS) -o $@ $^ $(LIBS)

$(OBJS): $(BUILD_PATH)/%.o: $(SRC_PATH)/%.cpp
	$(CC) -o $@ $(CPPFLAGS) -c $<

$(DEPS): $(BUILD_PATH)/%.d: $(SRC_PATH)/%.cpp
	@set -e; rm -f $@; \
	$(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \
	sed 's,.*\.o[ :]*,$(patsubst %.d,%.o,$@) $@ : ,g' < $@.$$$$ > $@; \
	rm -f $@.$$$$

.PHONY: clean
clean:
	rm -rf ./build

.PHONY: test
test:
	@echo $(DEPS)
	@echo $(OBJS)
	@echo $(SOURCES)
	@echo $(SRC_PATH)
	@echo $(BUILD_PATH)
	@echo $(OBJS_DIR)


```

