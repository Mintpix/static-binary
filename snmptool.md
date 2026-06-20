# net-snmp 5.9.5.2 静态构建步骤 (MSYS2 on Windows)

## 1. 下载源码 (GitHub)
mkdir -p /tmp/net-snmp-build
cd /tmp/net-snmp-build
curl -L -o net-snmp-5.9.5.2.tar.gz https://github.com/net-snmp/net-snmp/archive/refs/tags/v5.9.5.2.tar.gz

## 2. 解压
tar xzf net-snmp-5.9.5.2.tar.gz
cd net-snmp-5.9.5.2

## 3. 安装构建依赖
pacman -S --noconfirm gcc make autoconf automake libtool pkg-config
pacman -S --noconfirm openssl openssl-devel

## 4. Configure (静态链接 + OpenSSL + 禁用 agent + IPv6 + AES-256)
# 注意: 不使用 LDFLAGS="-static"，否则 configure 的 OpenSSL 检测会失败
# 注意: --disable-agent 避免 Cygwin 下 Windows 头文件与 POSIX 头文件冲突 (IN6_ADDR)
# 注意: --enable-blumenthal-aes 启用 AES-192/AES-256 加密 (SNMPv3 draft 标准)
./configure \
  --disable-shared \
  --enable-static \
  --with-openssl=/usr \
  --disable-scripts \
  --disable-manuals \
  --disable-mibs \
  --disable-agent \
  --enable-blumenthal-aes \
  --with-default-snmp-version="3" \
  --with-sys-contact="root@localhost" \
  --with-sys-location="unknown" \
  --with-logfile="/var/log/snmpd.log" \
  --with-persistent-directory="/var/net-snmp" \
  CFLAGS="-O2 -D_GNU_SOURCE" \
  LIBS="-lcrypto -lssl -lcrypt32 -ladvapi32 -luser32"

## 5. 修补 config: 禁用 HAVE_CLOSESOCKET (Cygwin 中应使用 close() 而非 closesocket())
sed -i 's/^#define HAVE_CLOSESOCKET 1$/\/\* #undef HAVE_CLOSESOCKET \*\//' include/net-snmp/net-snmp-config.h

## 6. 编译
make -j12

## 7. 手动静态链接 (libtool 在 MSYS2/Cygwin 下无法生成真正静态二进制)
# --- SNMP 客户端工具 (只依赖 libnetsnmp) ---
for tool in snmpget snmpwalk snmpbulkwalk snmpbulkget snmpset snmpgetnext \
            snmptranslate snmpstatus snmptable snmptrap snmpdf snmpdelta \
            snmpusm snmpvacm encode_keychange snmptest snmpping; do
  gcc -static -O2 -D_GNU_SOURCE -o apps/$tool.exe apps/$tool.o \
    snmplib/.libs/libnetsnmp.a \
    -lcrypto -lssl -lcrypt32 -ladvapi32 -luser32 -liphlpapi -lsnmpapi
done

# --- snmpnetstat (多个 .o 文件) ---
gcc -static -O2 -D_GNU_SOURCE -o apps/snmpnetstat/snmpnetstat.exe \
  apps/snmpnetstat/main.o apps/snmpnetstat/if.o apps/snmpnetstat/inet.o \
  apps/snmpnetstat/inet6.o apps/snmpnetstat/route.o apps/snmpnetstat/routex.o \
  apps/snmpnetstat/inetx.o apps/snmpnetstat/ffs.o apps/snmpnetstat/winstub.o \
  snmplib/.libs/libnetsnmp.a \
  -lcrypto -lssl -lcrypt32 -ladvapi32 -luser32 -liphlpapi -lsnmpapi

# --- snmptrapd (需要 agent 库) ---
gcc -static -O2 -D_GNU_SOURCE -o apps/snmptrapd.exe \
  apps/snmptrapd.o apps/snmptrapd_log.o apps/snmptrapd_auth.o \
  apps/snmptrapd_handlers.o apps/snmptrapd_sql.o \
  agent/.libs/libnetsnmpmibs.a agent/.libs/libnetsnmpagent.a \
  agent/helpers/.libs/libnetsnmphelpers.a snmplib/.libs/libnetsnmp.a \
  -lcrypto -lssl -lcrypt32 -ladvapi32 -luser32 -liphlpapi -lsnmpapi

# --- snmpps 无法静态链接 (需要 ncurses，MSYS2 ncurses 无静态库) ---
# 跳过 snmpps

## 8. 验证静态链接 (只应依赖 Windows 系统 DLL + msys-2.0.dll)
ldd apps/snmpget.exe
ldd apps/snmpwalk.exe
ldd apps/snmpnetstat/snmpnetstat.exe

## 9. 测试二进制
apps/snmpget.exe -V
apps/snmpwalk.exe -V
apps/snmptranslate.exe -V

# ============================================================
# 关键说明:
#   - _GNU_SOURCE: MSYS2/Cygwin 需要此宏
#   - HAVE_CLOSESOCKET: Cygwin 中应禁用，使用 close() 关闭 socket
#   - --disable-agent: Cygwin 下 agent 的 MIB-II 模块使用了 Windows iphlpapi.h，
#     与 POSIX 网络头文件冲突 (IN6_ADDR 重定义)，禁用 agent 可避免此问题
#   - 不使用 LDFLAGS="-static": configure 的 OpenSSL 链接测试在 -static 下会失败
#     (缺少 Windows 系统库)，改为编译后手动 gcc -static 链接
#   - Windows 系统库: OpenSSL 静态链接需要 -lcrypt32 -ladvapi32 -luser32
#   - 不链接 -lws2_32: MSYS 环境下 socket/getaddrinfo 由 msys-2.0.dll (Cygwin POSIX 层) 提供，
#     链接 Winsock 的 ws2_32 会造成 getaddrinfo 符号冲突，导致地址解析失败
#   - -liphlpapi -lsnmpapi: net-snmp 在 Cygwin 下需要这两个 Windows 网络库 (不提供 socket 函数，不冲突)
#   - snmpps: 无法静态链接，因为 MSYS2 的 ncurses 只提供动态库
#   - --enable-blumenthal-aes: 启用 AES-192/AES-256 加密支持 (基于 draft-blumenthal-aes-usm-04)
#     默认关闭，因为 AES-256 在 SNMPv3 中还不是正式标准。启用后 -x 选项可接受 AES256/AES256C
#   - 手动 gcc -static: libtool 在 MSYS2 下默认动态链接，需手动链接
