---
layout: post
title: Wayland の wl_shm とファイルディスクリプタ送信
tags: "wayland"
comments: true
---

Wayland は Linux デスクトップで使用されるディスプレイサーバプロトコルです。先日新しく Linux デスクトップ環境を整える際にはじめて Wayland を採用したことをきっかけに、しばらく [Wayland プロトコルの仕様][2] を眺めたり簡単なクライアントアプリケーションを作成したりしていました。

プロトコルで定義される仕様の一つにサーバ (コンポジタ)、クライアント (GUI アプリケーション) 間での共有メモリを扱う [wl\_shm][3] というものがあります。Wayland において共有メモリは、例えばウィンドウ描画用のメモリ領域をサーバと共有するために使用されたりします。共有メモリは `wl_shm_pool` というオブジェクトで管理されており、それを作成する API としてクライアント側には `wl_shm_create_pool` 関数が用意されています。

```c
 /**
  * @ingroup iface_wl_shm
  *
  * Create a new wl_shm_pool object.
  *
  * The pool can be used to create shared memory based buffer
  * objects.  The server will mmap size bytes of the passed file
  * descriptor, to use as backing memory for the pool.
  */
 static inline struct wl_shm_pool *
 wl_shm_create_pool(struct wl_shm *wl_shm, int32_t fd, int32_t size)
```

この関数はコメントや引数を見るとわかるように、共有したい領域をファイルディスクリプタとサイズで渡すようになっています。ここで渡すファイルディスクリプタは `memfd_create` や `mkostemp` 等で事前に作成した無名、一時ファイルを参照するものです。`wl_shm_create_pool` 後にクライアント、サーバそれぞれで `mmap` することで共有メモリを用意します。

... という流れは理解できるのですが、ただの整数である `fd` を送っても同じファイルを参照することはできないわけで、「あるプロセスが他のプロセスにファイルディスクリプタを渡す」というのはどうやって実現するんだっけ、というのを少し実装を読んで理解したので、以降ではその内容をメモしておこうと思います。

まず `man 2 memfd_create` にはまさにこの疑問の答えになるような記述があり、作成した無名ファイルを他のプロセスと共有する方法を 3 つ紹介しています。

>  - The process that called memfd\_create() could transfer the resulting file descriptor to the second process via a UNIX domain socket (see unix(7) and cmsg(3)). The secondprocess then maps the file using mmap(2).
>  - The second process is created via fork(2) and thus automatically inherits the file descriptor and mapping. (Note that in this case and the next, there is a natural trust relationship between the two processes, since they are running under the same user ID. Therefore, file sealing would not normally be necessary.)
>  - The  second  process opens the file /proc/pid/fd/fd, where \<pid\> is the PID of the first process (the one that called memfd\_create()), and \<fd\> is the number of the file descriptor returned by the call to memfd\_create() in that process. The second process then maps the file using mmap(2).

またここには含まれませんが、`pidfd_getfd` システムコールを使うという方法もあるようです (参考: [Seamless file descriptor transfer between processes with pidfd and pidfd\_getfd][4])。

wl\_shm はこの中の一番目、UNIX ドメインソケットを介して送信する方法を採用しています。それを確認するために `wl_shm_create_pool` を入口に実行される処理を見ていきます。

```c
static inline struct wl_shm_pool *
wl_shm_create_pool(struct wl_shm *wl_shm, int32_t fd, int32_t size)
{
	struct wl_proxy *id;

	id = wl_proxy_marshal_flags((struct wl_proxy *) wl_shm,
			 WL_SHM_CREATE_POOL, &wl_shm_pool_interface, wl_proxy_get_version((struct wl_proxy *) wl_shm), 0, NULL, fd, size);

	return (struct wl_shm_pool *) id;
}
```

内部では単に `WL_SHM_CREATE_POOL` を opcode として `wl_proxy_marshal_flags` を呼んでいます。この関数は `libwayland-client.so` に含まれており、ソースコードとしては Wayland リポジトリの [wayland-client.c][5] に当たるようです。

```c
/** Prepare a request to be sent to the compositor
 * ...
 * Translates the request given by opcode and the extra arguments into the
 * wire format and write it to the connection buffer.  This version takes an
 * array of the union type wl_argument.
 * ...
 */
WL_EXPORT struct wl_proxy *
wl_proxy_marshal_array_flags(struct wl_proxy *proxy, uint32_t opcode,
			     const struct wl_interface *interface, uint32_t version,
			     uint32_t flags, union wl_argument *args)
{
    struct wl_closure *closure;
    const struct wl_message *message;
    ...
    message = &proxy->object.interface->methods[opcode];
    ...
    closure = wl_closure_marshal(&proxy->object, opcode, args, message);
    ....
    if (wl_closure_send(closure, proxy->display->connection)) {
        wl_log("Error sending request: %s\n", strerror(errno));
    		 display_fatal_error(proxy->display, errno);
    }
    ....
}
```

