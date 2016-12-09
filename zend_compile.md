# PHP代码的编译

PHP是解析型语言，前面已经说过PHP只是一个普通的C程序，PHP代码经过语法解析转化为一个个opcode，代码里的所有信息都保存在opcode中，然后将opcode数组交给zend引擎执行，这就是PHP代码的编译过程。

编译过程包括词法分析、语法分析，旧的PHP版本直接生成了opcode，PHP7新增了抽象语法树（AST），在语法分析阶段生成AST，然后再生成opcodes。

PHP使用re2c、bison完成编译，关于这两个的使用可以单独查下相关的资料。

re2c的示例:(http://re2c.org/examples/examples.html)
```c
#include <stdio.h>

static const char *lex(const char *YYCURSOR)
{
    const char *YYMARKER;
    /*!re2c
        re2c:define:YYCTYPE = char;
        re2c:yyfill:enable = 0;

        end = "\x00";
        bin = '0b' [01]+;
        oct = "0" [0-7]*;
        dec = [1-9][0-9]*;
        hex = '0x' [0-9a-fA-F]+;

        *       { return "err"; }
        bin end { return "bin"; }
        oct end { return "oct"; }
        dec end { return "dec"; }
        hex end { return "hex"; }
    */
}

int main(int argc, char **argv)
{
    for (int i = 1; i < argc; ++i) {
        printf ("%s: %s\n", lex(argv[i]), argv[i]);
    }
    return 0;
}
```
```
$ re2c -o example.cc 01_recognizing_integers.re
$ g++ -o example example.cc
$ ./example 0 12345678901234567890 0xAbcDEf 0x00 007 0B0 0b110101010 0x 0b ? ""
oct: 0
dec: 12345678901234567890
hex: 0xAbcDEf
hex: 0x00
oct: 007
bin: 0B0
bin: 0b110101010
err: 0x
err: 0b
err: ?
err:
```
