主要需要配置字体和输入法。

字体参考[这篇文章](https://catcat.cc/post/2021-03-07/), 下载对应的字体包括中文和emoji，最后的配置如下:

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>

	<!-- Default system-ui fonts -->
	<match target="pattern">
		<test name="family">
			<string>system-ui</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>sans-serif</string>
		</edit>
	</match>

	<!-- Default sans-serif fonts -->
	<match target="pattern">
		<test name="family">
			<string>sans-serif</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Noto Sans CJK SC</string>
			<string>Noto Sans</string>
			<string>Twemoji</string>
		</edit>
	</match>

	<!-- Default serif fonts -->
	<match target="pattern">
		<test name="family">
			<string>serif</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Noto Serif CJK SC</string>
			<string>Noto Serif</string>
			<string>Twemoji</string>
		</edit>
	</match>

	<!-- Default monospace fonts -->
	<match target="pattern">
		<test name="family">
			<string>monospace</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Noto Sans Mono CJK SC</string>
			<string>Symbols Nerd Font</string>
			<string>Twemoji</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="prgname" compare="not_eq">
			<string>google-chrome-stable</string>
		</test>
		<test name="family" compare="contains">
			<string>Noto Sans Mono CJK</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Source Code Pro</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-HK</string>
		</test>
		<test name="family">
			<string>Noto Sans CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans CJK HK</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-HK</string>
		</test>
		<test name="family">
			<string>Noto Serif CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<!-- There is no Noto Serif CJK HK -->
			<string>Noto Serif CJK TC</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-HK</string>
		</test>
		<test name="family">
			<string>Noto Sans Mono CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans Mono CJK HK</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-TW</string>
		</test>
		<test name="family">
			<string>Noto Sans CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans CJK TC</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-TW</string>
		</test>
		<test name="family">
			<string>Noto Serif CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Serif CJK TC</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>zh-TW</string>
		</test>
		<test name="family">
			<string>Noto Sans Mono CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans Mono CJK TC</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ja</string>
		</test>
		<test name="family">
			<string>Noto Sans CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans CJK JP</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ja</string>
		</test>
		<test name="family">
			<string>Noto Serif CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Serif CJK JP</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ja</string>
		</test>
		<test name="family">
			<string>Noto Sans Mono CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans Mono CJK JP</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ko</string>
		</test>
		<test name="family">
			<string>Noto Sans CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans CJK KR</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ko</string>
		</test>
		<test name="family">
			<string>Noto Serif CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Serif CJK KR</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang">
			<string>ko</string>
		</test>
		<test name="family">
			<string>Noto Sans Mono CJK SC</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans Mono CJK KR</string>
		</edit>
	</match>

	<!-- Replace Arial -->
	<match target="pattern">
		<test name="family">
			<string>Arial</string>
		</test>
		<edit name="family" binding="strong">
			<string>Noto Sans CJK SC</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang" compare="contains">
			<string>en</string>
		</test>
		<test name="family" compare="contains">
			<string>Noto Sans CJK</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Noto Sans</string>
		</edit>
	</match>

	<match target="pattern">
		<test name="lang" compare="contains">
			<string>en</string>
		</test>
		<test name="family" compare="contains">
			<string>Noto Serif CJK</string>
		</test>
		<edit name="family" mode="prepend" binding="strong">
			<string>Noto Serif</string>
		</edit>
	</match>

</fontconfig>
```

接着配置中文输入法，比如fcitx5:

```bash
sudo pacman -S fcitx5-im fcitx5-chinese-addons
```

通过下面的方法启动:

```bash
fcitx5 -d --replace
```

启动后状态栏上会增加一个键盘的icon，右键打开配置界面，在界面右侧搜索`pinyin`然后加到input method里即可。