`wl_closure_marshal` 関数の定義は、先程と同じリポジトリの [connection.c][6] に存在します。ここでは `wl_closure` を作成し、メッセージの各引数の型を見て必要に応じて処理を行っています。`wl_closure` は実装内部でのみ使用される構造体で、メッセージは一度 `wl_closure` として管理され、送信時にここからシリアライズされるようです。

Wayland プロトコルで送受信されるメッセージフォーマットについては、簡単には [Wire Format][7] や [wl\_message][8] に記述があります。ファイルディスクリプタのシンボルは "h" なので、ここではメッセージの引数に "h" があればファイルディスクリプタの複製や close-on-exec フラグの設定を行っています。

```c
struct wl_closure *
wl_closure_marshal(struct wl_object *sender, uint32_t opcode,
		   union wl_argument *args,
		   const struct wl_message *message)
{
    const char *signature;
    struct argument_details arg;
    int fd, dup_fd;
    ...
    signature = message->signature;
    for (i = 0; i < count; i++) {
        signature = get_next_argument(signature, &arg);
        
        switch (arg.type) {
            ...
            case 'h':
                fd = args[i].h;
                dup_fd = wl_os_dupfd_cloexec(fd, 0);
                if (dup_fd < 0) {
                    wl_closure_destroy(closure);
                    wl_log("error marshalling arguments for %s: dup failed: %s\n",
                           message->name, strerror(errno));
                    return NULL;
                }
               closure->args[i].h = dup_fd;
               break;
        }
    }
}
```

`wl_proxy_marshal_array_flags` に戻ると、次に同じ `connection.c` で定義されている `wl_closure_send` が呼ばれます。

```c
int
wl_closure_send(struct wl_closure *closure, struct wl_connection *connection)
{
	int size;
	uint32_t buffer_size;
	uint32_t *buffer;
	int result;

	if (copy_fds_to_connection(closure, connection))
		return -1;

	buffer_size = buffer_size_for_closure(closure);
	buffer = zalloc(buffer_size * sizeof buffer[0]);
	if (buffer == NULL)
		return -1;

	size = serialize_closure(closure, buffer, buffer_size);
	if (size < 0) {
		free(buffer);
		return -1;
	}

	result = wl_connection_write(connection, buffer, size);
	free(buffer);

	return result;
}
```

この中で実行される `copy_fds_to_connection`, `serialize_closure`, `wl_connection_write` を順番に見ていきます。

`copy_fds_to_connection` 内部ではメッセージの引数のうちシンボルが "h" (fd) のものについて `wl_connection_put_fd` が呼ばれます。

```c
static int
copy_fds_to_connection(struct wl_closure *closure,
		       struct wl_connection *connection)
{
	const struct wl_message *message = closure->message;
	uint32_t i, count;
	struct argument_details arg;
	const char *signature = message->signature;
	int fd;

	count = arg_count_for_signature(signature);
	for (i = 0; i < count; i++) {
		signature = get_next_argument(signature, &arg);
		if (arg.type != 'h')
			continue;

		fd = closure->args[i].h;
		if (wl_connection_put_fd(connection, fd)) {
			wl_log("request could not be marshaled: "
			       "can't send file descriptor\n");
			return -1;
		}
		closure->args[i].h = -1;
	}

	return 0;
}
```

```c
static int
wl_connection_put_fd(struct wl_connection *connection, int32_t fd)
{
	if (ring_buffer_size(&connection->fds_out) == MAX_FDS_OUT * sizeof fd) {
		connection->want_flush = 1;
		if (wl_connection_flush(connection) < 0)
			return -1;
	}

	return ring_buffer_put(&connection->fds_out, &fd, sizeof fd);
}
```

関数名にも現れていますが、メッセージの送受信にはリングバッファが使われており、`ring_buffer_put` で送信用のバッファに fd を書きます。送信用のバッファには通常のものとファイルディスクリプタ用が別で管理されており、それぞれ `connection->out` と `connection->fds_out` です。

```c
struct wl_ring_buffer {
	char data[4096];
	uint32_t head, tail;
};

struct wl_connection {
	struct wl_ring_buffer in, out;
	struct wl_ring_buffer fds_in, fds_out;
	int fd;
	int want_flush;
};


static int
ring_buffer_put(struct wl_ring_buffer *b, const void *data, size_t count)
{
	uint32_t head, size;

	if (count > sizeof(b->data)) {
		wl_log("Data too big for buffer (%d > %d).\n",
		       count, sizeof(b->data));
		errno = E2BIG;
		return -1;
	}

	head = MASK(b->head);
	if (head + count <= sizeof b->data) {
		memcpy(b->data + head, data, count);
	} else {
		size = sizeof b->data - head;
		memcpy(b->data + head, data, size);
		memcpy(b->data, (const char *) data + size, count - size);
	}

	b->head += count;

	return 0;
}
```

少し戻って `serialize_closure` ではバッファに送信するメッセージを書き込んでいます。"h" (fd) の場合の処理は既に `copy_fds_to_connection` で行っているのでここでは何もなく、他の型についてだけ処理が行われます。以下の抜粋では参考までに "i" (int) 型の引数の処理だけ記載しています。

