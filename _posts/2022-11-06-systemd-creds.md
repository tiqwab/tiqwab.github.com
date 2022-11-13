---
layout: post
title: systemd-creds を使ったクレデンシャル管理
tags: "systemd"
comments: true
---

先日 Software Design 2022-11 月号で systemd-creds というツールの存在を知り興味を持ったので少し触ってみました。

systemd-creds を利用したクレデンシャル管理は systemd v250 から利用可能です。v250 は 2021-12 にリリースされたもので現時点ではまだ含まれていない Linux ディストリビューションも多そうですが、例えば Ubuntu だと 22.10 であれば v251 が入っているようです。

以下は Arch Linux で systemd v251 で試した実行結果を載せています。

```bash
$ systemctl --version
systemd 251 (251.6-2-arch)
+PAM +AUDIT -SELINUX -APPARMOR -IMA +SMACK +SECCOMP +GCRYPT +GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY +P11KIT -QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -BPF_FRAMEWORK +XKBCOMMON +UTMP -SYSVINIT default-hierarchy=unified
```

### systemd サービスにおけるクレデンシャルの扱い方

過去 systemd の仕組みの範疇でサービスにクレデンシャルを渡すためには、ユニットファイルに `Environment` ディレクティブで直書きする、外部ファイルで (適切なアクセス制御付きで) 管理して `EnvironmentFile` ディレクティブで指定する、といった方法が考えられました。しかし前者は `systemctl show` で簡単にクレデンシャルが漏洩します。後者は StackOverflow 等でよく見る解決策ですが、systemd では以下のような理由でそもそも環境変数によるクレデンシャル管理を推奨していません (`man 5 systemd.exec` の Environment ディレクティブより)。

- サービスに定義された環境変数は D-Bus や IPC 等で非特権プロセスから見える
- 機密情報であることがわかりにくい
- 子プロセスに伝搬する

