# ssh-keygen 的正确打开方式

----

前几天真香了 `MacBook Pro 15` ，那么最近就开始配环境啦！

今天主要对咕咕以及github的对一套认证链都配了一下，然后就可以在mbp上写博客啦～真香！

对 `ssh-keygen` 这个命令进了一次踩坑实践，因为之前在 **windows** 上配置ssh用的是`source tree`上的`putty`。

我输入命令直接回车

> ssh-keygen

然后提示我

>Generating public/private rsa key pair.
>Enter file in which to save the key (/home/[user]/.ssh/id_rsa):

根据他这个字面上的提示，我好像要输入一个文件路径，然后随着他不断提示我文件不存在以后，我亲手建立了一个这样的目录结构和文件 `/home/[user]/sshkey/.ssh/github_rsa` ，之后再次调用此命令，提示我是否要覆盖已存在的文件，那当然覆盖了 —— 所以ssh的key文件生成在了 `/home/[user]/sshkey/.ssh/` 目录下，并且没有设置安全密码。当我将 **.pub** 文件中的内容拷贝到github上的配置中保存后，竟然无法通过SSH协议的认证！

然后我就去查看了下`man ssh-keygen`，但是并没有得到特别针对性的信息。后来在网上发现一个网友说可以使用`ssh-add`命令将指定的SSH文件添加到 **验证代理** 上，我尝试调用如下命令

> ssh-add /home/[user]/sshkey/.ssh/github_rsa

但是反馈给我的提示竟然是因为我没有设置密码，所以不可以这么做！



于是我发现既然从一开始就很不对劲了，那么直接重来好了～

这次学乖了，输入`ssh-keygen`以后我再也没有输入文件路径，直接按了回车，果然文件生成在了默认的 **/home/[user]/.ssh/id_rsa** 路径！但是我也记得输入了密码（密码好像不是必须）。这回果然就通过了验证。。。。可以「快乐git」了。



我再去查看了下`man ssh-add`，果然这个命令是将指定路径的SSH文件放到 **/home/[user]/.ssh/** 目录下。。所以相当于SSH协议验证的私钥应当只是约定在这个路径下的，好吧！

看了下正确的常用生成方式应该是：

`ssh-keygen -f [文件名] -C [备注]`

> ​     **ssh-keygen** [**-q**] [**-b** bits] [**-t** **dsa** | **ecdsa** | **ed25519** | **rsa**] [**-N** new_passphrase] [**-C** comment] [**-f** output_keyfile]
>
> ​     **ssh-keygen** **-p** [**-P** old_passphrase] [**-N** new_passphrase] [**-f** keyfile]
>
> ​     **ssh-keygen** **-i** [**-m** key_format] [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-e** [**-m** key_format] [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-y** [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-c** [**-P** passphrase] [**-C** comment] [**-f** keyfile]
>
> ​     **ssh-keygen** **-l** [**-v**] [**-E** fingerprint_hash] [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-B** [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-D** pkcs11
>
> ​     **ssh-keygen** **-F** hostname [**-f** known_hosts_file] [**-l**]
>
> ​     **ssh-keygen** **-H** [**-f** known_hosts_file]
>
> ​     **ssh-keygen** **-R** hostname [**-f** known_hosts_file]
>
> ​     **ssh-keygen** **-r** hostname [**-f** input_keyfile] [**-g**]
>
> ​     **ssh-keygen** **-G** output_file [**-v**] [**-b** bits] [**-M** memory] [**-S** start_point]
>
> ​     **ssh-keygen** **-T** output_file **-f** input_file [**-v**] [**-a** rounds] [**-J** num_lines] [**-j** start_line] [**-K** checkpt] [**-W** generator]
>
> ​     **ssh-keygen** **-s** ca_key **-I** certificate_identity [**-h**] [**-U**] [**-D** pkcs11_provider] [**-n** principals] [**-O** option] [**-V** validity_interval] [**-z** serial_number] file ...
>
> ​     **ssh-keygen** **-L** [**-f** input_keyfile]
>
> ​     **ssh-keygen** **-A** [**-f** prefix_path]
>
> ​     **ssh-keygen** **-k** **-f** krl_file [**-u**] [**-s** ca_public] [**-z** version_number] file ...
>
> ​     **ssh-keygen** **-Q** **-f** krl_file file ...
