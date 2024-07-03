# GTK 应用打包玲珑时修复输入法问题

GTK 应用打包成玲珑后，使用拼音输入法输入选词较快时,选词后漏词或变成字母展示。

Issues 在这里 https://github.com/linuxdeepin/developer-center/issues/9409

## 问题排查

zsien 在 Issues 中回复

> 应用没有使用 fcitx5 的 GTK 插件

## 问题验证

需要确定下问题是否因为缺少插件导致，以下步骤也可用于排查其他应用缺少插件、依赖库等问题。

1. 首先使用 ll-cli 安装 BUG 中反馈的浏览器(org.deepin.browser)应用，

2. 切换到 /var/lib/linglong/layers/main/org.deepin.browser/6.5.3.2/x86_64/runtime 目录，这是浏览器的安装目录，查看目录中的 info.json 文件，可以看到浏览器使用的 base 和 runtime 版本

3. 切换到 /var/lib/linglong/layers/main/org.deepin.foundation/20.0.0.26/x86_64/runtime/ 目录，这是浏览器使用的 base 目录，将 files 目录改名字为 files.bk，再复制 files.bk 为 files（用于之后恢复原始的 base 目录）。

4. 使用`sudo chroot files`进入 base，执行`apt update && apt install fcitx5-frontend-gtk2` （浏览器使用的是 gtk2，如果是 gtk3 的应用则安装 fcitx5-frontend-gtk3)

5. 安装后再使用 `ll-cli run org.deepin.browser` 启动浏览器，验证问题得到解决，确实是 base 和 runtime 中缺少 gtk fcitx 插件导致的问题。

退出 chroot 环境，删除 files 目录，更改 files.bk 为 files

## 修复方案

问题得到验证后，需要考虑修复方案，最简单的当然时在 base 或 runtime 中添加相应的依赖库。

但考虑到 base 和 runtime 需要限制体积并保持兼容性，所有先从应用层修复问题。

玲珑支持在应用构建时安装 deb 包，可以在 org.deepin.browser 的 linglong.yaml 文件中添加 fcitx5-frontend-gtk2 deb 包依赖。

```yaml
sources:
  # linglong:gen_deb_source install fcitx5-frontend-gtk2
```

使用 vscode linglong 插件的 `Gen deb source`功能，自动生成相关的下载链接。

然后使用 ll-builder build 构建，使用 ll-builder run 进行测试，发现插件并未生效，漏字情况仍然存在。

搜索到 fcitx5-frontend-gtk2 包中有一个名字叫 fcitx5-gtk2-immodule-probing 的二进制，可用来探测环境

```bash
➜  ~ fcitx5-gtk2-immodule-probing
GTK_IM_MODULE=im
```

在宿主机使用 strace 跟踪二进制的系统调用，可查看到 fcitx5-gtk2-immodule-probing 尝试读取的文件中，

```bash
➜  ~ strace fcitx5-gtk2-immodule-probing 2>&1|grep openat
...
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache", O_RDONLY) = 5
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules/im-fcitx5.so", O_RDONLY|O_CLOEXEC) = 5
...
```

根据经验判断 immodules.cache 这个文件很可疑，使用 cat 查看文件内容，在开头写着文件由 gtk-query-immodules-2.0 生成，搜索该命令相关文档了解到，可以使用 --update-cache 参数更新 immodules.cache，在更新时可指定 so 文件路径。

由于玲珑的 base 在应用构建和运行时是只读的，无法直接更新 /usr 下的 immodules.cache 文件，使用`dpkg -S gtk-query-immodules-2.0`查到该命令是 `libgtk2.0-0` 包里面的，再使用`apt source libgtk2.0-0`下载源码，找到更新 immodules.cache 文件的部分

```cpp
  if (argc > 1 && strcmp (argv[1], "--update-cache") == 0)
    {
      cache_file = gtk_rc_get_im_module_file ();
      first_file = 2;
    }

  contents = g_string_new ("");
  g_string_append_printf (contents,
                          "# GTK+ Input Method Modules file\n"
                          "# Automatically generated file, do not edit\n"
                          "# Created by %s from gtk+-%d.%d.%d\n"
                          "#\n",
                          argv[0],
                          GTK_MAJOR_VERSION, GTK_MINOR_VERSION, GTK_MICRO_VERSION);
...

gchar *
gtk_rc_get_im_module_file (void)
{
  const gchar *var = g_getenv ("GTK_IM_MODULE_FILE");
  gchar *result = NULL;

  if (var)
    result = g_strdup (var);

  if (!result)
    {
      if (im_module_file)
	result = g_strdup (im_module_file);
      else
        result = gtk_rc_make_default_dir ("immodules.cache");
    }

  return result;
}
```

可以看到 cache_file 是使用`gtk_rc_get_im_module_file` 获取的，再继续跟踪 `gtk_rc_get_im_module_file`函数，会优先使用`GTK_IM_MODULE_FILE`环境变量，没有环境变量才使用默认目录。

## 修复实施

在 linglong.yaml 的 build 添加以下内容

```bash
# 环境变量 GTK_IM_MODULE_FILE 到应用目录
export GTK_IM_MODULE_FILE=$PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache
# 更新缓存文件
/usr/lib/x86_64-linux-gnu/libgtk2.0-0/gtk-query-immodules-2.0 --update-cache $PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules/im-fcitx5.so
```

这样就可以在构建应用时生成带 fctix 插件的 immodules.cache，但运行应用时仍会到默认的 /usr 目录查找，所以需要在运行应用之前再设置 GTK_IM_MODULE_FILE 环境变量。

由于 org.deepin.browser 的入口文件是一个 bash 脚本，在构建时使用 sed 添加环境变量设置即可。(如果应用入口文件是二进制，则需要自己生成一个脚本用作入口文件)

在 linglong.yaml 的 build 添加以下内容

```
sed -i "2i\export GTK_IM_MODULE_FILE=$PREFIX/lib/x86_64-linux-gnu/gtk-2.0/2.10.0/immodules.cache" $PREFIX/bin/browser
```
