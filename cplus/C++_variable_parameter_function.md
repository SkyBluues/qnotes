# C++不定参数函数

### 用到的Api

C语言里面提供了三个宏用于实现不定参数函数：
```C++
va_start(ap,v)
va_arg(ap,t)
va_end(ap)
```
具体的Api意义和用法可查看官方文档，这里写一段示例代码。这样的代码都是固定结构，看代码会更直接一点。

### 不定参数函数示例

实现一个不定参数的加法函数：
```C++
int AddAll(int number, ...) {
	int result = 0;
	if (number <= 0) return result;
	va_list arg_ptr;
	va_start(arg_ptr, number);
	for (; number > 0; --number) {
		int ele = va_arg(arg_ptr, int);
		result += ele;
	}
	va_end(arg_ptr);
	return result;
}
```

### 自定义格式化输出
bitcoin源码中的错误输出：
```C++
bool error(const char* format, ...)
{
    char buffer[50000];
    int limit = sizeof(buffer);
    va_list arg_ptr;
    va_start(arg_ptr, format);
    int ret = _vsnprintf(buffer, limit, format, arg_ptr);
    va_end(arg_ptr);
    if (ret < 0 || ret >= limit)
    {
        ret = limit - 1;
        buffer[limit-1] = 0;
    }
    printf("ERROR: %s\n", buffer);
    return false;
}
```
