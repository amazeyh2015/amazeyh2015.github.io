---
layout: post
title: "The CoreSymbolication Framework"
date: 2025-10-02 10:00:00 +0800
categories: sharing
---

# What is CoreSymbolication

CoreSymbolication是一个用于符号化的私有框架，位于：
> /System/Library/PrivateFrameworks/CoreSymbolication.framework/Versions/A/CoreSymbolication

CSTypeRef是CoreSymbolication中定义的一个基础类型，其结构为：

```c
typedef struct {
    uint64_t csCppData;
    uint64_t csCppObj;
} CSTypeRef;
```

CoreSymbolication中大部分的类型都是基于CSTypeRef来定义的，这里列举一些结构的定义：

```c
typedef struct {
    uint64_t location;
    uint64_t length;
} CSRange;

typedef struct {
    int cpu_type;
    int cpu_subtype;
} CSArchitecture;

typedef CSTypeRef CSSymbolicatorRef;
typedef CSTypeRef CSSymbolOwnerRef;
typedef CSTypeRef CSRegionRef;
typedef CSTypeRef CSSymbolRef;
typedef CSTypeRef CSSourceInfoRef;
```

通过上面列举的结构，我们了解到了CoreSymbolication中的几个重要的概念：
*   Symbolicator：相当于一个提供上层操作的Manager，里面可以包含一个或多个SymbolOwner
*   SymbolOwner：符号的拥有者，用来描述符号的归属，通常是符号所在的二进制文件信息
*   Region：符号所在的区域，用来描述符号更具体的位置，比如\_\_TEXT.\_\_text
*   Symbol：符号，是我们做符号化最终要获取的数据
*   SourceInfo：用于描述符号所在的源文件的信息，比如文件名称、路径、行号等

# How to use CoreSymbolication

先使用dlopen函数在项目中加载这个动态库

```c
void *handle = dlopen("/System/Library/PrivateFrameworks/CoreSymbolication.framework/Versions/A/CoreSymbolication", RTLD_LAZY);
```

然后使用dlsym函数获取关键函数的地址，这里介绍一些关键函数

```c
// 根据CPU架构的名称获取架构对象
CSArchitecture (*CSArchitectureGetArchitectureForName)(const char *name);
// 根据Signature创建Symbolicator
CSSymbolicatorRef (*CSSymbolicatorCreateWithSignature)(CFDataRef signature);
// 根据dsym或excitable创建Symbolicator
CSSymbolicatorRef (*CSSymbolicatorCreateWithPathAndArchitecture)(const char *path, CSArchitecture architecture);
// 获取Symbolicator中的Owner
CSSymbolOwnerRef (*CSSymbolicatorGetSymbolOwner)(CSSymbolicatorRef symbolicator);
// 获取Owner的名称
const char * (*CSSymbolOwnerGetName)(CSSymbolOwnerRef owner);
// 获取Owner的起始地址
uint64_t (*CSSymbolOwnerGetBaseAddress)(CSSymbolOwnerRef owner);
// 在Owner中枚举某个address的符号和源文件信息
void (*CSSymbolOwnerForEachStackFrameAtAddress)(CSSymbolOwnerRef owner, uint64_t address, id iterator);
// 获取Symbol的Name
const char * (*CSSymbolGetName)(CSSymbolRef symbol);
// 获取Symbol的MangledName
const char * (*CSSymbolGetMangledName)(CSSymbolRef symbol);
// 获取Symbol的Region
CSRegionRef (*CSSymbolGetRegion)(CSSymbolRef symbol);
// 获取Region的Name，例如：__TEXT __text
const char * (*CSRegionGetName)(CSRegionRef region);
// 获取Symbol所在的源文件名称
const char * (*CSSourceInfoGetFilename)(CSSourceInfoRef sourceInfo);
// 获取Symbol所在的源文件路径
const char * (*CSSourceInfoGetPath)(CSSourceInfoRef sourceInfo);
// 获取Symbol所在的源文件行号
uint32_t (*CSSourceInfoGetLineNumber)(CSSourceInfoRef sourceInfo);
```

