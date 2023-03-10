# Subversion 的自动化部署

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Automated-Deployment-with-Subversion>

当在 web 开发环境中引入版本控制时，这个过程通常是逐步进行的。通常，一个开发人员会建立一个存储库，将所有代码放在那里，并维护两个工作副本:一个用于开发，另一个用作生产代码库。有时，这些甚至会是相同的工作副本。

在这一点上，进行版本控制是很简单的:开发人员从他的本地工作副本提交，并将“活的”工作副本更新到新的版本。当然，这种方法有几个缺点:

No testing phase:

Unless the developer has a testing environment set up on his local machine, there's no way to test code that's being written, until it's sitting on the production server. If the code is bug-ridden this presents a problem, since the live dataset is being corrupted.

Single-developer testing:

Even if the developer in question has a test environment set up, a new developer assigned to the product will not be able to test his work without a replica of that setup on his own machine. Ideally, a testing machine would be made available so that any developer could test their work without recourse to a specialised setup on their own computer.

Manual deployment:

Once a developer is satisfied that their work will stand up in production, whether this be tested with a common testbed or on their own workstation, the repository now has to be `export`ed, and copied to the live server. Depending on the convolutions required to connect to the live server, this can be a tedious and/or complicated process.

本文讨论了存储库设置的下一个阶段:测试和生产环境的分离，以及两者的自动更新。

### 颠覆挂钩

如果在过程中引入一个公共的测试服务器，并将代码库的工作副本放入测试中，那么开发人员测试代码就非常简单了:只需将他们的工作副本提交到存储库中，并在测试服务器上更新副本。

然而，与标准的生产服务器一样，仍然需要手动操作:在使用测试副本之前，必须对其进行更新。如果测试服务器真正有用的话，提交过程本身必须自动更新测试副本。幸运的是，Subversion 通过 *hook* 脚本系统提供了一个工具来执行作为提交一部分的动作。

Subversion 钩子是 Subversion 服务器在特定事件发生时运行的脚本。有几个动作可以触发钩子，但是我们只对`commit`钩子脚本感兴趣:

*   **start-commit** :在 Subversion 打开一个数据库事务来存储更新的工作之前运行。如果这个脚本返回一个错误，提交甚至在开始之前就失败了。
*   **预提交**:在 Subversion 将提交写入数据库之前运行。如果出现错误，提交将被撤销，并且不会保存。
*   **提交后**:在提交修订后立即运行，并有一个修订号。Subversion 服务器会忽略该脚本产生的任何错误。

对于自动化部署过程，将使用`post-commit`钩子来确保只有成功的提交被复制到测试和/或活动服务器。

### 更新测试环境

Subversion 服务器给了`post-commit` hook 脚本两个参数:刚刚更新的存储库的路径和更新的修订号。由于挂钩脚本是特定于存储库的，因此可以对其进行定制:

![Updating the testing environment](img/9bc38ccba4ef9598d92093f9acf48db3.png) *Figure 1: Control flow for a commit with automatic testing environment update*

钩子脚本可以是服务器可用的任何语言，包括 Bash、Python 或 Perl 出于本文的目的，我使用了 Bash，所以上面的控制流将转化为下面的`post-commit`钩子:

#### 测试服务器工作副本的自动更新

```
#!/bin/bash
REPO="$1"
REV="$2"

TEST_SERVER="192.168.1.55"

# Update the working copy on the test server
ssh -l root $TEST_SERVER -t "cd /var/www && svn up"
```

注意，在上面的例子中，测试环境是由`ssh`访问的，这意味着测试服务器上的`root`用户必须知道 Subversion 服务器的用户帐户的公钥。例如，如果 WebDAV 通过 Apache 访问 Subversion 存储库，Apache 进程以用户`nobody`的身份运行，调用`post-commit`钩子的用户是`nobody@SVN_SERVER`，必须为该用户准备一个公共/私有`ssh`密钥对，并将其复制到测试服务器。

### 部署通知

上例中的控制流在每次更新提交到 repo 时更新测试环境。接下来需要的是一种将提交的更新自动推送到生产系统的方法，使用提交操作的某个部分作为触发器。这方面的理想载体是提交消息:开发人员输入的描述，作为这次更新的原因。

Subversion 发行版中有一个命令可以用来检查存储库的属性:`svnlook`。这可用于检查最新修订版的提交消息，并寻找开发人员插入的指示部署请求的信号。

![Checking for deployment requests](img/4d7fc07d9ac0db49df2359f0a11566ca.png) *Figure 2: Control flow for a commit with deployment request*

