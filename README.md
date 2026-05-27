# 用最新版 Nerd Fonts (3.4.0) 给 Consolas 打补丁

记录一次把 `wclr/my-nerd-fonts` 仓库里的 Consolas NF（基于 Nerd Fonts 2.0）升级到 **Nerd Fonts 3.4.0** 的全过程，给后人参考。



## 背景

- 原仓库：<https://github.com/wclr/my-nerd-fonts>
- 原版本：Consolas 7.00 + Nerd Fonts 2.0
- 目标：把 Nerd Fonts 升到最新（3.4.0），保持 Consolas 字形不动
- 环境：Ubuntu 22.04，无 root 权限，无 Docker（snap 装的 Docker 配置有限制），conda-forge 没有 fontforge 包

## 准备

需要四个**原始未打补丁**的 Consolas TTF（来自 `C:\Windows\Fonts\`）：

| 文件 | 样式 |
| --- | --- |
| `consola.ttf` | Regular |
| `consolab.ttf` | Bold |
| `consolai.ttf` | Italic |
| `consolaz.ttf` | Bold Italic |

把它们放到工作目录下的 `fonts/` 子目录。**已经打过补丁的字体不能再次打补丁**——必须用原始文件。

## 步骤

### 1. 准备 FontForge

`font-patcher` 是个 FontForge Python 脚本，必须用 FontForge 自带的 Python 跑（系统 Python 没有 `fontforge` 模块）。

没 root、没 conda 包、没 Docker，但 FontForge 官方提供 AppImage：

```bash
curl -L -o FontForge.AppImage \
  https://github.com/fontforge/fontforge/releases/download/20251009/FontForge-2025-10-09-Linux-x86_64.AppImage
chmod +x FontForge.AppImage
./FontForge.AppImage --version   # 验证：Version: 20251009
```

### 2. 下载 Nerd Fonts 的 FontPatcher

只下 patcher 包就行，整个 nerd-fonts 仓库 7GB+ 没必要拉。

```bash
curl -L -o FontPatcher.zip \
  https://github.com/ryanoasis/nerd-fonts/releases/download/v3.4.0/FontPatcher.zip
unzip -o FontPatcher.zip -d FontPatcher
```

查最新版本号：

```bash
curl -s https://api.github.com/repos/ryanoasis/nerd-fonts/releases/latest \
  | grep tag_name
```

### 3. 跑 patcher

#### 几个踩过的坑

1. **AppImage 里调用脚本要给绝对路径**：`-script FontPatcher/font-patcher` 会被 AppImage 内部当成它自己 mount 路径下的相对路径，报 `No such file or directory`。改用 `-script "$(pwd)/FontPatcher/font-patcher"`。
2. **输出目录必须可写**：默认 cwd 在某些环境下是只读的，显式指定 `-out /tmp/nf-output/`。
3. **`--windows` 参数 v3 已删除**：原仓库命令是 `--complete --mono --windows`，3.x 改成 `--complete --mono` 就够了。Windows 兼容现在是默认行为。
4. **字体文件路径也建议用绝对路径**：避免 AppImage 切目录。

#### 命令

```bash
mkdir -p /tmp/nf-output
for f in consola.ttf consolab.ttf consolai.ttf consolaz.ttf; do
  ./FontForge.AppImage -script "$(pwd)/FontPatcher/font-patcher" \
    "$(pwd)/fonts/$f" --complete --mono \
    -out /tmp/nf-output/
done
```

每个文件大约 30-60 秒。完成后 `/tmp/nf-output/` 里会有：

```
ConsolasNerdFontMono-Regular.ttf
ConsolasNerdFontMono-Bold.ttf
ConsolasNerdFontMono-Italic.ttf
ConsolasNerdFontMono-BoldItalic.ttf
```

注意 v3 的命名规范变了——不再是冗长的 `Consolas Nerd Font Complete Mono Windows Compatible.ttf`，而是 `ConsolasNerdFontMono-<Style>.ttf`。

### 4. 替换仓库里的字体

```bash
cd my-nerd-fonts
rm "Consolas NF/"*.ttf
cp /tmp/nf-output/*.ttf "Consolas NF/"
```

### 5. 验证

用 fonttools 查看字体元信息：

```bash
pip install fonttools
python3 -c "
from fontTools.ttLib import TTFont
import os
for f in sorted(os.listdir('Consolas NF')):
    if not f.endswith('.ttf'): continue
    font = TTFont(f'Consolas NF/{f}')
    for r in font['name'].names:
        if r.platformID == 3 and r.nameID in (1,2,4,5,6):
            print(f, r.nameID, r.toUnicode())
"
```

应该看到：

- Family（nameID=1）= `Consolas Nerd Font Mono`
- Version（nameID=5）= `Version 7.00;Nerd Fonts 3.4.0`
- PostScript（nameID=6）= `ConsolasNFM`、`ConsolasNFM-Bold` 等

## 安装后在终端/编辑器里选什么名字

字体族名是 **`Consolas Nerd Font Mono`**。Regular/Bold/Italic/Bold Italic 四个变体自动归入同一族，编辑器会自动按需选择。

## 关于 patcher 参数

| 参数 | 说明 |
| --- | --- |
| `--complete` | 包含所有图标集（Powerline、FontAwesome、Material 等） |
| `--mono` | 等宽——所有字形塞进单格，终端必选 |
| `-out DIR` | 输出目录 |

如果只想要某些图标集，去掉 `--complete`，按需加 `--powerline --fontawesome --octicons` 等。

## 总结：未来再升级时

1. 拿到原始 Consolas TTF（`fonts/consola*.ttf`）
2. 下最新版 FontForge AppImage 和 FontPatcher.zip
3. 跑 `./FontForge.AppImage -script "$(pwd)/FontPatcher/font-patcher" "$(pwd)/fonts/X.ttf" --complete --mono -out /tmp/nf-output/`
4. 替换 `Consolas NF/` 下的旧文件



该升级过程使用Claude Code完成，并已经通过作者测试。