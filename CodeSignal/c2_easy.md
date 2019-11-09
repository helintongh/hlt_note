#题目描述

这道题很简单就是回文判断，然后仅有一个字母的也判断为回文。
这里有一个坑点,c的函数strlen结束符也算所以慎用。

inputString.length()/2-1这个值是回文比较经典的一个限度。我们只需遍历前一半让其与对应的后一半的字符进行比较即可，同时我运用了哨兵函数来记录是否为回文。
```C++
bool checkPalindrome(std::string inputString) {
    int flag = 0;
    int last = inputString.length()-1;
    if(last == 0)
        return true;
    for(int i=0; i<=inputString.length()/2-1;i++){
        //last--;
        if(inputString[i]!= inputString[last]){
            flag++;
        }
        last--;
    }
    if(flag == 0)
        return true;
    else
        return false;
}

```