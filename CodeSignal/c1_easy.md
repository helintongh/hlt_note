# 题目描述
Given a year, return the century it is in. The first century spans from the year 1 up to and including the year 100, the second - from the year 101 up to and including the year 200, etc.

Example

For year = 1905, the output should be
centuryFromYear(year) = 20;
For year = 1700, the output should be
centuryFromYear(year) = 17.
Input/Output

[execution time limit] 0.5 seconds (cpp)

[input] integer year

A positive integer, designating the year.

Guaranteed constraints:
1 ≤ year ≤ 2005.

[output] integer

The number of the century the year is in.

这道题有两种思路，一种是通过把年份分为100年内和大于100年进行判断。第二种是通过循环不断减去100年来判断。
```C++
int centuryFromYear(int year) {
    int century;
    if(year%10 != 0)
        century = year/100 + 1;
    else if(year<=100){
        century = 2;
    }
    else
        century = year/100;
    
    return century;
}
//方法2
int centuryFromYear(int year) {
  int centuryCount = 0;
  while (year > 0){
    year = year - 100;
    centuryCount = centuryCount + 1;
  }
  return centuryCount;

}
```