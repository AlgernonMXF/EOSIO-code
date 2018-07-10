# print 

## print.h
定义应用台API，使应用可以log/print文本消息

```C++
void prints(const char*)
//打印固定长度的字符串
void prints_l(const char*, uint32_t)
//打印64位有符号整数
void printi(int64_t)
void printui(uint64_t)
//打印128位有符号整数
void printi128(const int128_t*)
void printui128(const uint128_t*)
//打印单精度浮点数
void printsf(float)
```

