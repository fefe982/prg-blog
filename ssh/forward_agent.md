# 通过 ForwardAgent ，在远程机器上，使用本机 ssh key 进行验证

原来在 VS Code 里使用 Remote SSH 的时候，是在远程机器上生成 ssh key，然后把每一个远程服务器的 ssh public key 添加到 git 服务里面去。这样在 git 服务那里就会攒起一堆 key （因为服务器还挺多）。

最近忽然不想使用远程机器上的(默认) key 了，因为远程机器一般都是多人共享的。有时候，甚至大家用的都是同一个账户。我本来是想在远程机器上生成一个私有的 key (加上 passphrase)，然后用 ssh-agent 管理。但是尝试了很久，发现 VS Code 好像很难用到一个“全局”的 ssh-agent。所以似乎不太可行。

在查找解决方案的过程中，忽然发现了另外一种途径，ForwardAgent。

## ForwardAgent

ForwardAgent 是 ssh 的一个特性，可以让 ssh-agent 在本地机器上运行，并且把本机的 ssh-agent 代理到远程机器上。这样，远程机器就可以使用本机的 ssh key 了。

这样，就不需要使用远程机器上的 ssh key 了，与不需要为每一个服务器单独在 git 服务上添加 key 了。

## 配置

### 本机

本机上首先得启动 ssh-agent，并且把 key 添加到 ssh-agent 中（ssh-add）。（这个就不细说了，不同系统也不太一样。可以看一下[这个](https://code.visualstudio.com/docs/remote/troubleshooting#_setting-up-the-ssh-agent)，里面其实也有对 ForwardAgent 的介绍。）

然后需要修改一下本机的 ssh 配置文件。一般 Windows 在 `C:\Users\<用户名>\.ssh\config`，Linux 在 `~/.ssh/config`。没有就创建一个。

在配置文件中添加以下内容：

```
Host 远程机器地址
    ForwardAgent yes
```

或者偷懒的话，也可以直接写：

```
Host *
    ForwardAgent yes
```
这样可以对所有的远程机器都启用 ForwardAgent。

之后，新建立的 ssh 连接里都会多出一个环境变量，类似：`SSH_AUTH_SOCK="/tmp/ssh-XXXXXXXXXX/agent.1234567"`。这样，在 ssh 连接里再使用 ssh 的时候（比如 git），就可以通过它，使用本机的 key 进行验证。

注意新建的 ssh 连接可能需要在 ssh-agent 生效的情况下建立。而且已经存在的连接也可能需要断掉重新连接才成。

### 远程

远程机器上其实啥都不需要做。

不过，VS Code 的 Remote SSH ，在远程机器上会启动一个 server ，这个 server 是长时间运行的，并不会每个连接新建一个。所以，新增加的环境变量在已有的 server 里就不会生效，在 VS Code 里还是用不到本机的 key。解决方法也简单，就是把 server 都杀掉就好了。我是直接用 `ps aux | grep vscode` 找到所有进程，然后 kill 掉的。

用下面的命令可以直接得到所有 pid ，然后传给 `xargs kill` 就可以了。直接 pipe 有风险，`ps aux` 会找到所有用户的进程，`grep 'vscode'` 也可能找到无关的进程，所以得自己确认好，我也就不贴完整的命令了。

```
ps aux | grep 'vscode' | awk '{print $2}'
```
