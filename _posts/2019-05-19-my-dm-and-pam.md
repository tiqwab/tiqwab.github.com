---
layout: post
title: 自作 display manager や PAM の話
tags: "display-manager, pam, linux"
comments: true
---

先日人生二度目の Arch Linux インストールを行いました。インストール自体は無事できたのですが、改めて X Window System あたりの知識が自分に全然無いことに気が付きました。

知識を得るには作るのが一番ということで手軽そうな display manager (DM) を作成してみることにしました。調べていると良質な資料として [How to Write a Display Manager][1] を見つけたのですが、ただ実際にやってみると display manager どうこうより PAM やプロセスの実行者みたいな話で躓く点が多かったのでここではそれらについて得たものをまとめてみました。

ここで扱う内容をまとめると

- PAM と設定ファイル
  - PAM を使用するアプリケーションは `/etc/pam.d` に設定ファイルを配置しないといけない
- pam\_unix モジュールのパスワード認証
  - root であるか自分自身を認証する場合でないと成功しない
- setuid
  - バイナリ実行ファイルには setuid というフラグがある
  - su でも pam\_unix を使っているのに sudo なしで他ユーザにスイッチできるのはこれのおかげ
- nosuid マウントオプション
  - これが設定されているファイルシステムでは setuid フラグは機能しない

ということになります。以後これらを順に見ていきます。

ちなみに作成した自作 display manager は [tiqwab/display-manager-sample][8] に配置しています。

### 1. PAM と設定ファイル

PAM (Pluggable Authentication Modules) とは Linux や FreeBSD で広く使われている認証の仕組みです。例えば display manager であったり su や sudo コマンド等で利用されています。

[How to Write a Display Manager][1] を読むとわかるように PAM を利用するアプリケーションは PAM のインタフェースに従ってある程度決まりきった処理を書く感じになります。

はじめこれだけ書いてうまく動かないなと思っていたのですが、PAM を動かすにはランタイム時に `/etc/pam.d` 以下に設定ファイルを配置する必要があるためでした。PAM ではあえてこの 2 つを切り離すことで環境ごとに認証の仕様を変えられる仕組みになっています。

PAM の詳細を知るには

- PAM の man ページを見る
- [Linux-PAM][2] を見る

するのがいいかなと思いますが、今回のようにとりあえず動かしたいだけなら既存の設定ファイルを真似して書くのがよさそうです。例えば `/etc/pam.d/su` は

```
#%PAM-1.0
auth		sufficient	pam_rootok.so
auth		required	pam_unix.so
account		required	pam_unix.so
session		required	pam_unix.so
```

となっているのでこれをそのまま `/etc/pam.d/my_display_manager` のようにして使用すれば unix ログインパスワード `/etc/shadow` の情報で認証を行うことができます。

ちなみにアプリケーション用の PAM 設定ファイルを配置しない場合は `/etc/pam.d/other` が参照されるようで、自分の環境では以下のように常に拒否されるような設定になっていました。

```
#%PAM-1.0
auth      required   pam_deny.so
auth      required   pam_warn.so
account   required   pam_deny.so
account   required   pam_warn.so
password  required   pam_deny.so
password  required   pam_warn.so
session   required   pam_deny.so
session   required   pam_warn.so
```

### 2. pam\_unix モジュールのパスワード認証

[How to Write a Display Manager][1] では Xephyr 上で自作 display manager の動作確認をしています。このとき display manager を実行するユーザ (自分自身) では PAM 認証に成功するのですが、他のユーザの名前とパスワードでは失敗するということに気付きました。上で述べたように auth グループには `pam_unix` モジュールを指定しているので、この部分のソースをざっと追ってみます。

アプリケーションでパスワード認証を行っているのは `pam_authenticate` であり [Authentication Management][3] を見るとモジュールの `pam_sm_authenticate` が呼ばれるようなので [linux-pam/pam\_unix\_auth.c][4] を見ます。

```c
int
pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv)
{
    ...

    /* verify the password of this user */
    retval = _unix_verify_password(pamh, name, p, ctrl);
    ...
}
```

コメントのようにパスワード認証を行っているのは [\_unix\_verify\_password][5] のようなので見てみます。

```c
int _unix_verify_password(pam_handle_t * pamh, const char *name
              ,const char *p, unsigned int ctrl)
{
    ...
    retval = get_pwd_hash(pamh, name, &pwd, &salt);
    ...
    if (retval != PAM_SUCCESS) {
        if (retval == PAM_UNIX_RUN_HELPER) {
            D(("running helper binary"));
            retval = _unix_run_helper_binary(pamh, p, ctrl, name);
    ...
}
```

`get_pwd_hash` では 対象ユーザの `/etc/passwd` に含まれる情報を取得するのですが、このときパスワードにあたるものが `x` (シャドウパスワード) だと `PAM_UNIX_RUN_HELPER` を返します。

`PAM_UNIX_RUN_HELPER` が返された場合 `_unix_run_helper_binary` で実際の認証が行われるのですが、これは `/usr/bin/unix_chkpwd` を呼び出すことになっています。

