# st
[st](https://st.suckless.org/) 是 suckless.org 社区中一个基于X的简单终端实现. 其代码短小精悍, 简洁高效, 十分的优雅.

## Prerequirement
fonts: Source Code Pro  
> debian系linux 需要从github上下载: https://github.com/adobe-fonts/source-code-pro  
> Linux 系统下载 \*.otf 格式，Windows 和 macOS 则下载 \*.ttf 格式  

```sh
sudo pacman -S adobe-source-code-pro-fonts nerd-fonts-source-code-pro
```

library (如果你在Debian类系统下使用)
```sh
sudo apt install libx11-dev libxft-dev
```


## patchs
打补丁的方式:  
将补丁`.diff` 文件下载到`st` 的源代码目录, 然后运行下面命令:
```sh
patch < 补丁.diff
```
若出现提示`file to patch:` 可能是删除了`config.def.h` 导致的, 输入`config.h` 即可.

### patch list
> 一些推荐的补丁

- [x] alpha 使终端透明 
- [x] dracula 主题配色
- [x] anysize 使st可以铺满整个屏幕
- [x] copyurl 选择复制终端下`Mod1+l` 显示的最后一个url
- [x] scrollback 使st可以向上回滚  
在这个补丁的基础下打下面的[补丁](https://github.com/theniceboy/st/blob/master/patches/st-lukesmith-externalpipe(if_you_have_scrollback).diff) 支持一个复制面板选项
```diff
diff --git a/config.def.h b/config.def.h
index 546edda..ee8558f 100644
--- a/config.def.h
+++ b/config.def.h
@@ -172,8 +172,20 @@ static MouseShortcut mshortcuts[] = {
 #define MODKEY Mod1Mask
 #define TERMMOD (ControlMask|ShiftMask)
 
+// from @LukeSmithxyz
+static char *openurlcmd[] = { "/bin/sh", "-c",
+    "sed 's/.*│//g' | tr -d '\n' | grep -aEo '(((http|https)://|www\\.)[a-zA-Z0-9.]*[:]?[a-zA-Z0-9./&%?#=_-]*)|((magnet:\\?xt=urn:btih:)[a-zA-Z0-9]*)'| uniq | sed 's/^www./http:\\/\\/www\\./g' | dmenu -i -p 'Follow which url?' -l 10 | xargs -r xdg-open",
+    "externalpipe", NULL };
+static char *copyurlcmd[] = { "/bin/sh", "-c",
+    "sed 's/.*│//g' | tr -d '\n' | grep -aEo '(((http|https)://|www\\.)[a-zA-Z0-9.]*[:]?[a-zA-Z0-9./&%?#=_-]*)|((magnet:\\?xt=urn:btih:)[a-zA-Z0-9]*)' | uniq | sed 's/^www./http:\\/\\/www\\./g' | dmenu -i -p 'Copy which url?' -l 10 | tr -d '\n' | xclip -selection clipboard",
+    "externalpipe", NULL };
+static char *copyoutput[] = { "/bin/sh", "-c", "st-copyout", "externalpipe", NULL };
+
 static Shortcut shortcuts[] = {
 	/* mask                 keysym          function        argument */
+	{ Mod1Mask|ControlMask, XK_l,           externalpipe,   {.v = openurlcmd } },
+	{ Mod1Mask,             XK_y,           externalpipe,   {.v = copyurlcmd } },
+	{ Mod1Mask,             XK_o,           externalpipe,   {.v = copyoutput } },
 	{ XK_ANY_MOD,           XK_Break,       sendbreak,      {.i =  0} },
 	{ ControlMask,          XK_Print,       toggleprinter,  {.i =  0} },
 	{ ShiftMask,            XK_Print,       printscreen,    {.i =  0} },
diff --git a/st.c b/st.c
--- a/st.c
+++ b/st.c
@@ -43,6 +44,9 @@
 #define ISCONTROL(c)		(ISCONTROLC0(c) || ISCONTROLC1(c))
 #define ISDELIM(u)		(u && wcschr(worddelimiters, u))
 
+// from @LukeSmithxyz
+#define TLINE_HIST(y)           ((y) <= HISTSIZE-term.row+2 ? term.hist[(y)] : term.line[(y-HISTSIZE+term.row-3)])
+
 enum term_mode {
 	MODE_WRAP        = 1 << 0,
 	MODE_INSERT      = 1 << 1,
@@ -1954,6 +1961,78 @@ sendbreak(const Arg *arg)
 		perror("Error sending break");
 }
 
+// from @LukeSmithxyz
+int
+tlinehistlen(int y)
+{
+	int i = term.col;
+
+	if (TLINE_HIST(y)[i - 1].mode & ATTR_WRAP)
+		return i;
+
+	while (i > 0 && TLINE_HIST(y)[i - 1].u == ' ')
+		--i;
+
+	return i;
+}
+
+// from @LukeSmithxyz
+void
+externalpipe(const Arg *arg)
+{
+	int to[2];
+	char buf[UTF_SIZ];
+	void (*oldsigpipe)(int);
+	Glyph *bp, *end;
+	int lastpos, n, newline;
+
+	if (pipe(to) == -1)
+		return;
+
+	switch (fork()) {
+	case -1:
+		close(to[0]);
+		close(to[1]);
+		return;
+	case 0:
+		dup2(to[0], STDIN_FILENO);
+		close(to[0]);
+		close(to[1]);
+		execvp(((char **)arg->v)[0], (char **)arg->v);
+		fprintf(stderr, "st: execvp %s\n", ((char **)arg->v)[0]);
+		perror("failed");
+		exit(0);
+	}
+
+	close(to[0]);
+	/* ignore sigpipe for now, in case child exists early */
+	oldsigpipe = signal(SIGPIPE, SIG_IGN);
+	newline = 0;
+	/* modify externalpipe patch to pipe history too      */
+	for (n = 0; n <= HISTSIZE + 2; n++) {
+		bp = TLINE_HIST(n);
+		lastpos = MIN(tlinehistlen(n) +1, term.col) - 1;
+		if (lastpos < 0)
+			break;
+		if (lastpos == 0)
+			continue;
+		end = &bp[lastpos + 1];
+		for (; bp < end; ++bp)
+			if (xwrite(to[1], buf, utf8encode(bp->u, buf)) < 0)
+				break;
+		if ((newline = TLINE_HIST(n)[lastpos].mode & ATTR_WRAP))
+			continue;
+		if (xwrite(to[1], "\n", 1) < 0)
+			break;
+		newline = 0;
+	}
+	if (newline)
+		(void)xwrite(to[1], "\n", 1);
+	close(to[1]);
+	/* restore */
+	signal(SIGPIPE, oldsigpipe);
+}
+
 void
 tprinter(char *s, size_t len)
 {
diff --git a/st.h b/st.h
index a1928ca..c2a2b9c 100644
--- a/st.h
+++ b/st.h
@@ -111,6 +111,9 @@ void *xmalloc(size_t);
 void *xrealloc(void *, size_t);
 char *xstrdup(char *);
 
+// from @LukeSmithxyz
+void externalpipe(const Arg *);
+
 /* config.h globals */
 extern char *utmp;
 extern char *stty_args;
diff --git a/Makefile b/Makefile
index 1b553fb..fa86cac 100644
--- a/Makefile
+++ b/Makefile
@@ -43,7 +43,9 @@ dist: clean
 install: st
 	mkdir -p $(DESTDIR)$(PREFIX)/bin
 	cp -f st $(DESTDIR)$(PREFIX)/bin
+	cp -f st-copyout $(DESTDIR)$(PREFIX)/bin
 	chmod 755 $(DESTDIR)$(PREFIX)/bin/st
+	chmod 755 $(DESTDIR)$(PREFIX)/bin/st-copyout
 	mkdir -p $(DESTDIR)$(MANPREFIX)/man1
 	sed "s/VERSION/$(VERSION)/g" < st.1 > $(DESTDIR)$(MANPREFIX)/man1/st.1
 	chmod 644 $(DESTDIR)$(MANPREFIX)/man1/st.1
@@ -53,6 +55,7 @@ install: st
 
 uninstall:
 	rm -f $(DESTDIR)$(PREFIX)/bin/st
+	rm -f $(DESTDIR)$(PREFIX)/bin/st-copyout
 	rm -f $(DESTDIR)$(MANPREFIX)/man1/st.1
 	rm -f /usr/share/applications/st.desktop
 
diff --git a/st-copyout b/st-copyout
new file mode 100644
index 0000000..8eafc58
--- /dev/null
+++ b/st-copyout
@@ -0,0 +1,12 @@
+#!/bin/sh
+# Using external pipe with st, give a dmenu prompt of recent commands,
+# allowing the user to copy the output of one.
+# xclip required for this script.
+# By Jaywalker and Luke
+tmpfile=$(mktemp /tmp/st-cmd-output.XXXXXX)
+trap 'rm "$tmpfile"' 0 1 15
+sed -n "w $tmpfile"
+ps1="$(grep "\S" "$tmpfile" | tail -n 1 | sed 's/^\s*//' | cut -d' ' -f1)"
+chosen="$(grep -F "$ps1" "$tmpfile" | sed '$ d' | tac | dmenu -p "Copy which command's output?" -i -l 10 | sed 's/[^^]/[&]/g; s/\^/\\^/g')"
+eps1="$(echo "$ps1" | sed 's/[^^]/[&]/g; s/\^/\\^/g')"
+awk "/^$chosen$/{p=1;print;next} p&&/$eps1/{p=0};p" "$tmpfile" | xclip -selection clipboard
```


- [x] fontfix.diff 修复字体 让st可以更好的显示nerdfont
```diff
diff --git a/x.c b/x.c
index 6170176..97dc9cb 100644
--- a/x.c
+++ b/x.c
@@ -1454,7 +1454,12 @@ xdrawglyphfontspecs(const XftGlyphFontSpec *specs, Glyph base, int len, int x, i
 	XftDrawSetClipRectangles(xw.draw, winx, winy, &r, 1);
 
 	/* Render the glyphs. */
-	XftDrawGlyphFontSpec(xw.draw, fg, specs, len);
+	/*XftDrawGlyphFontSpec(xw.draw, fg, specs, len);*/
+	FcBool b = FcFalse;
+FcPatternGetBool(specs->font->pattern, FC_COLOR, 0, &b);
+if (!b) {
+    XftDrawGlyphFontSpec(xw.draw, fg, specs, len);
+}
 
 	/* Render underline and strikethrough. */
 	if (base.mode & ATTR_UNDERLINE) {

```

- [x] desktopentry 是st创建一个desktop条目到`/usr/share/applications` 目录中
- [x] hidecursor 在终端下隐藏鼠标光标 当移动鼠标的时候又会显示
- [x] blinking cursor 使st光标闪烁
- [x] font2 当默认字体找不到编码时去font2数组中找



## 自定义的快捷键
| 键位              | 说明                         |
|-------------------|------------------------------|
| `Ctrl+Alt+l`      | 选择一个url用浏览器打开      |
| `Alt+o`           | 选择一个输出结果复制到剪切板 |
| `Alt+y`           | 选择一个url复制              |
| `Alt+j`           | 向下滚一行                   |
| `Alt+k`           | 向上滚一行                   |
| `Alt+Ctrl+j`      | 向下滚一屏幕                 |
| `Alt+Ctrl+k`      | 向上滚一屏幕                 |
| `Shift+Ctrl+`    | 加大终端字体大小             |
| `Shift+Ctrl+`    | 减小终端字体大小             |
| `Shift+Ctrl+Home` | 回到默认终端字体大小         |


## 字体
1. 字体的安装(e.g. 下载了一个nerd fond字体之后)
```sh
mv 下载的字体.ttf ~/.local/share/fonts/
fc-cache -vf ~/.local/share/fonts/
```

2. 查找系统中可以用的字体
```sh
# 例如我这里找 source code pro的字体
fc-list : family style | grep -i source
```

3. 替换`st/config.h` font数组中的字体, 或添加到font2数组中


