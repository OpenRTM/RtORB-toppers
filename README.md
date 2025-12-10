# RtORB
Light-weight CORBA implementation with C-Language.

## LWIP組み込み
組み込みOS上動作することを目的にTCP/IPプロトコルスタックとしてLWIP組み込みを実施した。

### RtORBソース修正方針
malloc等のメモリ管理、gettimeofday等の関数以外に必要な、
mutex、socket通信用関数等はすべてLWIPで提供される関数を使用する構成とした。

定義値LWIPを用いて、LWIP固有の処理については「#ifdef LWIP #endif」節内で記述することとした。

実施内容
- インクルードするヘッダのLWIP対応
- 型宣言、関数定義のLWIP追加対応
- 標準ライブラリ定義とのコンフリクト対応
- メモリバッファサイズ等のLWIP対応
- 動的メモリ管理機構の組み込み
- ビルドスクリプトの変更
- メイクファイルの変更

#### インクルードするヘッダのLWIP対応
```
corba-defs.h:31
#ifdef LWIP
#else
#include <pthread.h>
#endif

corba-object-defs.h:48
#ifdef LWIP
#else
  pthread_t thread;
#endif

corba.h:37
#ifdef LWIP
#else
#include <pthread.h>
#endif


giop.h:41
#ifdef LWIP
#include <lwip/sockets.h>
#include <lwip/ip_addr.h>
#include <lwip/inet.h>
#else
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#endif

poa-defs.h:29
#ifdef LWIP
#include "lwip/sys.h"
#else
#include <pthread.h>
#endif

sockport.h:33
#ifdef LWIP
#include <lwip/sockets.h>
#include <lwip/netdb.h>
#else
#include <sys/socket.h>
#include <sys/un.h>
#include <netdb.h>
#include <netinet/in.h>
#endif

#include <signal.h>
#include <sys/time.h>
#ifdef LWIP
#else
#include <sys/ioctl.h>
#include <net/if.h>
#endif

giop.c:37
#if !defined(Cygwin) && !(LWIP) <---LWIPを追加
#include <ifaddrs.h>
#endif

util.c:28
#ifdef LWIP
#define _SELECT_H_
#include "lwip/sockets.h"
#endif

util.c:41
#ifdef LWIP
#else
#include <sys/ioctl.h>
#include <net/if.h>
#endif
```

#### 型宣言、関数定義のLWIP追加対応
```
cdrstream.h:206
#if (defined(Cygwin) && ( __GNUC__ < 4 )) ||(defined(LWIP))

giop.h:29
#ifdef LWIP
#if !defined(in_addr_t)
#include <stdint.h>+
typedef uint32_t in_addr_t;
#endif
#endif

pthread.c:33
#ifdef LWIP
#else
pthread_t *
RunThread(pthread_t *thr, void *(*func)(), void *arg, int detach){
    pthread_create(thr, NULL, func, arg);
    if(detach) pthread_detach(*thr);
    return thr;
}
#endif

socket.c:288
#else
#if defined(LWIP)
extern  struct netif *netif_list;
char *get_ip_address(int sock){
  struct netif *netif_ptr = netif_list;
  struct sockaddr_in addr;
  socklen_t socklen;
  ip_addr_t socket_ip_addr;

  //getsockname(sock,(struct sockaddr *)&addr,&socklen);
  //socket_ip_addr = *(ip_addr_t*)&addr.sin_addr;

  while (netif_ptr != NULL) {
      //if(ip4_addr_cmp(&netif_ptr->ip_addr,&socket_ip_addr))
      if(netif_ptr->ip_addr.addr!=0)
      {
        char *ip_addr =  ipaddr_ntoa(&netif_ptr->ip_addr);
        if(  1 //(addr.sin_family == AF_INET)
        #if 1
              && (strcmp(ip_addr,"127.0.0.1") != 0)
              && (strncmp(ip_addr,"169.254.", 8) != 0)
              && (strcmp(ip_addr,"0.0.0.0") != 0)
        #endif
        )
        {
            return RtORB_strdup(ip_addr, "get_ip_address");
        }  
      }
       netif_ptr = netif_ptr->next;
  }
  return (char *)NULL;
}

util.c:177
else
#if defined(LWIP)

char *new_ObjectID(){
  int fd;
  struct ifreq *ifr;
  struct timeval tv;
#ifndef Linux
  struct timezone tz;
#endif
  struct netif *netif_ptr = netif_list;

  char *ID = (char *)RtORB_alloc(33, "new_ObjectID"); 
  memset(ID, 0, 33);

  while (netif_ptr != NULL) {
    if(netif_ptr->ip_addr.addr!=0)
    {
      /* IP4 有効*/
      break;
    }
    netif_ptr = netif_ptr->next;
  }


  gettimeofday(&tv, NULL);

  sprintf(ID, "RtORB%.2X%.2X%.2X%.2X%.2X%.2X",
	(unsigned char)netif_ptr->hwaddr[0],
   	(unsigned char)netif_ptr->hwaddr[1],
   	(unsigned char)netif_ptr->hwaddr[2],
   	(unsigned char)netif_ptr->hwaddr[3],
   	(unsigned char)netif_ptr->hwaddr[4],
  	(unsigned char)netif_ptr->hwaddr[5]);
  sprintf(ID+16, "%010X", (int)tv.tv_sec);
  sprintf(ID+26, "%06X", (int)tv.tv_usec);

  return ID;
}
```

