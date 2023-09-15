# freetype

开源字体引擎库，可实现矢量字体显示（需提供字体文件）。
步骤：取出关键点、实现闭合曲线、填充颜色。（关键点（glyph）存在于字体文件中）


给定一个字符，怎么在字体文件中找到它的关键点？
首先要确定该字符的编码值：比如 ASCII码、GB2312码、UNICODE码。如果字体文件支持某种编码格式(charset)，就可以使用这类编码值去找到该字符的关键点(glyph)。有些字体文件支持多种编码格式(charset)，这在文件中被称为charmaps(注意：这个单词是复数，意味着可能支持多种charset)

以 simsun.ttc 为例，该字体文件的格如下：头部含有 charmaps，可以使用某种编码值去 charmaps 中找到它对应的关键点。下图中的“A、B、中、国、韦”等只是 glyph 的示意图，表示关键点。

Charmaps 表示字符映射表，字体文件可能支持哪一些编码，GB2312、UNICODE、BIG5 或其他。如果字体文件支持该编码，使用编码值通过 charmap 就可以找到对应的 glyph，一般而言都支持 UNICODE 码。


由上，一个文字的显示过程可以概括为如下步骤：
1. 给定一个字符可以确定它的编码值(ASCII、UNICODE、GB2312)；
2. 设置字体大小；
3. 根据编码值，从文件头部中通过 charmap 找到对应的关键点(glyph)，然后根据字体大小调整关键点；
4. 把关键点转换为位图点阵；
5. 在LCD上显示

freetype官方网址：https://www.freetype.org/

使用freetype库步骤：
1. 初始化：FT_InitFreetpye
2. 加载（打开）字体：FT_New_Face
3. 设置字体大小： FT_Set_Char_Sizes 或 FT_Set_Pixel_Sizes
4. 选择 charmap: FT_Select_Charmap
5. 根据编码值charcode找到glyph_index: glyph_index = FT_Get_Char_Index(face, charcode)
6. 根据glyph_index取出glyph: FT_Load_Glyph(face, glyph_index)
7. 转为位图：FT_Render_Glyph
8. 移动或旋转：FT_Set_Transform
9. 显示

5、6、7可使用函数FT_Load_Char（face, charcode, FT_LOAD_RENDER）代替，就可得到位图。

## 在LCD上显示一个矢量字体
**使用wchar_t获得字符的UNICODE值**
要显示一个字符，首先要确定它的编码值。常用的是 UNICODE 编码，在程序里使用`char *str = "中"`这样的语句定义字符串时，str 中保存的要么是 GB2312 编码值，要么是UTF-8 格式的编码值，即使编译时使用“-fexec-charset=UTF-8”，str 中保存的也不是直接能用的 UNICODE 值。
如果想在代码中能直接使用 UNICODE 值，需要使用 wchar_t，宽字符，示例代码如下：
```C
#include <stdio.h>
#include <string.h>
#include <wchar.h>

int main(int argc, char **argv)
{
    wchar_t *chinese_str = L"中gif";
    unsigned int *p = (wchar_t *)chinese_str;

    printf("sizeof(wchar_t) = %d, str's Unicode: \n", (int)sizeof(wchar_t));
    
    int i;
    for(i = 0; i < wcslen(chinese_str); i ++)
    {
        printf("0x%x", p[i]);
    }
    printf("\n");

    return 0;
}
```

```shell
gcc -o test_wchar test_wchar.c后
./test_wchar
sizeof(wchar_t) = 4, str's Unicode:
0x4e2d 0x67 0x69 0x66
```
每个 wchar_t 占据 4 字节，可执行程序里 wchar_t 中保存的就是字符的 UNICODE值。
注：如果test_wchar.c是以 ANSI(GB2312)格式保存，那么需要使用以下命令来编译：
```shell
gcc -finput-charset=GB2312 -fexec-charset=UTF-8 -o test_wchar test_wchar.c
```