[unix\_chkpwd][6] の目的は `/etc/shadow` の情報と入力されたパスワード (のハッシュ値) で認証を行うことです。[PAM auth working just for user rundeck][7] という rundeck の issue で言われるように `unix_chkpwd` は自分自身を認証するケースか `/etc/shadow` を見れるようなユーザ (普通だと root のみのはず) で実行しているケースでないと通らないというのが、はじめの疑問の理由になります。なので試しに display manager を sudo で実行すれば他ユーザのパスワードでも認証に成功します。

`unix_chkpwd` の挙動をもう少し知るには 3 の内容が必要なので残りはそちらで触れます。

### 3. setuid

自作 display manager の PAM 設定ファイルは su から流用したものでした。しかし su は sudo をつけなくても機能します (他ユーザとしての認証ができている)。

```
$ echo $USER
user1
$ su -l user2
Password: 
$ echo $USER
user2
```

この違いは su 実行ファイルの setuid フラグによるものです。

```
$ ls -l /usr/bin/su
-rwsr-xr-x 1 root root 63448 Apr  9 23:22 /usr/bin/su
```

プロセスには実行者を表す情報として real user id, effective user id, saved set-user-id という 3 つがあります。saved set-user-id は一旦置いておくと real user id がプロセス実行者であり effective user id が権限を見る際に使われる id です。通常はいずれも 1000 であったり、sudo すればいずれも 0 というように一致した値になっています。

実行バイナリファイルに setuid フラグが立っている、というのは上のように `ls -l` で見たときに所有者の実行権限を表す部分に s という文字が出現する、ということなのですがこのとき「effective user id と saved set-user-id をファイルの所有者の uid に設定してファイルを実行する」という動作になります。上の su の場合所有者が root なので実行時には real user id がログインユーザ、effective user id と saved set-user-id が root ということで `/etc/shadow` といった情報を見ることができパスワード認証ができる... という感じです。

(ちなみに setgid フラグというのもありますがここでは省略)

上で見た [unix\_chkpwd][6] を見直すとはじめの方の処理にこのような分岐があります。

```c
int main(int argc, char *argv[])
{
    ...
    /*
     * Determine what the current user's name is.
     * We must thus skip the check if the real uid is 0.
     */
    if (getuid() == 0) {
      user=argv[1];
    }
    else {
      user = getuidname(getuid());
      /* if the caller specifies the username, verify that user
         matches it */
      if (strcmp(user, argv[1])) {
        user = argv[1];
        /* no match -> permanently change to the real user and proceed */
        if (setuid(getuid()) != 0)
            return PAM_AUTH_ERR;
      }
    ...
}
```

`unix_chkpwd` も所有者 root で setuid フラグあり、という実行ファイルなので main 開始時には effective user id , saved set-user-id は root のはずです。また `setuid` 関数は実行ファイルで見た setuid フラグと似たような役目のものですが、

- (必要な権限があれば) real user id, effective user id, saved set-user-id を指定した uid に変更する
- 指定した uid が現在の real user id あるいは saved set-user-id に等しいならば effective user id をそれに変更する

のいずれかの挙動をします (man setuid より)。

これらを踏まえるとこの分岐後にありうる real user id, effective user id, saved set-user-id の組み合わせは

- real, effective, saved いずれも root
  - root が認証を開始したケース
- real は認証対象と同じ user, effective, saved は root
  - 自分自身を認証するケース
- real, effective, saved いずれも認証対象の user
  - root ではないユーザが他一般ユーザとして認証しようとしているケース

となり、「自分自身を認証するケースか `/etc/shadow` を見れるようなユーザ (普通だと root のみのはず) で実行しているケースでないと通らない」というのが確かにということになります。

### 4. nosuid マウントオプション

setuid フラグの挙動確認のために `/tmp` 下でサンプルファイルを作成していたのですがそこで一つハマりどころがありました。

自分の環境では `/tmp` マウントを以下のようにしています。

```
$ mount -l | grep '/tmp'
tmpfs on /tmp type tmpfs (rw,nosuid,nodev)
```

マウント時には rw であったり relatime であったりマウントオプションを色々設定するかと思うのですが、その中で上のように nosuid というオプションがあります。このオプションがあるとマウントされたファイルシステムでは setuid フラグが (設定されているようには見えるけど) 無視されるという挙動になります。

nosuid 自体はセキュリティ的な意味合いのオプションっぽいですし `/tmp` にそれが設定されているのも妥当だと思います。ただ今回はそれに気付かず、しかもどうやら setuid フラグを設定することはできる (けど実行時に無視されていそう) という挙動だったので中々気付かず四苦八苦してしまいました。

[1]: https://www.gulshansingh.com/posts/how-to-write-a-display-manager/
[2]: http://www.linux-pam.org/
[3]: http://www.linux-pam.org/Linux-PAM-html/mwg-expected-of-module-auth.html
[4]: https://github.com/linux-pam/linux-pam/blob/v1.3.1/modules/pam_unix/pam_unix_auth.c#L99
[5]: https://github.com/linux-pam/linux-pam/blob/v1.3.1/modules/pam_unix/support.c#L705
[6]: https://github.com/linux-pam/linux-pam/blob/00b38702c70ad3847a2b3d38930a6a390df81645/modules/pam_unix/unix_chkpwd.c#L90
[7]: https://github.com/rundeck/rundeck/issues/2315
[8]: https://github.com/tiqwab/example/tree/master/display-manager-sample
