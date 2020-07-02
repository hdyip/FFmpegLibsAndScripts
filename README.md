FFmpeg是非常强大的编解码库，支持相当多的格式。如果你仔细看一下手机中各种播放器的许可条款，一定不会缺少FFmpeg的身影。
但是网络上FFmpeg移植的资料都非常少，很多还是使用很老的库，比如使用2.xx版本的FFmpeg，NDK版本还使用r9（2019年最新的NDK版本已经到了r21）。但是程序员一定是要与时俱进的嘛，因此我花了挺长时间研究了一下新版本移植的问题，最终终于成功了。

编译完成的库请看我的github项目：FFmpegLibsAndScripts。其中包含了已经完成编译的库和编译的脚本。注意的是iOS版本的库还未实验过。Android已经经过实验并无问题，但仅编译了armv7和arm64的库，如果你要改动脚本为x86编译，可能需要注意asm相关的设置。记住如果需要自己编译，一定要修改脚本中的NDK_HOME变量为你自己的NDK目录。

FFmpeg虽然包含了绝大多数格式的编解码库，但是某些有专利的库并没有包含进去，需要我们手动去链接它。为了实现基本的播放功能，主要需要包含3个库：FDK-AAC（用以解码aac格式是比较先进的压缩格式，如果你仔细看一下大多数的视频格式信息，会发现很多视频的音频部分是以aac格式编码的）；mp3lame（用以编解码mp3格式的音频）；x264（非常流行的视频编码格式，基本90%的视频目前都是以x264编码的）。首要任务就是编译这三个库。

对于我的编译脚本，我都是放在工程根目录中的。因此运行时请注意位置。输出目录也在根目录的/thin目录中。对于FFmpeg编译时链接的libfdk-aac, libmp3lame以及libx264，都存放在FFmpeg根目录的external-lib目录中，具体如下：

在这里插入图片描述

有几个点需要注意。

首先是x264编译出的so库的名称是以libx264.so.1234，其中1234是版本号。显然对于这样的名称，Android是无法识别的，因此我们需要修改x264的configure脚本，其中有这么一段是用来在产生不同平台下库的名称：

if [ "$shared" = "yes" ]; then
    API=$(grep '#define X264_BUILD' < ${SRCPATH}/x264.h | cut -f 3 -d ' ')
    if [ "$SYS" = "WINDOWS" -o "$SYS" = "CYGWIN" ]; then
        echo "SONAME=libx264-$API.dll" >> config.mak
        if [ $compiler_style = MS ]; then
            echo 'IMPLIBNAME=libx264.dll.lib' >> config.mak
            echo "SOFLAGS=-dll -implib:\$(IMPLIBNAME) $SOFLAGS" >> config.mak
        else
            echo 'IMPLIBNAME=libx264.dll.a' >> config.mak
            echo "SOFLAGS=-shared -Wl,--out-implib,\$(IMPLIBNAME) $SOFLAGS" >> config.mak
        fi
    elif [ "$SYS" = "MACOSX" ]; then
        echo "SOSUFFIX=dylib" >> config.mak
        echo "SONAME=libx264.$API.dylib" >> config.mak
        echo "SOFLAGS=-shared -dynamiclib -Wl,-single_module -Wl,-read_only_relocs,suppress -install_name \$(DESTDIR)\$(libdir)/\$(SONAME) $SOFLAGS" >> config.mak
    elif [ "$SYS" = "SunOS" ]; then
        echo "SOSUFFIX=so" >> config.mak
        echo "SONAME=libx264.so.$API" >> config.mak
        echo "SOFLAGS=-shared -Wl,-h,\$(SONAME) $SOFLAGS" >> config.mak
    else
        echo "SOSUFFIX=so" >> config.mak
        echo "SONAME=libx264.x.so" >> config.mak
        echo "SOFLAGS=-shared -Wl,-soname,\$(SONAME) $SOFLAGS" >> config.mak
    fi
    echo 'default: lib-shared' >> config.mak
    echo 'install: install-lib-shared' >> config.mak
fi

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27

最后一个else情况中就是对应我们的情况，我改成了libx264.x.so，不知为何如果直接使用libx264.so会编译失败，所以就改成了这样。这样一来，最后库的名字就是libx264.x.so，同时会生成一个名字为libx264.so的链接。但是即使如此，我们也不能直接使用这个so库去和FFmpeg进行链接，因为在FFmpeg中库的名字是已经写死的，当你设置–enable-x264之后，它只会去找libx264.so。因此我们可以将这个库复制到FFmpeg要链接的目录下之后，再在这个目录中新建一个指向这个库的软链接，只要让这个链接的名字是libx264.so即可。（软链接创建命令：ln -s 源文件 链接）。

另外一个比较坑的地方是fdk-aac的头文件位置。FFmpeg中fdk-aac头文件的包含是这样写的：

#include "fdk-aac/xxx.h"

    1

因此，你在写包含参数（也就是传给编译器的-I参数，表示包含头文件的位置），一定不要直接指向最底层的文件夹，比如你的路径是/a/fdk-aac/xxx.h，你可以写-I/a，但不能写-I/a/fdk-aac，这样FFmpeg是找不到的。

最后写一下关于怎样对编译过程进行debug，一个是多看log文件，对于FFmpeg来说，编译log位置就在ffbuild/config.log，其他的库在出问题时，会提示你去看log并标明log位置。另外一个就是我们主要目标在于configure文件，只要configure没问题，那 编译基本就没问题了。因此在debug阶段，先不要写make & make install，注意看configure的输出，一旦有一个错误，那基本就是跑不过的。configure会产生makefile文件，那是真正用来编译的。如果你上次跑过了，比如为电脑平台编译了FFmpeg，然后又为Android平台编译，如果Android平台configure没有通过，但你仍然还是可以make，因为configure没有成功的话是不会修改makefile文件的，所以你这次make的还是上次电脑平台的目标文件。这点比较迷惑，千万注意。
————————————————
版权声明：本文为CSDN博主「zuguorui」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zuguorui/article/details/104150008