```c
static int
serialize_closure(struct wl_closure *closure, uint32_t *buffer,
		  size_t buffer_count)
{
	const struct wl_message *message = closure->message;
	unsigned int i, count, size;
	uint32_t *p, *end;
	struct argument_details arg;
	const char *signature;

	p = buffer + 2;
	end = buffer + buffer_count;

	signature = message->signature;
	count = arg_count_for_signature(signature);
	for (i = 0; i < count; i++) {
		signature = get_next_argument(signature, &arg);

		if (arg.type == 'h')
			continue;

		switch (arg.type) {
        ...
		case 'i':
			*p++ = closure->args[i].i;
			break;
		default:
			break;
		}
	}

	size = (p - buffer) * sizeof *p;

	buffer[0] = closure->sender_id;
	buffer[1] = size << 16 | (closure->opcode & 0x0000ffff);

	return size;
}
```

ここで用意した `buffer` は `wl_connection_write` 関数内の処理で `connection->out` (送信用のリングバッファ) にコピーされます。

このあとが少し正確には追えていないのですが、`wl_connection_flush` でリングバッファに書き出したメッセージを送信していると思われます。そしてここが今回の目的である UNIX ドメインソケットを介したファイルディスクリプタ送信処理です。送信には `sendmsg` 関数を使います。

`sendmsg` では送信するメッセージを `struct msghdr` に含めます。メッセージ自体は `struct iovec` 型の `msg_iov` フィールドに詰められます。一方でファイルディスクリプタは `void *` 型である `msg_control` フィールドに詰められますが、その実体は `struct cmsghdr` であり、`build_cmsg` で用意されます。

```c
int
wl_connection_flush(struct wl_connection *connection)
{
	struct iovec iov[2];
	struct msghdr msg = {0};
	char cmsg[CLEN];
	int len = 0, count;
	size_t clen;
	uint32_t tail;

	if (!connection->want_flush)
		return 0;

	tail = connection->out.tail;
	while (connection->out.head - connection->out.tail > 0) {
		ring_buffer_get_iov(&connection->out, iov, &count);

		build_cmsg(&connection->fds_out, cmsg, &clen);

		msg.msg_iov = iov;
		msg.msg_iovlen = count;
		msg.msg_control = (clen > 0) ? cmsg : NULL;
		msg.msg_controllen = clen;

		do {
			len = sendmsg(connection->fd, &msg,
				      MSG_NOSIGNAL | MSG_DONTWAIT);
		} while (len == -1 && errno == EINTR);

		if (len == -1)
			return -1;

		close_fds(&connection->fds_out, MAX_FDS_OUT);

		connection->out.tail += len;
	}

	connection->want_flush = 0;

	return connection->out.head - tail;
}
```

```c
static void
build_cmsg(struct wl_ring_buffer *buffer, char *data, size_t *clen)
{
	struct cmsghdr *cmsg;
	size_t size;

	size = ring_buffer_size(buffer);
	if (size > MAX_FDS_OUT * sizeof(int32_t))
		size = MAX_FDS_OUT * sizeof(int32_t);

	if (size > 0) {
		cmsg = (struct cmsghdr *) data;
		cmsg->cmsg_level = SOL_SOCKET;
		cmsg->cmsg_type = SCM_RIGHTS;
		cmsg->cmsg_len = CMSG_LEN(size);
		ring_buffer_copy(buffer, CMSG_DATA(cmsg), size);
		*clen = cmsg->cmsg_len;
	} else {
		*clen = 0;
	}
}
```

ファイルディスクリプタを送信する際は、`build_cmsg` のように以下の設定を行います。

- `cmsg_len` に `struct cmsghdr` のサイズ + ファイルディスクリプタのサイズを設定する
- `cmsg_level` には `SOL_SOCKET` を設定する
- `cmsg_type` には `SCM_RIGHTS` を設定する

以上がクライアントから見た wl\_shm におけるファイルディスクリプタのコンポジタへの渡し方となります。ここでは紹介しないですが、コンポジタ側では受け取った `cmsg` から fd を取り出し `mmap` することでメモリの共有が実現されます。

[1]: https://docs.xfce.org/start
[2]: https://wayland.freedesktop.org/docs/html/
[3]: https://wayland.freedesktop.org/docs/html/apa.html#protocol-spec-wl_shm
[4]: https://copyconstruct.medium.com/seamless-file-descriptor-transfer-between-processes-with-pidfd-and-pidfd-getfd-816afcd19ed4
[5]: https://gitlab.freedesktop.org/wayland/wayland/-/blob/main/src/wayland-client.c
[6]: https://gitlab.freedesktop.org/wayland/wayland/-/blob/main/src/connection.c
[7]: https://wayland.freedesktop.org/docs/html/ch04.html#sect-Protocol-Wire-Format
[8]: https://wayland.freedesktop.org/docs/html/apb.html#Client-structwl__message
