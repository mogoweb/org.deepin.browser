# GTK 应用打包玲珑时修复输入法问题

GTK 应用打包成玲珑后，使用拼音输入法输入选词较快时,选词后漏词或变成字母展示。

Issues 在这里 https://github.com/linuxdeepin/developer-center/issues/9409

## 问题排查

zsien 在 Issues 中回复

> 应用没有使用 fcitx5 的 GTK 插件

## 问题验证

cd 到 /var/lib/linglong/layers/main/org.deepin.foundation/20.0.0.26/x86_64/runtime/，这是浏览器使用的 base 。将 files 目录改名字为 files.bk，在复制 files.bk 为 files。

使用`sudo chroot files`进入 base，执行`apt update apt install fcitx5-frontend-gtk2`，browser 使用的是 gtk2，如果是 gtk3 的应用则安装 fcitx5-frontend-gtk2

安装后再使用 `ll-cli run org.deepin.browser` 启动浏览器，验证问题得到解决，确实是 gtk 缺少 fcitx 插件

退出 chroot 环境，删除 files 目录，更改 files.bk 为 files

## 修复实施

由于玲珑的 base 在应用构建和运行时是只读的，所以缺少的插件要安装到应用自身的目录。

在 org.deepin.browser 的 linglong.yaml 文件中添加 fcitx5-frontend-gtk2 deb 包依赖，测试未生效，搜索到 fcitx5-frontend-gtk2 包中有一个名字叫 fcitx5-gtk2-immodule-probing 的二进制，可用来探测会环境，执行后输出`GTK_IM_MODULE=im`，显然插件并没有生效。

查看 fcitx5-frontend-gtk2 源代码，fcitx5-gtk2-immodule-probing 是从 /usr/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache 文件中读取的环境，而这个文件可以使用 GTK_IM_MODULE_FILE 环境变量自定义路径。

查看 immodules.cache 文件内容，在开头写着文件由 gtk-query-immodules-2.0 生成，搜索该命令相关文档了解到，可以使用 --update-cache 参数更新 immodules.cache，在更新时可指定 so 插件文件。

因为玲珑只允许应用更改 PREFIX 环境变量指定的目录，所以先设置环境变量 GTK_IM_MODULE_FILE 到这里面，然后使用 gtk-query-immodules-2.0 更新缓存文件。

在 linglong.yaml 的 build 添加以下内容

```
export GTK_IM_MODULE_FILE=$PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache
/usr/lib/x86_64-linux-gnu/libgtk2.0-0/gtk-query-immodules-2.0 --update-cache $PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules/im-fcitx5.so
```

这样就可以在构建时生成带 fctix 的 immodules.cache，但运行应用时仍会到默认的/usr 目录查找，所以需要在运行应用之前再设置 GTK_IM_MODULE_FILE 变量。

由于 org.deepin.browser 的入口文件是一个 bash 脚本，在构建时使用 sed 添加环境变量设置即可。如果入口文件是二进制，则需要自己生成一个脚本用作入口文件。

在 linglong.yaml 的 build 添加以下内容

```
sed -i "2i\export GTK_IM_MODULE_FILE=$PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache" $PREFIX/bin/browser
```