完整代码如下

```c
#import <dlfcn.h>

Boolean (*CSIsNull)(CSTypeRef cs);
CSTypeRef (*CSRetain)(CSTypeRef cs);
void (*CSRelease)(CSTypeRef cs);

CSArchitecture (*CSArchitectureGetArchitectureForName)(const char *name);

CSSymbolicatorRef (*CSSymbolicatorCreateWithSignature)(CFDataRef signature);
CSSymbolicatorRef (*CSSymbolicatorCreateWithPathAndArchitecture)(const char *path, CSArchitecture architecture);
CSSymbolOwnerRef (*CSSymbolicatorGetSymbolOwner)(CSSymbolicatorRef symbolicator);
CFDataRef (*CSSymbolicatorCreateSignature)(CSSymbolicatorRef symbolicator);

const char * (*CSSymbolOwnerGetName)(CSSymbolOwnerRef owner);
uint64_t (*CSSymbolOwnerGetBaseAddress)(CSSymbolOwnerRef owner);
void (*CSSymbolOwnerForEachStackFrameAtAddress)(CSSymbolOwnerRef owner, uint64_t address, id iterator);

const char * (*CSSymbolGetName)(CSSymbolRef symbol);
const char * (*CSSymbolGetMangledName)(CSSymbolRef symbol);
CSRegionRef (*CSSymbolGetRegion)(CSSymbolRef symbol);

const char * (*CSRegionGetName)(CSRegionRef region);

const char * (*CSSourceInfoGetFilename)(CSSourceInfoRef sourceInfo);
const char * (*CSSourceInfoGetPath)(CSSourceInfoRef sourceInfo);
uint32_t (*CSSourceInfoGetLineNumber)(CSSourceInfoRef sourceInfo);

void * Bind(void *handle, const char *name) {
    void *symbol = dlsym(handle, name);
    NSCAssert(symbol != NULL, @"%s", dlerror());
    return symbol;
}

void * Open(const char *path, int mode) {
    void *handle = dlopen(path, mode);
    NSCAssert(handle != NULL, @"%s", dlerror());
    return handle;
}

void * OpenLazy(const char *path) {
    return Open(path, RTLD_LAZY);
}

void openCoreSymbolication(void) {
    void *handle = OpenLazy("/System/Library/PrivateFrameworks/CoreSymbolication.framework/Versions/A/CoreSymbolication");

    CSIsNull = Bind(handle, "CSIsNull");
    CSRetain = Bind(handle, "CSRetain");
    CSRelease = Bind(handle, "CSRelease");

    CSArchitectureGetArchitectureForName = Bind(handle, "CSArchitectureGetArchitectureForName");

    CSSymbolicatorCreateWithSignature = Bind(handle, "CSSymbolicatorCreateWithSignature");
    CSSymbolicatorCreateWithPathAndArchitecture = Bind(handle, "CSSymbolicatorCreateWithPathAndArchitecture");
    CSSymbolicatorGetSymbolOwner = Bind(handle, "CSSymbolicatorGetSymbolOwner");
    CSSymbolicatorCreateSignature = Bind(handle, "CSSymbolicatorCreateSignature");

    CSSymbolOwnerGetName = Bind(handle, "CSSymbolOwnerGetName");
    CSSymbolOwnerGetBaseAddress = Bind(handle, "CSSymbolOwnerGetBaseAddress");
    CSSymbolOwnerForEachStackFrameAtAddress = Bind(handle, "CSSymbolOwnerForEachStackFrameAtAddress");

    CSSymbolGetName = Bind(handle, "CSSymbolGetName");
    CSSymbolGetMangledName = Bind(handle, "CSSymbolGetMangledName");
    CSSymbolGetRegion = Bind(handle, "CSSymbolGetRegion");

    CSRegionGetName = Bind(handle, "CSRegionGetName");

    CSSourceInfoGetFilename = Bind(handle, "CSSourceInfoGetFilename");
    CSSourceInfoGetPath = Bind(handle, "CSSourceInfoGetPath");
    CSSourceInfoGetLineNumber = Bind(handle, "CSSourceInfoGetLineNumber");
}
```

