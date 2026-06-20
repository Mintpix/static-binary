# ipmitool 1.8.19 静态构建步骤 (MSYS2 on Windows)

## 1. 下载源码 (Codeberg)
mkdir -p /tmp/ipmitool-build
cd /tmp/ipmitool-build
curl -L -o ipmitool-1.8.19.tar.gz https://codeberg.org/IPMITool/ipmitool/archive/IPMITOOL_1_8_19.tar.gz

## 2. 解压
tar xzf ipmitool-1.8.19.tar.gz
# 解压后目录名为 ipmitool (Codeberg tarball 格式)
cd ipmitool

## 3. 修补 configure.ac (从 tarball 构建，去掉 git 版本后缀)
sed -i 's|m4_define(\[git_suffix\], m4_esyscmd_s(./csv-revision))|m4_define([git_suffix], [])|' configure.ac

## 4. 安装构建依赖
pacman -S --noconfirm gcc make autoconf automake libtool pkg-config
pacman -S --noconfirm openssl openssl-devel libreadline libreadline-devel ncurses-devel

## 5. Bootstrap (生成 configure 脚本)
autoreconf -fi

## 6. 下载 IANA PEN registry 文件
curl -L -o /usr/share/misc/enterprise-numbers https://www.iana.org/assignments/enterprise-numbers.txt

## 7. Configure (启用 lanplus + 静态链接 + IANADIR)
./configure \
  --enable-static \
  --disable-shared \
  --disable-openipmi \
  LDFLAGS="-static" \
  LIBS="-lcrypto -lssl -lcrypt32 -lws2_32 -ladvapi32 -luser32 -lreadline -lncurses" \
  CFLAGS="-O2 -D_GNU_SOURCE" \
  IANADIR=/usr/share/misc

## 8. 编译
make -j12

## 9. 手动静态链接 (libtool 在 MSYS2/Cygwin 下无法生成真正静态二进制)
gcc -static -O2 -D_GNU_SOURCE -o src/ipmitool.exe \
  src/ipmitool.o src/ipmishell.o \
  lib/.libs/libipmitool.a \
  src/plugins/.libs/libintf.a \
  src/plugins/lan/.libs/libintf_lan.a \
  src/plugins/lanplus/.libs/libintf_lanplus.a \
  src/plugins/serial/.libs/libintf_serial.a \
  -lcrypto -lssl -lcrypt32 -lws2_32 -ladvapi32 -luser32 -lreadline -lncurses

gcc -static -O2 -D_GNU_SOURCE -o src/ipmievd.exe \
  src/ipmievd.o \
  lib/.libs/libipmitool.a \
  src/plugins/.libs/libintf.a \
  src/plugins/lan/.libs/libintf_lan.a \
  src/plugins/lanplus/.libs/libintf_lanplus.a \
  src/plugins/serial/.libs/libintf_serial.a \
  -lcrypto -lssl -lcrypt32 -lws2_32 -ladvapi32 -luser32

## 10. 验证静态链接 (只应依赖 Windows 系统 DLL)
ldd src/ipmitool.exe
ldd src/ipmievd.exe

## 11. 测试二进制
src/ipmitool.exe -V
src/ipmitool.exe -I list
src/ipmitool.exe -o list

# ============================================================
# 关键说明:
#   - _GNU_SOURCE: MSYS2/Cygwin 需要此宏才能使用 strptime()
#   - Windows 系统库: OpenSSL 静态链接需要 -lcrypt32 -lws2_32 -ladvapi32 -luser32
#   - readline: 静态链接需要显式指定 -lreadline -lncurses
#   - 手动 gcc -static: libtool 在 MSYS2 下默认动态链接，需手动链接
#   - IANADIR: 编译时指定 IANA PEN registry 文件路径，避免运行时下载
#   - 部署时需将 enterprise-numbers 文件放到目标机器对应路径，
#     或放在用户目录 ~/.ipmitool/ 下 (ipmitool 会优先查找此路径)
