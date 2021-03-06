# 对linux kernel double-free类型漏洞的较通用利用方法
+ update Wed Nov 29 16:39:01 HKT 2017
	+ linux kernel 4.14 released this month;
	+ [Kees Cook: "a bunch of security features I'm excited about"](https://outflux.net/blog/archives/2017/11/14/security-things-in-linux-v4-14/)
	+ [relevant PATCH 0: add a naive detection of double free or corruption](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ce6fa91b93630396ca220c33dd38ffc62686d499)
	这个补丁, 在进行连续的kfree的时候会起作用终止当前进程. 但是如果在连续的kfree之间其他进程kfree了相邻的一些对象, 导致page->freelist改写, 补丁无作用.
	+ [relevant PATCH 1: add SLUB free list pointer obfuscation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2482ddec)
	这个补丁, 将写入对象首地址(s->offset=0)的数据进行异或, 在申请对象的时候进行逆运算得到下一个申请的对象的位置.
	+ [relevant PATCH 2: prefetch next freelist pointer in slab_alloc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0ad9500e1)
	这个补丁, 会在申请一个对象的时候, 得到下一个可以申请的对象的位置, 同时对下一个对象保存的异或数据进行逆运算, 然后检测那个位置是否合法.
	+ 这几个补丁, 对SLUB的freelist进行了加强. 导致此文的方法不再适用.

+ update Sat Nov 25 19:25:45 HKT 2017
	+ [对encrypted_key的代码修改: KEYS: encrypted: sanitize all key material](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a9dd74b2)
+ UPDATE Mon Oct  9 10:07:00 HKT 2017
	[upstream对double free类型漏洞的修补, 来自SHA2017](http://blog.ptsecurity.com/2017/08/linux-block-double-free.html)

### 背景
由于系统在kfree一个对象时, 将前一个释放的空间的地址保存在将释放的空间的首地址.
如果执行如下代码:
```c
kfree(ptr0);
kfree(ptr1);
kfree(ptr1);
kfree(ptr2);
```
那么会得到`*(unsigned long *)ptr1 = ptr1; *(unsigned long *)ptr2 = ptr1;`
两个可以被重新申请的空间的首地址数据相同.
```c
kmalloc(size, ...);
kmalloc(size, ...);
...
```
可以得到指向同一空间的两个对象.
什么样的对象可以用来进行利用? 两个对象没有特定大小, 一个可以随意写入, 一个包含指针.
测试的所用的需要填充的slab对象为kmalloc-8192, 内核版本为3.10.x, cve-2017-2636, [测试POC](https://github.com/snorez/exploits/blob/master/cve-2017-2636/cve-2017-2636.c) [Alexander Popov的文档](https://a13xp0p0v.github.io/2017/03/24/CVE-2017-2636.html)

### 对象0: encrypted key
```c
struct encrypted_key_payload {
	struct rcu_head rcu;
	char *format;		/* datablob: format */
	char *master_desc;	/* datablob: master key name */
	char *datalen;		/* datablob: decrypted key length */
	u8 *iv;			/* datablob: iv */
	u8 *encrypted_data;	/* datablob: encrypted data */
	unsigned short datablob_len;	/* length of datablob */
	unsigned short decrypted_datalen;	/* decrypted data length */
	unsigned short payload_datalen;		/* payload data length */
	unsigned short encrypted_key_format;	/* encrypted key format */
	u8 *decrypted_data;	/* decrypted data */
	u8 payload_data[0];	/* payload data + datablob + hmac */
};
```
这个对象基础大小为0x48, 先看看对象如何申请的, 在`encrypted_key_alloc`函数中.
```c
static struct encrypted_key_payload *encrypted_key_alloc(struct key *key,
							 const char *format,
							 const char *master_desc,
							 const char *datalen)
{
	...
	ret = kstrtol(datalen, 10, &dlen);
	if (ret < 0 || dlen < MIN_DATA_SIZE || dlen > MAX_DATA_SIZE)
		return ERR_PTR(-EINVAL);

	format_len = (!format) ? strlen(key_format_default) : strlen(format);
	decrypted_datalen = dlen;
	payload_datalen = decrypted_datalen;
	if (format && !strcmp(format, key_format_ecryptfs)) {
		...
	}

	encrypted_datalen = roundup(decrypted_datalen, blksize);

	datablob_len = format_len + 1 + strlen(master_desc) + 1
	    + strlen(datalen) + 1 + ivsize + 1 + encrypted_datalen;

	/* 这个函数也比较重要 */
	ret = key_payload_reserve(key, payload_datalen + datablob_len
				  + HASH_SIZE + 1);
	if (ret < 0)
		return ERR_PTR(ret);

	/* 申请指定大小的对象 */
	epayload = kzalloc(sizeof(*epayload) + payload_datalen +
			   datablob_len + HASH_SIZE + 1, GFP_KERNEL);
	if (!epayload)
		return ERR_PTR(-ENOMEM);

	epayload->payload_datalen = payload_datalen;
	epayload->decrypted_datalen = decrypted_datalen;
	epayload->datablob_len = datablob_len;
	return epayload;
}
```
对于encrypted key的用法, 可以参考Documentations/security/keys-trusted-encrypted.txt
这里简单说一下用到的payload的格式.
`"new default user:user_key_desc payload_len"`.
函数参数中的`datalen`指向`payload_len`, `master_desc`指向`user:user_key_desc`, `format`指向`default`, `payload`最大为4096, 也即`encrypted_key_payload`对象最大的时候会取kmalloc-8192. 最小的时候由于加上了HASH_SIZE+1, 最小0x48+32+1=0x69
***因此这个对象可以落在 ~~kmalloc-96~~ kmalloc-128 - kmalloc-8192区域***

### 使用encrypted key的系统限制 以及 对应的策略
在/proc/sys/kernel/keys/中, 保存着当前系统普通用户能申请的key数以及总大小,限制了这个对象的喷的总数.
在`encrypted_update`函数中, 也调用了`encrypted_key_alloc`函数, 然后会释放之前申请的空间, 可以利用这个函数来进行交替性的堆喷.
```c
static int encrypted_update(struct key *key, struct key_preparsed_payload *prep)
{
	struct encrypted_key_payload *epayload = key->payload.data[0];
	struct encrypted_key_payload *new_epayload;
	char *buf;
	char *new_master_desc = NULL;
	const char *format = NULL;
	size_t datalen = prep->datalen;
	int ret = 0;

	if (test_bit(KEY_FLAG_NEGATIVE, &key->flags))
		return -ENOKEY;
	if (datalen <= 0 || datalen > 32767 || !prep->data)
		return -EINVAL;

	buf = kmalloc(datalen + 1, GFP_KERNEL);
	if (!buf)
		return -ENOMEM;

	buf[datalen] = 0;
	memcpy(buf, prep->data, datalen);
	ret = datablob_parse(buf, &format, &new_master_desc, NULL, NULL);
	if (ret < 0)
		goto out;

	/* update的时候, 如果master_desc不匹配, 返回EINVAL */
	ret = valid_master_desc(new_master_desc, epayload->master_desc);
	if (ret < 0)
		goto out;

	/* 校验完成, 申请新的payload */
	new_epayload = encrypted_key_alloc(key, epayload->format,
					   new_master_desc, epayload->datalen);
	if (IS_ERR(new_epayload)) {
		ret = PTR_ERR(new_epayload);
		goto out;
	}

	__ekey_init(new_epayload, epayload->format, new_master_desc,
		    epayload->datalen);

	memcpy(new_epayload->iv, epayload->iv, ivsize);
	memcpy(new_epayload->payload_data, epayload->payload_data,
	       epayload->payload_datalen);

	rcu_assign_keypointer(key, new_epayload);
	/* 释放之前的payload */
	call_rcu(&epayload->rcu, encrypted_rcu_free);
out:
	kfree(buf);
	return ret;
}
```

### 用encrypted_key_payload 来任意地址读
在double-free环境中, 另外一个对象覆盖了encrypted_key_payload的数据.
在`encrypted_read`函数中, 会读取`payload->format` `payload->master_desc` `payload->datalen` `payload->iv`指向的数据.
```c
static long encrypted_read(const struct key *key, char __user *buffer,
			   size_t buflen)
{
	struct encrypted_key_payload *epayload;
	struct key *mkey;
	const u8 *master_key;
	size_t master_keylen;
	char derived_key[HASH_SIZE];
	char *ascii_buf;
	size_t asciiblob_len;
	int ret;

	epayload = rcu_dereference_key(key);

	/* returns the hex encoded iv, encrypted-data, and hmac as ascii */
	asciiblob_len = epayload->datablob_len + ivsize + 1
	    + roundup(epayload->decrypted_datalen, blksize)
	    + (HASH_SIZE * 2);

	if (!buffer || buflen < asciiblob_len)
		return asciiblob_len;

	mkey = request_master_key(epayload, &master_key, &master_keylen);
	if (IS_ERR(mkey))
		return PTR_ERR(mkey);

	ret = get_derived_key(derived_key, ENC_KEY, master_key, master_keylen);
	if (ret < 0)
		goto out;

	ret = derived_key_encrypt(epayload, derived_key, sizeof derived_key);
	if (ret < 0)
		goto out;

	ret = datablob_hmac_append(epayload, master_key, master_keylen);
	if (ret < 0)
		goto out;

	/* 读取所需数据到buf中 */
	ascii_buf = datablob_format(epayload, asciiblob_len);
	if (!ascii_buf) {
		ret = -ENOMEM;
		goto out;
	}

	up_read(&mkey->sem);
	key_put(mkey);

	if (copy_to_user(buffer, ascii_buf, asciiblob_len) != 0)
		ret = -EFAULT;
	kfree(ascii_buf);

	return asciiblob_len;
out:
	up_read(&mkey->sem);
	key_put(mkey);
	return ret;
}
```
```c
static char *datablob_format(struct encrypted_key_payload *epayload,
			     size_t asciiblob_len)
{
	char *ascii_buf, *bufp;
	u8 *iv = epayload->iv;
	int len;
	int i;

	ascii_buf = kmalloc(asciiblob_len + 1, GFP_KERNEL);
	if (!ascii_buf)
		goto out;

	ascii_buf[asciiblob_len] = '\0';

	/* copy datablob master_desc and datalen strings */
	len = sprintf(ascii_buf, "%s %s %s ", epayload->format,
		      epayload->master_desc, epayload->datalen);

	/* convert the hex encoded iv, encrypted-data and HMAC to ascii */
	bufp = &ascii_buf[len];
	for (i = 0; i < (asciiblob_len - len) / 2; i++)
		bufp = hex_byte_pack(bufp, iv[i]);
out:
	return ascii_buf;
}
```

### 用encrypted key 提权
~~在`encrypted_destroy`函数中, 会将区域清0, 用此可完成提权.~~
```c
static void encrypted_destroy(struct key *key)
{
	struct encrypted_key_payload *epayload = key->payload.data[0];

	if (!epayload)
		return;

	memset(epayload->decrypted_data, 0, epayload->decrypted_datalen);
	kfree(key->payload.data[0]);
}
```

### 对象1: tty_struct.write_buf
```c
struct tty_struct {
	...
#define N_TTY_BUF_SIZE 4096

	...
	unsigned char *write_buf;
	int write_cnt;
	...
};
```
write_buf成员在`do_tty_write`函数中申请, 默认长度为2048.
```c
static inline ssize_t do_tty_write(
	ssize_t (*write)(struct tty_struct *, struct file *, const unsigned char *, size_t),
	struct tty_struct *tty,
	struct file *file,
	const char __user *buf,
	size_t count)
{
	ssize_t ret, written = 0;
	unsigned int chunk;

	ret = tty_write_lock(tty, file->f_flags & O_NDELAY);
	if (ret < 0)
		return ret;

	chunk = 2048;	/* 默认大小为2048 */
	if (test_bit(TTY_NO_WRITE_SPLIT, &tty->flags))
		chunk = 65536;	/* 如果标志置位, 则扩充大小到65536 */
	if (count < chunk)
		chunk = count;

	/* write_buf/write_cnt is protected by the atomic_write_lock mutex */
	if (tty->write_cnt < chunk) {
		unsigned char *buf_chunk;

		if (chunk < 1024)
			chunk = 1024;

		buf_chunk = kmalloc(chunk, GFP_KERNEL);
		if (!buf_chunk) {
			ret = -ENOMEM;
			goto out;
		}
		kfree(tty->write_buf);
		tty->write_cnt = chunk;
		tty->write_buf = buf_chunk;
	}

	/* Do the write .. */
	for (;;) {
		...
	}
	...
out:
	tty_write_unlock(tty);
	return ret;
}
```
从代码里面可以看出, write_buf的大小也是可控的, 大小[2048, 65536].
搜索代码, 得到`TTY_NO_WRITE_SPLIT`标志在n_hdlc.c中有路径会将其置位. 而write_buf指向的空间数据可以通过write系统调用来实现.
***NOTE:*** 需要注意的是, 用open打开tty时需要加上O_NONBLOCK标志.

### 利用步骤
结合encrypted_key_payload和tty_struct.write_buf, 完成利用.
+ 准备工作.
	+ 堆喷, 准备大量的所需大小的对象, 放入内核空间, 便于后续的检测反馈.
	+ 一个user-type的key, encrypted key需要这个.
	+ 一个或多个encrypted的key, 消耗key的总大小, 便于后续的检测反馈.
+ 触发double-free.
+ (交替)申请write_buf, encrypted_key_payload对象(使用encrypted_update函数).
	这个可能需要根据漏洞具体的环境来看申请的对象的顺序.
+ 检测`encrypted_update`的返回值, 如果为EINVAL, 则判断此时的内核空间中两个对象重叠.
+ 不停的调用`encrypted_update`检测合适的master_desc的位置.
	由于在read函数中需要master_desc的值, 所以我们首先需要遍历内核空间, 找到所需要的字串.
	所以我们设置好`encrypted_update`的参数(通过write_buf), 使调用过程如下.
	`encrypted_update -> encrypted_key_alloc -> key_payload_reserve`.
	当其返回EDQUOT时, 即找到对应的master_desc.
	在准备工作中的堆喷和消耗key的总大小, 即是为了找到这个master_desc.
+ 此时已经具备任意地址读的能力. 检测init_task (此过程未测试), 或者检测相应的task_struct结构中的comm字段, 找到目标进程的task_struct地址, 然后获取cred地址.
+ 调用`encrypted_destroy`, 完成提权.
	这个函数的调用需要先`keyctl_revoke`, 它只是将key进行一下标记, 然后调用gc.
	在测试过程中发现, 在`keyctl_revoke`之后, 立即调用add_key来申请一个与需要destroy的key相同的payload, 会立即触发`encrypted_destory`函数.

### 总结
+ 利用的主要对象为encrypted_key_payload, 适用大小为 ~~[96-8192]~~ [128-8192]的对象, POC中只进行了kmalloc-8192的测试.
+ write_buf可以应用在目标对象为[2048, 4096, 8192]大小时.
+ 依赖于slab的优先申请最近释放的块的特性.
+ KSPP中的CONFIG_SLAB_FREELIST_HARDENED应该已经加固了这个特性