#### 標準ライブラリ定義とのコンフリクト対応
標準のselect.hの定義とコンフリクトするため、定義値を定義して標準のselect.h読み込みを抑制する。今回はLWIPが提供しているFD_SETSIZEと違う値が定義されてしまい、ファイルディスクリプタの扱いに支障があるため読み込みを抑制。
```
#ifdef LWIP
#define _SELECT_H_
#include "lwip/sockets.h"
#endif
```
#### メモリバッファサイズ等のLWIP対応
GIOPの受信バッファサイズ、返答バッファサイズを2Mbytesから4kbytesに変更。
```
giop.h:57
#ifdef LWIP
#define GIOP_SIZEMAX 1024*4
#else
#define GIOP_SIZEMAX 1024*2048
#endif

giop.c:42
#define RECV_BUF_SIZE  GIOP_SIZEMAX

giop.c:907
  buf = ( char* )RtORB_alloc( RECV_BUF_SIZE, "GIOP_enqueue_request");

giop.c:910
  if(receiveMessage(h, &header, (octet *)buf, RECV_BUF_SIZE) < 0){

giop.c:943
  int recvBufSize = RECV_BUF_SIZE;

giop-marshal.c:1376
int MaxSize = GIOP_SIZEMAX;
```
#### 動的メモリ管理機構の組み込み
動的メモリ管理機構としてtlsfを組み込んで使用する。tlsf自体はtoppers側で提供する。
```
corba.h:33
#ifdef USE_TLSF
#ifdef __cplusplus
extern "C"{
#endif
#include "tlsf.h"
char* strdup(const char* s);
char* strndup(const char* s,size_t n);
#ifdef __cplusplus
}
#endif
#endif 

corba.h:129
#ifdef USE_TLSF
#  define RtORB_alloc(s, info)		tlsf_malloc(s)
#  define RtORB_realloc(p, s, info)	tlsf_realloc(p, s)
#  define RtORB_calloc(s, n, info)	tlsf_calloc(n, s)
#  define RtORB_strdup(s, info)		strdup(s)
#  define RtORB_strndup(s, n, info)	strndup(s, n)
#  define RtORB_free(s, info)		tlsf_free(s)
#else

corba.hh:33
#ifdef USE_TLSF
#include "tlsf.h"
char* strdup(const char* s);
char* strndup(const char* s,size_t n);
#endif

```
#### ビルドスクリプトの変更
```
gen_config:6
NO_UUIDLIB=0  <--- 1->0

```
#### メイクファイルの変更
Makefile.lwipを作成して、Linux用のメイクファイルとは完全に分離した。
##### ツールチェインを変更
```
CC = /usr/bin/arm-none-eabi-gcc
CXX = /usr/bin/arm-none-eabi-g++
AR = /usr/bin/arm-none-eabi-ar
```
##### インクルードパスの追加
```
INCLUDE = -I. -I../include -I./CosName -I../include/RtORB/functions/
INCLUDE += -I../../asp3_3.7/appli/ -I../../asp3_3.7/bsp/include/
INCLUDE += -I../../lwip/src/include/ -I../../lwip/src/include/lwip  -I../../lwip/contrib/ports/toppers/include/
```
##### CFLAGSの追加
```
CFLAGS += -DLWIP -DUSE_TLSF
CFLAGS += -mcpu=cortex-a9  -DUSE_ARM_FPU_ALWAYS -mfpu=vfpv3-d16 -mfloat-abi=hard -mlittle-endian
```
##### LDFLAGSの追加
```
LDFLAGS += -mcpu=cortex-a9  -DUSE_ARM_FPU_ALWAYS -mfpu=vfpv3-d16 -mfloat-abi=hard -mlittle-endian
```

#####  静的ライブラリの生成ルールの追加及び<br>シェアドライブラリの生成抑制

```
all: 
	$(MAKE) CosNaming
	$(MAKE) -f Makefile.lwip libCXX
	$(MAKE) -f Makefile.lwip $(LIBORB_A)


$(LIBORB_A): $(LIBOBJS) $(NAMING_HEADER) $(NAMING_SRCS)
	$(AR) rcs $@ $(LIBOBJS)
	$(RANLIB)  $@

```