### 使用dSYM符号化

首先获取dSYM文件中binary的路径

```c
const char *dsymPath = "xxx/xxx.app.dSYM/Contents/Resources/DWARF/xxx";
```

获取CPU的架构

```c
CSArchitecture arch = CSArchitectureGetArchitectureForName("arm64");
```

创建Symbolicator

```c
CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithPathAndArchitecture(dsymPath, arch);
```

获取Owner

```c
CSSymbolOwnerRef owner = CSSymbolicatorGetSymbolOwner(symbolicator);
```

根据slide修正address

```c
uint64_t loadAddress = 0x104ba8000; // use your real load address instead
uint64_t baseAddress = CSSymbolOwnerGetBaseAddress(owner);
uint64_t slide = loadAddress - baseAddress;
address -= slide;
```

在Owner中查找address对应的符号信息和源文件信息

```c
CSSymbolOwnerForEachStackFrameAtAddress(owner, address, ^(CSSymbolRef symbol, CSSourceInfoRef sourceInfo) {
    // use symbol and sourceInfo
});
```

如果address在某个inline函数中，则对应的符号有多个，调用CSSymbolOwnerForEachStackFrameAtAddress函数会按照外层到内层的顺序调用参数中的block

```c
// a inline function
static inline testInlineFunction(void) {
    ...
    // address for this line is 0x1234567890
    ...
}

// a instance method
- (void)testMethod {
    ...
    testInlineFunction();
    ...
}

// symbols for address 0x1234567890 are ["-[XXX testMethod]", "testInlineFunction"]
```

调用CSSymbolOwnerGetSymbolWithAddress函数只会获得最外层的符号

```c
CSSymbolRef (*CSSymbolOwnerGetSymbolWithAddress)(CSSymbolOwnerRef owner, uint64_t address);
CSSourceInfoRef (*CSSymbolOwnerGetSourceInfoWithAddress)(CSSymbolOwnerRef owner, uint64_t address);
```

不再使用Symbolicator时，需要释放Symbolicator

```c
CSRelease(symbolicator);
```

完整代码如下

```objective-c
void symbolicationWithDsym(void) {
    uint64_t loadAddress = 0x104ba8000; // use your real load address instead
    uint64_t address = 0x123456789; // use your real address instead
    const char *dsymPath = "xxx/xxx.app.dSYM/Contents/Resources/DWARF/xxx"; // use your real path instead
    CSArchitecture arch = CSArchitectureGetArchitectureForName("arm64");
    CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithPathAndArchitecture(dsymPath, arch);
    CSSymbolOwnerRef owner = CSSymbolicatorGetSymbolOwner(symbolicator);
    uint64_t baseAddress = CSSymbolOwnerGetBaseAddress(owner);
    printf("base address: %llx\n", baseAddress);
    uint64_t slide = loadAddress - baseAddress;
    address -= slide;
    const char *ownerName = CSSymbolOwnerGetName(owner);
    printf("owner name: %s\n", ownerName);

    CSSymbolOwnerForEachStackFrameAtAddress(owner, address, ^(CSSymbolRef symbol, CSSourceInfoRef sourceInfo) {
        const char *symbolName = CSSymbolGetName(symbol);
        const char *symbolMangledName = CSSymbolGetMangledName(symbol);
        printf("symbol name: %s\n", symbolName);
        printf("symbol mangled name: %s\n", symbolMangledName);
        const char *fileName = CSSourceInfoGetFilename(sourceInfo);
        const char *filePath = CSSourceInfoGetPath(sourceInfo);
        uint32_t line = CSSourceInfoGetLineNumber(sourceInfo);
        printf("source file name: %s\n", fileName);
        printf("source file path: %s\n", filePath);
        printf("source file line: %u\n", line);
        CSRegionRef region = CSSymbolGetRegion(symbol);
        const char *regionName = CSRegionGetName(region);
        printf("region name: %s\n", regionName);
    });

    CSRelease(symbolicator);
}
```

