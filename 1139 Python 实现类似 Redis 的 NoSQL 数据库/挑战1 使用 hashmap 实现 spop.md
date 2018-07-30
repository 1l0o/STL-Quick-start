# 完善和补充命令

## 一、说明

在`command_handle.py`文件中，我们的`SPOP`功能不够完善。我们删除的是第一个元素，而`SPOP`真正的功能是随机删除元素。

在代码中，也没有读取指定key所有值得`SMEMBERS`和`LRANGE`命令。

## 二、挑战

完善真正的SPOP功能。并增加`SMEMBERS`和`LRANGE`命令功能。


## 三、提示

- 使用hashmap完成`SPOP`
- 读取memory值完成`SMEMBERS`和`LRANGE`

## 四、代码

完整代码可以在GitHub上查看，地址：[Challenge1](https://github.com/KeepitReal555/PyRedis/Cha1)

```
$ git clone -b Cha1 https://github.com/KeepitReal555/PyRedis.git 
```

