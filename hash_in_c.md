# C语言中巧用哈希
## 前言
&emsp;今天在刷 **leetcode** 的时候遇到一个 [简单题目](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)，要求求出字符串中的最大的不重复字符的字串长度。  
&emsp;这个题目本身不难，但是在处理如何判断某个字符是否已经出现过的问题上，就出现了问题。看了下其他语言都可以使用自带的 **哈希** 来解决这个问题，但是 c 呢，就有点难搞了。当然也有为 c 开发的第三方库，但是如何自己解决呢？——我想的是用数组，没错，就是 **整形数组**
## 整体思路
* 问题转化
    * 将对符号的标记转化为对数字的标记
    * 将对数字的标记再转化为对变量数位的标记——独热
* 通过位操作完成对某个特定位的标记和检测

## 原理
### 问题转化过程
* 将对**符号**的标记转变为对**数字**的标记  
在哈希中，要求键值对之间是1对1的关系。在 ASCII 中，她的编号和符号之间也是一对一的关系。有了这样的对应关系，我们就可以把题目要求的符号的标记问题，转换为对数字的处理。
* 将对**数字**的标记转换为对整形变量**位**的标记  
如果想标记一个数字，是否出现过，最直接的办法是遍历，一个更快的办法是把数字转变为相应的位，让后通过 **位操作** 来处理，当然，这也是c语言的强项  
示例：
    1. 标记数字 8
    ```c
    unsigned int hash = 0; // 初始化
    hash |= 1<<8; //将 hash 的第8为设置为1,表示8存在过了
    ```
    2. 判断 8 是否存在
    ```c
    if (hash & (unsigned int)1<<8) {
        printf("8 已存在\n")
    } else {
    printf("8 还未被标记\n")
    }
    ```
### 单个整型变量的局限性
* 无符号整形(unsigned int) 只有32位，那就意味着她能标记的范围是[1~32]——**独热表示法**, 如何标记大于32的数字呢？用数组。
### 使用无符号整型数组扩展可以标记的数字的上限
* 单个变量中标记数字时，可以直接将数字想象称在整型变量中的数位，在数组中，则需要知道应该在数字在整个数组中的数位,那么，可以这么办——通过整除确定数组下标，通过取余知道在整型变量中数位。  
示例：标记数字 120  
    1. 确定数组大小:120/32 = 3....24，则整型数组大小可谓4  
    2. 标记  
    ```c
    int index = 120 / 32;
    int position = 120 % 32;
    unsigned int hash[4] = {0};
    hash[index] |= (unsigned int)1<<position;
    ```
># NOTE
> 1. 笔者在处理的过程中使用的是无符号整形变量，当然也可使使用其他类型，诸如`char, shot, long`等，处理思路都是一样的
## 解题代码
### 接口
```c
	int s_len; // 字符串长度
	int count; //最长不重复字符串的长度

	// ASSCII码总共128
	// 找偏移位:'x'
	// 'x' 字符相对 0 偏移为：'x' - 0  = 120 -0 = 120
	// 对应 hash[]中字节的位置： index = 120 / 32 = 3
	// 对应 字节中位的位置：position = 120 $ 32 = 24
	// 则 x 可在hash中被标记：hash[index] |= 1<<position
	unsigned int hash[4] = {0}; // 哈希 32 x 4 = 128

	count = 0;
	for (s_len = 0; *(s+s_len) != '\0'; s_len++); // 获得 s 的长度

	int r_ptr = 0; // 右边界指针
	int index; // 字符的索引
	int position; // 整型变量内部偏移
	char c;
	for (int i = 0; i < s_len; i++) {

		// 滑动左边界：取消对 s[i-1]的标记
		if (i > 0) {
			c = s[i-1];
			index = (c - 0) / 32;
			position = (c - 0) % 32;
			hash[index] &= ~((unsigned int)1<<position); //将 s[i]存于 hash中
		}

		for (; r_ptr < s_len;) {
			c = s[r_ptr];

			index = (c - 0) / 32;
			position = (c - 0) % 32;
			if (hash[index] & (unsigned int)1<<position) {
				break; //已经被标记过
			} else {
				hash[index] |= (unsigned int)1<<position; //将 s[r_ptr]存于 hash中
				r_ptr++; // 滑动右边界
			}
		}
		
		count = count > r_ptr-i ? count : r_ptr-i;
	}

	return count;
}

```
### 测试 
```c
#include <stdio.h>

struct data_type {
	char* s;
	int expected_value;
};
enum {
	caseBASE = -1,
	case1,
	case2,
	case3,
	case4,
	case5,
	caseTOP
};
struct data_type test_data[] = {
	[case1] = {"abcabcbb", 3},
	[case2] = {"bbbbb", 1},
	[case3] = {"pwwkew", 3},
	[case4] = {"sjdgisgjisg", 5},
	[case5] = {"abcABC,.ab", 8},
};

int main(int argc, char* argv[])
{
	
	char* s = NULL;
	int expected_value = 0;
	int ret = 0;
	for (int i = caseBASE + 1; i < caseTOP; i++) {
		s = test_data[i].s;
		expected_value = test_data[i].expected_value;

		ret = lengthOfLongestSubstring(s);
		printf("testcase No.%d ", i);
		if (ret == expected_value) {
			printf("pass\n");
		} else {
			printf("=========================\n");
			printf("fail:ret=%d\n", ret);
			printf("expected_value:%d\n", expected_value);
			printf("=========================\n");
		}
	}
	return 0;
}
```
