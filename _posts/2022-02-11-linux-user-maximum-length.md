---
layout: post
title: Linux ユーザ名の最大長
tags: "linux"
comments: true
---

Linux システムにおけるユーザ名の最大長っていくつになるんだろう、というのを調べました。
通常 POSIX 仕様やユーザ追加に使用するコマンド (`useradd`) が関係するようです。

### POSIX 仕様

POSIX 仕様では `LOGIN_NAME_MAX` という定数がログインユーザ名の最大長として定義されています。

> LOGIN\_NAME\_MAX - _SC\_LOGIN\_NAME\_MAX
>   Maximum length of a login name, including the terminating
>   null byte. Must not be less than \_POSIX\_LOGIN\_NAME\_MAX (9).
>
> (man 3 sysconf より)

ある環境における `LOGIN_NAME_MAX` の値は以下のようなプログラムにより確認できます。
例えば Ubuntu 20.04 LTS (Focal Fossa) では 256 でした。

```c
#include <limits.h>
#include <stdio.h>
#include <unistd.h>

void main(void) {
  printf("%d\n", LOGIN_NAME_MAX);
}
```

### useradd

ユーザ作成に使用される `useradd` コマンド内で別の制限が加えられていることがあります。

多くの Linux システムでは `useradd` は [shadow-utils][1] から来ていると思いますが、その man ページでは作成できるユーザ名の最大長が 32 かもしれないと記述されています。

> Usernames may only be up to 32 characters long.
>
> (shadow-utils 4.8.1 より)

shadow-utils では `shadow/lib/defines.h` の `USER_NAME_MAX_LENGTH` が許容されるユーザ名の最大長として定義されています。

```c
/* Maximum length of usernames */
#ifdef HAVE_UTMPX_H
# include <utmpx.h>
# define USER_NAME_MAX_LENGTH (sizeof (((struct utmpx *)NULL)->ut_user))
#else
# include <utmp.h>
# ifdef HAVE_STRUCT_UTMP_UT_USER
#  define USER_NAME_MAX_LENGTH (sizeof (((struct utmp *)NULL)->ut_user))
# else
#  ifdef HAVE_STRUCT_UTMP_UT_NAME
#   define USER_NAME_MAX_LENGTH (sizeof (((struct utmp *)NULL)->ut_name))
#  else
#   define USER_NAME_MAX_LENGTH 32
#  endif
# endif
#endif
```

utmp は

> utmp, wtmp, btmp and variants such as utmpx, wtmpx and btmpx are files on Unix-like systems that keep track of all logins and logouts to the system.
>
> (from utmp - Wikipedia)

であり、`w` や `last` コマンドの元になる情報です。特に utmpx は POSIX 仕様に含まれます。

`man 5 utmpx` によると仕様では `ut_user` のサイズは規定されないとのことですが、多くの Linux システムでは `utmpx` 構造体の `ut_user` のサイズは 32 bytes になっているようです。これが `useradd` コマンドで作成できるユーザ名の最大長になります。

```c
// Ubuntu 20.04 LTS 上 `/usr/include/x86_64-linux-gnu/bits/utmpx.h` からの抜粋:

...
#define __UT_NAMESIZE 32
...
struct utmpx
{
   ...
   char ut_user[__UT_NAMESIZE]
     __attribute_nonstring__;    /* Username.  */
   ...
}
...
```

ちなみに Alpine Linux では BusyBox 由来の `useradd` が使われるので上述とは異なる挙動になります。`die_if_bad_username` という関数でユーザ名の検証が行われ、そのとき最大長として前述の `LOGIN_NAME_MAX` を使用するので例えば 255 文字 (`LOGIN_NAME_MAX` は 末端 null byte も含むので) までのユーザが作成できます。

```c
/* To avoid problems, the username should consist only of
 * letters, digits, underscores, periods, at signs and dashes,
 * and not start with a dash (as defined by IEEE Std 1003.1-2001).
 * For compatibility with Samba machine accounts $ is also supported
 * at the end of the username.
 */

void FAST_FUNC die_if_bad_username(const char *name)
{
...
	/* The minimum size of the login name is one char or two if
	 * last char is the '$'. Violations of this are caught above.
	 * The maximum size of the login name is LOGIN_NAME_MAX
	 * including the terminating null byte.
	 */
	if (name - start >= LOGIN_NAME_MAX)
		bb_error_msg_and_die("name is too long");
}
```