### 使用executable符号化

和使用dSYM的方式相比，只是把创建Symbolicator的文件从dSYM binary换成了app executable

```c
const char *executablePath = "xxx/xxx/xxx";
CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithPathAndArchitecture(executablePath, arch);
```

简单对比下使用dSYM和executable符号化的优缺点：
*   通常情况下，使用executable方式比dSYM方式首次符号化更快，并且更节省资源
*   使用executable方式需要保证打包的时候保留符号信息，局限性较大

### 使用signature符号化

还有一种比较常用的创建Symbolicator的方式是使用signature

```c
CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithSignature(signature);
```

signature是符号化用到的数据，可以从已创建的Symbolicator中导出

```c
CFDataRef signature = CSSymbolicatorCreateSignature(symbolicator);
```

可以将导出的signature数据根据architecture进行保存

```c
const char *dsymPath = "xxx/xxx/xxx";
CSArchitecture arch = CSArchitectureGetArchitectureForName("arm64");
CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithPathAndArchitecture(dsymPath, arch);
CFDataRef signature = CSSymbolicatorCreateSignature(symbolicator);
NSString *savePath = @"xxx/xxx/xxx_signature_arm64";
[(__bridge NSData *)signature writeToFile:savePath atomically:YES];

CFRelease(signature);
CSRelease(symbolicator);
```

完整代码如下

```c
void symbolicationWithSignature(void) {
    uint64_t loadAddress = 0x104ba8000; // use your real load address instead
    uint64_t address = 0x123456789; // use your real address instead
    NSString *signaturePath = @"xxx/xxx/xxx_signature_arm64"; // use your real path instead
    NSData *signature = [NSData dataWithContentsOfFile:signaturePath];
    CSSymbolicatorRef symbolicator = CSSymbolicatorCreateWithSignature((__bridge CFDataRef)(signature));
    CSSymbolOwnerRef owner = CSSymbolicatorGetSymbolOwner(symbolicator);
    uint64_t baseAddress = CSSymbolOwnerGetBaseAddress(owner);
    printf("base address: %llx\n", baseAddress);
    uint64_t slide = loadAddress - baseAddress;
    address -= slide;
    const char *ownerName = CSSymbolOwnerGetName(owner);
    printf("owner name: %s\n", ownerName);
    
    CSSymbolOwnerForEachStackFrameAtAddress(owner, address, ^(CSSymbolRef symbol, CSSourceInfoRef sourceInfo) {
        const char *symbolName = CSSymbolGetName(symbol);
        const char *symbolMangledName = CSSymbolGetMangledName(symbol);
        printf("symbol name: %s\n", symbolName);
        printf("symbol mangled name: %s\n", symbolMangledName);
        const char *fileName = CSSourceInfoGetFilename(sourceInfo);
        const char *filePath = CSSourceInfoGetPath(sourceInfo);
        uint32_t line = CSSourceInfoGetLineNumber(sourceInfo);
        printf("source file name: %s\n", fileName);
        printf("source file path: %s\n", filePath);
        printf("source file line: %u\n", line);
        CSRegionRef region = CSSymbolGetRegion(symbol);
        const char *regionName = CSRegionGetName(region);
        printf("region name: %s\n", regionName);
    });
    
    CSRelease(symbolicator);
}
```

使用dSYM符号化的一个缺点是dSYM文件通常较大，而且首次符号化的速度比较慢，这种方式既能规避dSYM方式的缺点，又有较快的符号化速度，不足之处是多了导出和保存signature的步骤，增加方案的实现复杂度

# Why use CoreSymbolication

虽然已有像atos这样的命令行工具可以用来做符号化，但和CoreSymbolication这类底层框架相比：
*   命令行工具不便于在其他框架或者项目中使用
*   底层框架提供更多的功能，可以根据项目需求灵活使用

另外，atos在内部也是使用CoreSymbolication做符号化
