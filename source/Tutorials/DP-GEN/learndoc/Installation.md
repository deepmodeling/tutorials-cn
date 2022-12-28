# DP-GEN 安装

安装 DP-GEN 有几种不同的方法，用户可以根据自己的系统环境选择最适合的方法。以下是对两种简单方法的详细介绍。

## 通过 pip 安装

如果您的设备可以联网，推荐您通过 pip 安装 DP-GEN，仅需要在您需要安装的 Python 虚拟环境终端中输入以下命令。

```sh
pip install dpgen
```

## 通过源码安装

如果您的设备不能联网，您可以通过使用源码的方式进行 DP-GEN 的安装。首先您需要一台联网设备从 Github 项目中获得源码：

```sh
git clone https://github.com/deepmodeling/dpgen.git
```

克隆完成后，您应该获得了一个 dpgen 文件夹。然后您需要将文件夹手动上传到需要安装的机器，并执行如下命令。

```sh
cd dpgen
pip install --user
```

这个命令会将 DP-GEN 安装到以下路近：`$Home/.local/bin/dpgen`。在使用前，您可能需要先激活变量路径 `Path`：

```sh
export PATH=$HOME/.local/bin:$PATH
```

您也可以将其添加到环境变量中，只需将该语句添加到 `$Home/.bashrc` 文件末尾。

## 验证安装

如果安装成功，DP-GEN(`dpgen`)将在终端中可用。为了测试安装，您可以在终端中输入：

```sh
dpgen -h
```

终端将会显示类似以下的帮助信息：
```sh
usage: dpgen [-h]
             {init_surf,init_bulk,auto_gen_param,init_reaction,run,run/report,collect,simplify,autotest,db} 
	     ...

dpgen is a convenient script that uses DeepGenerator to prepare initial data, drive DeepMDkit and analyze results. This script works based on several sub-commands with their own options. To see the options for the sub-commands, type "dpgen sub-command -h".
```

**翻译：阙浩辉**