(ただ一番目については具体的に何を指しているかはわからない。[ある StackOverflow の回答][3] では、systemd がサービス開始時に設定を D-Bus にブロードキャストはするが、環境変数自体を流すような仕組みがあるわけではないとしている。-> [追記1](#d-bus))

こうした方法に代わり、systemd v250 からはサービスにクレデンシャルを安全に渡すための方法として `LoadCredentialEncrypted` や `SetCredentialEncrypted` ディレクティブを使用することができます。`LoadCredentialEncrypted` は予めクレデンシャルを暗号化したファイルを用意しそのパスを指定、サービス起動時に復号したファイルとして渡すという動きをします。`SetCredentialEncrypted` も似ていますが、こちらには暗号化したファイルのパスではなく暗号化した文字列を直接指定します。

### systemd-creds によるクレデンシャル管理

systemd-creds はこの暗号化のために使用されるツールです。暗号化は AES256-GCM により行われ、その鍵は `/var/lib/systemd/credential.secret` か (利用可能ならば) TPM2、あるいはその両方です。デフォルトでは両方を使おうとするようです。

システムで TPM2 が利用可能かは `systemd-creds has-tpm2` により確認できます。

```bash
$ systemd-creds has-tpm2
yes
+firmware
+driver
+system
```

暗号化は `systemd-creds encrypt` で行います。`--name` は省略可能なオプションでここで指定した値をキーとして `LoadCredentialEncrypted` に利用します。省略時は暗号化ファイル名が使われます。

```bash
# 暗号化ファイルを生成
# credential.plain.txt が平文ファイル、credential.txt に Base64 エンコーディングされた暗号化ファイルが生成される
$ sudo systemd-creds encrypt --name credential.txt credential.plain.txt credential.txt

# 平文ファイルを削除
$ shred -u cred-plain.txt
```

ここではお試し実行として `systemd-run` により bash を実行します。 

```bash
# クレデンシャルを渡しつつ bash サービス実行
$ sudo systemd-run -t -p DynamicUser=yes -p LoadCredentialEncrypted=credential.txt:$(pwd)/credential.txt /bin/bash
Running as unit: run-u867.service
Press ^] three times within 1s to disconnect TTY.
```

実行した bash から復号されたファイルを確認します。
サービスからは `CREDENTIALS_DIRECTORY` 環境変数でクレデンシャルを格納したディレクトリがわかります。

```bash
$ ls -l ${CREDENTIALS_DIRECTORY}/
-r-------- 1 run-u867 root 6 Nov  5 15:55 credential.txt

$ cat ${CREDENTIALS_DIRECTORY}/credential.txt
hello
```

`ls -l` の結果からわかるように、ファイルはサービスを実行するユーザのみがアクセスでき、かつ read-only となるように調整されます。なおここでは `DynamicUser` を指定したのでサービス用に一時的なユーザ (run-u867) が作成されています。

マウントを確認するとこのディレクトリには ramfs が使用されています。ramfs はスワップされない RAM ディスクなのでクレデンシャルの保存には適しているということだと思います。

```bash
$ mount -l
...
ramfs on /run/credentials/run-u867.service type ramfs (ro,nosuid,nodev,noexec,relatime,mode=700)
...
```

systemd の資料を読むとクレデンシャルをこのように管理するときは同時に `PrivateMounts=true` の設定が推奨されています。これはサービス用に mount namespace が新しく切るための既存の設定で、以下のように有効にすることで別サービスからのクレデンシャルファイルへのアクセスを防ぐことができます。

```bash
# あるターミナルで
(terminal1) $ sudo systemd-run -t -p PrivateTmp=true -p PrivateMounts=true -p DynamicUser=yes -p LoadCredential=abc:/etc/hosts /bin/bash
Running as unit: run-u311.service
...

# 別のターミナルで
(terminal2) $ sudo systemd-run -t -p PrivateTmp=true -p PrivateMounts=true -p DynamicUser=yes -p LoadCredential=abc:/etc/hosts /bin/bash
Running as unit: run-u316.service
...

# それぞれでは自分の credential しか見れない
(terminal1) $ ls /run/credentials/
run-u314.service
(terminal2) $ ls /run/credentials/
run-u316.service

# なおホストからは両方見える (DynamicUser=yes なので専用のユーザが動的に作成されている)
$ ls -l /run/credentials/
total 0
dr-x------ 2 run-u314 root 0 Nov  3 18:03 run-u314.service
dr-x------ 2 run-u316 root 0 Nov  3 18:03 run-u316.service
```

### 参考

- [Credentials - systemd.io][1]
- [The magic of systemd-creds - smallstep.com][2]

---

<div id="d-bus" />

#### 追記1: D-Bus によるサービス定義の取得

ここでの疑問に対してコメントを頂けたので少しこの点について自分でも確認してみました。

> これは systemctl show などでunitの属性をみるとEnvironment=の定義が見えることを言っていると思います

`SYSTEMD_LOG_LEVEL=debug` 環境変数付きで `systemctl show` を実行したデバッグログ例:

```
# my-sample は確認のために用意したサービス
$ SYSTEMD_LOG_LEVEL=debug systemctl show my-sample

running_in_chroot(): Permission denied
Bus n/a: changing state UNSET → OPENING
sd-bus: starting bus by connecting to /run/dbus/system_bus_socket...
Bus n/a: changing state OPENING → AUTHENTICATING
Successfully forked off '(pager)' as PID 151302.
Skipping PR_SET_MM, as we don't have privileges.
Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
Failed to execute 'pager', using next fallback pager: No such file or directory
Pager executable is "less", options "FRSXMK", quit_on_interrupt: yes
Showing one /org/freedesktop/systemd1/unit/my_2dsample_2eservice
Bus n/a: changing state AUTHENTICATING → HELLO
Sent message type=method_call sender=n/a destination=org.freedesktop.DBus path=/org/freedesktop/DBus interface=org.freedesktop.DBus member=Hello cookie=1 reply_cookie=0 signature=n/a error-name=n/a error-message=n/a
Got message type=method_return sender=org.freedesktop.DBus destination=:1.3284 path=n/a interface=n/a member=n/a cookie=1 reply_cookie=1 signature=s error-name=n/a error-message=n/a
Bus n/a: changing state HELLO → RUNNING
Sent message type=method_call sender=n/a destination=org.freedesktop.systemd1 path=/org/freedesktop/systemd1/unit/my_2dsample_2eservice interface=org.freedesktop.DBus.Properties member=GetAll cookie=2 reply_cookie=0 signature=s error-name=n/a error-message=n/a
Got message type=method_return sender=:1.2 destination=:1.3284 path=n/a interface=n/a member=n/a cookie=27878 reply_cookie=2 signature=a{sv} error-name=n/a error-message=n/a

... (以下通常通りユニットの設定出力)
Environment=foo=bar
```

この中で以下のログからサービスの情報取得のために D-Bus が利用されていること、また送信メッセージの詳細がわかります (可読性のために整形済)。

```
Sent message
  type=method_call
  sender=n/a
  destination=org.freedesktop.systemd1
  path=/org/freedesktop/systemd1/unit/my_2dsample_2eservice
  interface=org.freedesktop.DBus.Properties member=GetAll
  cookie=2
  reply_cookie=0
  signature=s
  error-name=n/a
  error-message=n/a
```

これを踏まえて自分で `dbus-send` してみると `systemctl show` と同様な情報が取得できます。

```bash
$ dbus-send --print-reply --system \
    --dest=org.freedesktop.systemd1 \
    /org/freedesktop/systemd1/unit/my_2dsample_2eservice \
    --type=method_call \
    org.freedesktop.DBus.Properties.GetAll \
    string:

method return time=1668239124.727145 sender=:1.2 -> destination=:1.3368 serial=28854 reply_serial=2
   array [
     ...
      dict entry(
         string "Environment"
         variant             array [
               string "foo=bar"
            ]
      )
      ...
   ]
```

[1]: https://systemd.io/CREDENTIALS/
[2]: https://smallstep.com/blog/systemd-creds-hardware-protected-secrets/
[3]: https://security.stackexchange.com/questions/264373/details-snoop-environment-variables-using-d-bus-ipc
