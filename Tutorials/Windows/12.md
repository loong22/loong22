### 0. 编译环境

```
Windows 10 22H2 19045.6159
git version 2.50.1.windows.1
Python 3.11.0 (main, Oct 24 2022, 18:26:48) [MSC v.1933 64 bit (AMD64)] on win32
gcc version 14.2.0 (Rev3, Built by MSYS2 project)
```

### 1. 下载源码

```
wget https://github.com/su2code/SU2/archive/refs/tags/v8.3.0.tar.gz

或者

git clone https://github.com/su2code/SU2.git
```

### 2. 下载依赖库和配置项目编译选项

示例: 使用debug模式, 不启用mpi, 安装路径C:
```
python.exe ./meson.py setup build --buildtype=debug -Dwith-mpi=disabled --prefix=/c/
```

编译可选项请查看: [https://su2code.github.io/docs_v7/Build-SU2-Linux-MacOS/](https://su2code.github.io/docs_v7/Build-SU2-Linux-MacOS/)

windows平台编译说明: [https://su2code.github.io/docs_v7/Build-SU2-Windows/](https://su2code.github.io/docs_v7/Build-SU2-Windows/)

### 3. MinGW64需要修改两处代码

#### 3.1 SU2_CFD/include/output/filewriter/CTecplotBinaryFileWriter.hpp

在此头文件前面添加下列语句, 修正编译数据类型错误问题
```
#if defined(__MINGW32__)
#include <cstdint>
#endif
```

#### 3.2 externals/tecio/teciosrc/JobControl_s.h

因MinGW64自带pthread库和Windows平台的theard库冲突, 导致无法编译tecplot库文件, 修改为以下内容:

```cpp
#pragma once
#include "ThirdPartyHeadersBegin.h"
// 统一包含 pthread.h，即使在 Windows 上也是如此，以使用 MinGW 的兼容库
#include <pthread.h> 

#if defined _WIN32
#include <windows.h> // 仍然需要它，因为 GetSystemInfo 等函数还在使用
#elif defined __APPLE__
#include <sys/types.h>
#include <sys/sysctl.h>
#else
#include <unistd.h> // 为了 sysconf
#endif
#include "ThirdPartyHeadersEnd.h"
#include "MASTER.h"
#include "GLOBAL.h"
#include "Mutex_s.h"

// ------------------- 解决方案 1: 删除冲突的 typedef -------------------
// #if defined _WIN32
// typedef HANDLE pthread_t; // <-- 删除这一行，因为它与 MinGW 的 pthread.h 冲突
// #endif

struct ___2122 
{ 
    ___2664 ___2494; 
    std::vector<pthread_t> ___2648; // 现在无论在哪个平台，pthread_t 都是由 <pthread.h> 定义的类型

    static int ___2827(); 
    ___2122() { ___2494 = new ___2665(); } 
    ~___2122() { delete ___2494; } 

    struct ThreadJobData 
    { 
        ___4160 m_job; 
        ___90 m_jobData; 
        ThreadJobData(___4160 ___2118, ___90 ___2123) : m_job(___2118) , m_jobData(___2123) {} 
    };

    // ------------------- 解决方案 2 & 3: 统一函数签名和实现 -------------------
    // 统一使用 POSIX 兼容的线程函数签名
    static void* ___4162(void* data);

    void addJob(___4160 ___2118, ___90 ___2123); 
    void wait(); 
    void lock() { ___2494->lock(); } 
    void unlock() { ___2494->unlock(); } 
}; 

inline int ___2122::___2827() 
{ 
    int ___2828 = 0;
#if defined _WIN32
    SYSTEM_INFO sysinfo; 
    GetSystemInfo(&sysinfo); 
    ___2828 = static_cast<int>(sysinfo.dwNumberOfProcessors);
#elif defined __APPLE__
    int nm[2]; 
    size_t len = 4; 
    uint32_t count; 
    nm[0] = CTL_HW; 
    nm[1] = HW_AVAILCPU; 
    sysctl(nm, 2, &count, &len, NULL, 0); 
    if(count < 1) 
    { 
        nm[1] = HW_NCPU; 
        sysctl(nm, 2, &count, &len, NULL, 0); 
        if(count < 1) count = 1; 
    } 
    ___2828 = static_cast<int>(count);
#else
    ___2828 = static_cast<int>(sysconf(_SC_NPROCESSORS_ONLN));
#endif
    return ___2828; 
}

// 统一使用 POSIX 线程函数实现，`return NULL` 在这里是正确的
inline void* ___2122::___4162(void* data)
{ 
    ThreadJobData* ___2123 = reinterpret_cast<ThreadJobData*>(data); 
    ___2123->m_job(___2123->m_jobData); 
    delete ___2123; 
    return NULL; // 对于返回 void* 的函数，返回 NULL 是完全正确的，警告也会消失
} 

inline void ___2122::addJob(___4160 ___2118, ___90 ___2123) 
{ 
    lock(); 
    ___2122::ThreadJobData* threadJobData = new ThreadJobData(___2118, ___2123);

    // 统一使用 pthread_create，MinGW 会处理好底层的 Windows API 调用
    pthread_t thread; 
    pthread_create(&thread, NULL, ___4162, (void*)threadJobData); 
    ___2648.push_back(thread); // 现在 thread 是正确的 pthread_t 类型，可以 push_back

    unlock(); 
} 

inline void ___2122::wait() 
{ 
    // 注意：这里原代码的循环和加解锁逻辑可能存在竞态条件，
    // 但为保持与原文一致，我们仅修改线程等待的部分。
    // 一个更好的实现是在循环外加锁，复制整个线程列表，然后解锁去等待。
    size_t i; 
    for(i = 0; i < ___2648.size(); ++i) 
    { 
        lock(); 
        pthread_t thr = ___2648[i]; 
        unlock();

        // 统一使用 pthread_join
        pthread_join(thr, NULL);
    } 
    lock(); 
    ___2648.erase(___2648.begin(), ___2648.begin() + i); 
    unlock(); 
}
```

### 4. 执行编译

-C 后为build路径的绝对路径

```
./ninja.exe -C /c/Users/admin/Desktop/Dev/SU2/build/ install
```


