# firasan
A faster,smaller,asan

#                                                                 FirASAN

## 结论:

|          | 内存消耗  | CPU消耗   |
| :------- | --------- | --------- |
| ASAN原版 | +100-150% | +200%以上 |
| FirASAN  | +10%      | 1%以内    |
|          |           |           |

## 背景:

AddressSanitize具有强大的宕机分析定位能力，但因为拦截的函数过于底层及常用，导致只能在开发版中使用，无法帮助生产环境定位。虽然有blacklist,suppressions等功能，但因为需要过滤的第三方库，第三方函数实在太多，导致性能问题，并且有些函数也无法过滤。  所以需要一个定制版的ASAN.  

## 原理:

- 自定义fir_new,fir_delete,来分配及释放内存，业务逻辑代码不再使用默认的new,delete。
- 取消asan默认拦截的new,delete,malloc,free,及一系列函数因为第三方库，STL等库，默认可以依赖，不需要拦截
- 让asan只对fir_new分配的内存进行空指针，野指针，堆越界访问，进行拦截，可以大大减小消耗。
- 附图1，修改前的函数拦截情况，附图2，修改以后的函数拦截情况 
- FirASAN缺陷:不能捕获第三方库，STL库内部的产生的野指针，访问越界，以及内存泄漏。

## 使用:

- 加入fir_new,fir_delete实现

```c++
fir_new.h:
#ifndef _FIR_NEW_H
#define _FIR_NEW_H
extern void* operator new(size_t size, const char* file, int line);
extern void* operator new[](std::size_t size, const char* file, int line);

extern void operator delete(void* p, const char* f, int line);
extern void operator delete[](void* p , const char* f, int line);

#define FIR_NEW new(__FILE__, __LINE__)
#define SAFE_DELETE(x) { if (x) { operator delete (x,__FILE__,__LINE__); (x) = NULL; } }
#define SAFE_DELETE_VEC(x) { if (x) { operator delete [] (x,__FILE__,__LINE__); (x) = NULL; } }
#endif

fir_new.cpp:
extern "C"
{
	void* __lib_fir_malloc(size_t size)
	{
		return malloc(size);
	}
	void __lib_fir_free(void* ptr)
	{
		free(ptr);
	}

	void* __interceptor_fir_malloc(size_t size) __attribute__((weak, alias("__lib_fir_malloc")));
	void __interceptor_fir_free(void* ptr) __attribute__((weak, alias("__lib_fir_free")));
}

void* operator new(size_t size, const char* file, int line)
{
    return __interceptor_fir_malloc(size);
}

void operator delete(void* pointer, const char* file, int line)
{
    __interceptor_fir_free(pointer);
}

void  operator delete[](void* pointer, const char* file, int line)
{
    __interceptor_fir_free(pointer);
}
```

- asan启动选项

  同时使用disable_coredump, unmap_shadow_on_exit,abort_on_error，才能在64位系统上，生成core。

  直接给个案例:

  ```c++
  `ASAN_OPTIONS="halt_on_error=0:disable_coredump=0:log_path=./asan:log_exe_name=true:abort_on_error=1:unmap_shadow_on_exit=1" ./SuperServer`
  ```

  为避免生产环境下不方便设置环境变量，可在程序里，直接实现weak函数

  ```c++
  const char* __asan_default_options() {
  return "halt_on_error=0:disable_coredump=0:log_path=./asan:log_exe_name=true:abort_on_error=1:unmap_shadow_on_exit=1\";
  }
  ```

  