部署信号可以像提交消息中的文本块一样简单:如果`post-commit`钩子检测到消息中的文本块，它将执行部署。如上所述，`svnlook`可以用来查看提交消息:

#### 检查提交消息:`svnlook`

```
PROD_SERVER="172.16.16.1"

if ( svnlook log -r $REV $REPO | grep "~~DEPLOY~~" )
then
    /usr/local/bin/svn-deploy $REPO $REV "root@${PROD_SERVER}:/var/www"
fi
```

通过使用`-r`标志请求特定的修订，我们可以确保传递给钩子脚本的修订号是被检查的修订号。尽管这个数字应该是回购中的最新版本，但最好在给出版本号时使用它。

### 执行部署

部署 Subversion repo 的一种方法是简单地将工作副本作为生产环境:在这种情况下，部署就像更新生产工作副本一样简单。这样做的缺点是，Subversion 控制文件和目录将在生产中可用；因为这些控制文件包括工作副本的文本库，所以这在纯文本文件中公开了后端代码和数据库接口。

Subversion 提供了一个命令，旨在生成一个“干净的”存储库副本:内容的转储，没有杂乱无章的目录。那个命令是`svn export`:

#### 导出回购的内容:`svn export`

```
svn export -r $REV "file://$REPO" /destination/path
```

通过请求一个特定的修订，就像`svnlook`一样，我们确保传递给钩子脚本的修订是导出的修订。导出完成后，可以上传到生产服务器，或者以任何需要的方式与`rsync`同步。

### 把东西放在一起

有了这些组件，我们可以将钩子和部署脚本放在一起:

#### `post-commit`:挂钩脚本

```
#!/bin/bash
REPO="$1"
REV="$2"

TEST_SERVER="192.168.1.55"
PROD_SERVER="172.16.16.1"

# Update the working copy on the test server
ssh -l root $TEST_SERVER -t "cd /var/www && svn up"

# Check for a deployment signal
if ( svnlook log -r $REV $REPO | grep "~~DEPLOY~~" )
then
    /usr/local/bin/svn-deploy $REPO $REV "root@${PROD_SERVER}:/var/www"
fi
```

#### `svn-deploy`:示例部署脚本

```
#!/bin/bash
REPO="$1"
REV="$2"
TARGET="$3"

# Connect to datacentre VPN
sudo pppd call datacentre nodetach
sudo route add -net 172.16.16.0/24 dev ppp0

# Export the repo
rm -rf /tmp/export
svn export -r $REV "file://$REPO" /tmp/export

# Synchronise with production
rsync -az -e ssh /tmp/export/* $TARGET
```

在这种特殊情况下，生产服务器位于数据中心的 VPN 后面，必须通过隧道进行部署。

### 运行部署:开发人员的观点

一旦存储库管理员将提交后挂钩放置到位，任何拥有签出的回购副本的开发人员都可以提交更新；任何更新都会导致测试环境副本被更新，从而允许一个公共的测试点。

部署是作为修订提交消息的一部分发出的，如下所示:

#### 部署提交消息示例

```
- Frontend: Checkout process: CC payment handling added
- Admin: Orders: Status dropdown now autosaves on change
~~DEPLOY~~
```

如上所述，部署代码"`~~DEPLOY~~`"必须出现在提交消息中，以便用信号通知部署。在部署之前，作为提交的一部分而更改的任何文件都将保存在新的修订中；复制到生产环境将包括自上次部署以来存储库中已更改的所有文件。

### 美好的未来:可能的进步

有几种方法可以增强上面的简单脚本。

Branch handling:

The scripts assume that the codebase is stored in its entirety in the repository, and only the trunk of the codebase is in the repo. If the repo is structured in a trunk-and-branch fashion, everything will be exported and deployed. By using a commit-message signal similar to the deployment signal, it should be possible to test a particular branch, by updating the trunk on the testing server and then copying the contents of the signalled branch over the top.

Changelog production:

By checking the commit messages using `svnlook`, it's possible to generate a Changelog for the repository: a list of revisions ordered by date, showing what changes were made to the codebase at each point. It's also possible to email the Changelog to the developers, if this is desired.

Database structure updates:

`svnlook` also allows the hook script to look at which files were modified with the commit. If a pre-determined SQL file is modified, this can be used to signal a change in database structure, and the changes can be applied to the production database through an `ssh` connection.

这些可能的变化中的每一个都会给自动部署系统带来复杂性；目前，这里给出的脚本是加速测试和部署过程的简单方法。

*版权所有伊姆兰·纳扎尔<tf@oopsilon.com2008 年*

*文章日期:2008 年 10 月 21 日*