### 長い名前のユーザを作成

ここまでの内容から、

- システムとしては `LOGIN_NAME_MAX` までの長さを持つユーザ名は許容されそう
- utmp, wtmp あたりを使用するコマンドだと 33 文字以上で問題になるのでは?

という推測がたったので、実際に長い名前を持つユーザを作成してみました。
この項では Ubuntu 20.04 LTS 環境を使用した結果を載せています。

必要なだけ長い名前を持つユーザが作成できるよう、shadow-utils を改変してビルドします。

まず `USER_NAME_MAX_LENGTH` を 255 で定義し直します。

```c
// in shadow/lib/defines.h

#define USER_NAME_MAX_LENGTH 255
```

次に `shadow/.builds/ubuntu-focal.yml` を参考にビルドを行います。

```
$ sudo apt-get install automake autopoint xsltproc libselinux1-dev gettext expect byacc libtool
$ git clone https://github.com/shadow-maint/shadow.git
$ cd shadow/
$ ./autogen.sh --without-selinux --disable-man
$ make
```

修正した `useradd` で適当に長い名前を持つユーザを作成してみます。
(ここでは 33 文字のユーザ名を利用。255 文字でも同様ですが記載上は見辛すぎるので)

```
# 33 文字の名前を持つユーザ
$ sudo ./src/useradd -m -s /bin/bash aaaaaaaaaabbbbbbbbbbccccccccccddd
```

無事作成できました。実際にログインして簡単に動かしてみても問題なさそうです。

```
$ ssh aaaaaaaaaabbbbbbbbbbccccccccccddd@192.168.40.10
aaaaaaaaaabbbbbbbbbbccccccccccddd@192.168.40.10's password:

Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-92-generic x86_64)
...

$ pwd
/home/aaaaaaaaaabbbbbbbbbbccccccccccddd

$ cat /etc/passwd | grep aaa
aaaaaaaaaabbbbbbbbbbccccccccccddd:x:1002:1002::/home/aaaaaaaaaabbbbbbbbbbccccccccccddd:/bin/bash
```

utmp, wtmp 関連のコマンドは実行に失敗したりはしませんが、`w` コマンドでユーザ数にはカウントされているのに情報が出力されないといった不思議な挙動が見られます。

```
$ last
aaaaaaaa pts/1        192.168.40.10    Fri Feb 11 07:09    gone - no logout
aaaaaaaa pts/1        192.168.40.10    Fri Feb 11 06:53 - 07:05  (00:11)
vagrant  pts/0        10.0.2.2         Fri Feb 11 06:35   still logged in
vagrant  pts/0        10.0.2.2         Fri Feb 11 02:51 - 04:01  (01:10)
reboot   system boot  5.4.0-92-generic Fri Feb 11 02:50   still running
vagrant  pts/0        10.0.2.2         Tue Feb  8 01:43 - 03:40  (01:56)
reboot   system boot  5.4.0-92-generic Tue Feb  8 01:42 - 03:40  (01:58)
vagrant  pts/1        10.0.2.2         Fri Dec 24 01:18 - 01:18  (00:00)
vagrant  pts/0        10.0.2.2         Fri Dec 24 01:12 - 01:58 (1+00:45)
reboot   system boot  5.4.0-77-generic Fri Dec 24 01:11 - 03:40 (46+02:28)
vagrant  pts/0        10.0.2.2         Fri Dec 24 01:06 - 01:06  (00:00)
reboot   system boot  5.4.0-77-generic Fri Dec 24 01:01 - 01:11  (00:09)

wtmp begins Fri Dec 24 01:01:42 2021

$ w
 07:12:10 up  2:30,  2 users,  load average: 0.01, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
vagrant  pts/0    10.0.2.2         06:35    2.00s  0.41s  0.02s ssh aaaaaaaaaabbbbbbbbbbccccccccccddd@192.168.40.10
```

[1]: https://github.com/shadow-maint/shadow/
