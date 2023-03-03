## substr()：字符串截取

```c++
string ret = str.substr(pos1, pos2)
```

返回值为截取的字符串，但显然只是截取了一个副本，不改变原字符串。

## find()：字符串查找

```c++
int pos = str.find("str_find", pos_begin)
```

返回值为在在str中pos_find_begin开始，字符串str_find出现的第一个位置。

如果没找到，返回的是string类里的npos标志(size_t类型)，在不同计算机中这个标志的值不一样。

```c++
while(s.find("aa")!=string::npos)
```

