# freetype

��Դ��������⣬��ʵ��ʸ��������ʾ�����ṩ�����ļ�����
���裺ȡ���ؼ��㡢ʵ�ֱպ����ߡ������ɫ�����ؼ��㣨glyph�������������ļ��У�


����һ���ַ�����ô�������ļ����ҵ����Ĺؼ��㣿
����Ҫȷ�����ַ��ı���ֵ������ ASCII�롢GB2312�롢UNICODE�롣��������ļ�֧��ĳ�ֱ����ʽ(charset)���Ϳ���ʹ���������ֵȥ�ҵ����ַ��Ĺؼ���(glyph)����Щ�����ļ�֧�ֶ��ֱ����ʽ(charset)�������ļ��б���Ϊcharmaps(ע�⣺��������Ǹ�������ζ�ſ���֧�ֶ���charset)

�� simsun.ttc Ϊ�����������ļ��ĸ����£�ͷ������ charmaps������ʹ��ĳ�ֱ���ֵȥ charmaps ���ҵ�����Ӧ�Ĺؼ��㡣��ͼ�еġ�A��B���С�����Τ����ֻ�� glyph ��ʾ��ͼ����ʾ�ؼ��㡣

Charmaps ��ʾ�ַ�ӳ��������ļ�����֧����һЩ���룬GB2312��UNICODE��BIG5 ����������������ļ�֧�ָñ��룬ʹ�ñ���ֵͨ�� charmap �Ϳ����ҵ���Ӧ�� glyph��һ����Զ�֧�� UNICODE �롣


���ϣ�һ�����ֵ���ʾ���̿��Ը���Ϊ���²��裺
1. ����һ���ַ�����ȷ�����ı���ֵ(ASCII��UNICODE��GB2312)��
2. ���������С��
3. ���ݱ���ֵ�����ļ�ͷ����ͨ�� charmap �ҵ���Ӧ�Ĺؼ���(glyph)��Ȼ����������С�����ؼ��㣻
4. �ѹؼ���ת��Ϊλͼ����
5. ��LCD����ʾ

freetype�ٷ���ַ��https://www.freetype.org/

ʹ��freetype�ⲽ�裺
1. ��ʼ����FT_InitFreetpye
2. ���أ��򿪣����壺FT_New_Face
3. ���������С�� FT_Set_Char_Sizes �� FT_Set_Pixel_Sizes
4. ѡ�� charmap: FT_Select_Charmap
5. ���ݱ���ֵcharcode�ҵ�glyph_index: glyph_index = FT_Get_Char_Index(face, charcode)
6. ����glyph_indexȡ��glyph: FT_Load_Glyph(face, glyph_index)
7. תΪλͼ��FT_Render_Glyph
8. �ƶ�����ת��FT_Set_Transform
9. ��ʾ

5��6��7��ʹ�ú���FT_Load_Char��face, charcode, FT_LOAD_RENDER�����棬�Ϳɵõ�λͼ��

## ��LCD����ʾһ��ʸ������
**ʹ��wchar_t����ַ���UNICODEֵ**
Ҫ��ʾһ���ַ�������Ҫȷ�����ı���ֵ�����õ��� UNICODE ���룬�ڳ�����ʹ��`char *str = "��"`��������䶨���ַ���ʱ��str �б����Ҫô�� GB2312 ����ֵ��Ҫô��UTF-8 ��ʽ�ı���ֵ����ʹ����ʱʹ�á�-fexec-charset=UTF-8����str �б����Ҳ����ֱ�����õ� UNICODE ֵ��
������ڴ�������ֱ��ʹ�� UNICODE ֵ����Ҫʹ�� wchar_t�����ַ���ʾ���������£�
```C
#include <stdio.h>
#include <string.h>
#include <wchar.h>

int main(int argc, char **argv)
{
    wchar_t *chinese_str = L"��gif";
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
gcc -o test_wchar test_wchar.c��
./test_wchar
sizeof(wchar_t) = 4, str's Unicode:
0x4e2d 0x67 0x69 0x66
```
ÿ�� wchar_t ռ�� 4 �ֽڣ���ִ�г����� wchar_t �б���ľ����ַ��� UNICODEֵ��
ע�����test_wchar.c���� ANSI(GB2312)��ʽ���棬��ô��Ҫʹ���������������룺
```shell
gcc -finput-charset=GB2312 -fexec-charset=UTF-8 -o test_wchar test_wchar.c
```