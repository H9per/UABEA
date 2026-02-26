# UABEA CLI 命令行指令表

## 命令格式

```bash
UABEAvalonia <command> <options> [flags]
```

> **文件类型自动检测**：`export`、`import`、`list`、`info` 命令使用 `-f <file>` 提供文件路径后，
> 程序会自动判断该文件是 Bundle 还是 Assets 文件，无需手动指定 `bundle`/`assets` 子命令。

---

## 一、资源提取命令 (Export)

**语法：**
```bash
# 单个文件 (自动检测 Bundle 或 Assets)
UABEAvalonia export -f <file_path> -o <output_dir> [flags]

# 目录批量 (自动检测目录下每个文件)
UABEAvalonia export -d <directory> -o <output_dir> [flags]
```

**示例：**
```bash
# 从 Bundle 或 Assets 导出全部资源 (自动检测)
UABEAvalonia export -f game.bundle -o ./out/
UABEAvalonia export -f sharedassets0.assets -o ./out/

# 只导出 Texture2D，以 PNG 格式
UABEAvalonia export -f game.bundle -o ./out/ -t Texture2D --format png

# 批量处理目录（自动识别 bundle/assets 混合）
UABEAvalonia export -d ./GameData/ -o ./out/ --recursive

# 过滤指定 PathID
UABEAvalonia export -f sharedassets0.assets -o ./out/ -p 3,7,42
```

### 导出文件命名规则
CLI 导出文件采用与 GUI 一致的命名格式，方便直接通过文件名匹配导入：
- **格式**：`{AssetName}-{AssetsFileName}-{PathID}.{Extension}`
- **示例**：`Arial-sharedassets0-123.dat`、`Texture_Bg-level0-45.png`

---

## 二、资源写回命令 (Import)

**语法：**
```bash
# 单个文件 (自动检测 Bundle 或 Assets)
UABEAvalonia import -f <file_path> -i <input_dir> [flags]

# 目录批量 (自动检测目录下每个文件)
UABEAvalonia import -d <directory> -i <input_dir> [flags]
```

**示例：**
```bash
# 写回单个文件 (自动检测)
UABEAvalonia import -f game.bundle -i ./import/
UABEAvalonia import -f sharedassets0.assets -i ./import/

# 写回前备份
UABEAvalonia import -f game.bundle -i ./import/ --backup

# 指定纹理格式（默认保持原始格式）
UABEAvalonia import -f game.bundle -i ./import/ --tex-format RGBA32

# 批量写回
UABEAvalonia import -d ./GameData/ -i ./import/ --recursive
```

### 导入匹配规则
导入时自动搜索输入目录 (`-i`) 中文件名以 `-{AssetsFileName}-{PathID}.{ext}` 结尾的文件。
- 前缀（资源名）无需完全匹配，后缀正确即可识别。
- 支持直接导入 `.otf`/`.ttf` 字体文件替换字体数据。
- 导入 `.png`/`.tga` 图片到 Texture2D 时，**默认保持原始纹理格式**（如 ETC2、ASTC、DXT 等），通过 TexturePlugin.dll 编码。若插件不可用则回退为 RGBA32。可通过 `--tex-format <format>` 强制覆盖格式。

---

## 三、资源查询命令 (list / info)

### 3.1 列出资源 (list)

列出文件内所有资源，支持过滤和关键词搜索。

**语法：**
```bash
# 单个文件
UABEAvalonia list -f <file_path> [filters]

# 目录批量（自动检测每个文件）
UABEAvalonia list -d <directory> [filters] [--recursive]
```

**过滤参数（可组合使用）：**

| 参数 | 说明 |
|------|------|
| `-n <pattern>` | 按资源名过滤（支持多种匹配模式，见下方详细说明） |
| `-t <type>` | 按资源类型过滤（如 `-t Texture2D`） |
| `-p <pathid>` | 按 PathID 过滤（逗号分隔） |

**`-n` 名称过滤详细语法：**

| 语法 | 模式 | 示例 | 说明 |
|------|------|------|------|
| `keyword` | 子串匹配 | `-n hero` | 名称包含 `hero` 即匹配（默认，不区分大小写） |
| `=name` | 精确匹配 | `-n "=hero_icon"` | 名称必须完全等于 `hero_icon` |
| `~regex` | 正则匹配 | `-n "~hero_\d+"` | 使用正则表达式匹配 |
| `*`/`?` | 通配符 | `-n "hero_*"` | `*` 匹配任意字符，`?` 匹配单个字符 |
| `!pattern` | 取反排除 | `-n "!villain"` | 排除名称包含 `villain` 的资源 |
| `a,b,c` | OR 组合 | `-n "hero,warrior"` | 匹配任意一个模式即通过 |
| `!=name` | 组合 | `-n "!=bg_dark"` | 取反 + 精确：排除名称为 `bg_dark` 的资源 |
| `!~regex` | 组合 | `-n "!~bad_\d+"` | 取反 + 正则：排除匹配正则的资源 |

**匹配逻辑：**
- 正向匹配（无 `!`）之间为 **OR** 关系：任意一个匹配即通过
- 取反匹配（`!` 前缀）之间为 **AND** 关系：所有取反条件都必须通过
- 可混合使用，如 `-n "hero,warrior,!hero_bg"` 表示"包含 hero 或 warrior，但排除含 hero_bg 的"

**输出格式（bundle 文件）：**
```
Bundle: /path/to/game.bundle
PathID       Entry                               Type                 Size Name
----------------------------------------------------------------------------------------------------
3            level0                              Font                 7680 Arial
45           level0                              Texture2D           65536 sprite_bg
```

**输出格式（assets 文件）：**
```
Assets File: /path/to/sharedassets0.assets (12 assets)
PathID       Type                         Size Name
--------------------------------------------------------------------------------
3            Font                        12345 Arial
```

**搜索示例：**
```bash
# 子串匹配：找名称含 "bg" 的 Texture2D
UABEAvalonia list -d ./bundles/ -n bg -t Texture2D

# 精确匹配：找名称恰好为 "hero_icon" 的资源
UABEAvalonia list -f game.bundle -n "=hero_icon"

# 正则匹配：找名称匹配 hero_\d+ 模式的资源
UABEAvalonia list -f game.bundle -n "~hero_\d+"

# 通配符：找以 bg_ 开头的资源
UABEAvalonia list -f game.bundle -n "bg_*"

# OR 组合：找包含 hero 或 warrior 的资源
UABEAvalonia list -d ./GameData/ -n "hero,warrior" --recursive

# 取反：找所有 Texture2D 但排除名称含 shadow 的
UABEAvalonia list -f game.bundle -t Texture2D -n "!shadow"

# 混合：找 hero 或 warrior，但排除含 _disabled 的
UABEAvalonia list -f game.bundle -n "hero,warrior,!_disabled"
```

输出时，**只显示有匹配项的文件**，并在末尾汇总：
```
Bundle: ./bundles/level1.bundle
PathID       Entry     Type         Size Name
...
45           level1    Texture2D   65536 hero_idle

Bundle: ./bundles/ui.bundle
...

Total: 3 matching asset(s) across 2 file(s).
```

### 3.2 查看文件信息 (info)

打印文件元数据概要（不枚举内部资源）。

**语法：**
```bash
# 单个文件
UABEAvalonia info -f <file_path>

# 目录批量
UABEAvalonia info -d <directory> [--recursive]
```

---

## 四、压缩/解压命令

```bash
# 解压 Bundle
UABEAvalonia decompress -b <bundle_path> -o <output_path>

# 压缩 Bundle
UABEAvalonia compress -b <bundle_path> -o <output_path> --method lz4|lzma
```

---

## 五、应用 EMIP 补丁

```bash
UABEAvalonia apply emip -e <emip_file> -d <game_dir>
```

---

## 六、参数说明

### 6.1 路径参数

| 参数 | 全称 | 说明 |
|------|------|------|
| `-f` | `--file` | 文件路径（自动检测类型，推荐） |
| `-b` | `--bundle` | `-f` 的别名，兼容旧版；compress/decompress 专用 |
| `-a` | `--assets` | `-f` 的别名，兼容旧版语法 |
| `-d` | `--directory` | 目录批量操作 |
| `-o` | `--output` | 输出目录/文件路径 |
| `-i` | `--input` | 输入目录路径（用于导入） |

### 6.2 过滤参数

| 参数 | 全称 | 说明 |
|------|------|------|
| `-t` | `--type` | 资源类型过滤，逗号分隔（如 `-t Texture2D,Font`） |
| `-p` | `--pathid` | PathID 过滤，逗号分隔 |
| `-n` | `--name` | 资源名称过滤（支持子串/精确`=`/正则`~`/通配符`*?`/取反`!`/OR`,`） |

### 6.3 通用 Flags

| Flag | 说明 |
|------|------|
| `--format` | 导出格式：`raw`(默认)、`png`、`txt`、`wav`、`dump`、`json` |
| `--method` | 压缩算法：`lz4`、`lzma`、`none` |
| `--tex-format` | 覆盖纹理导入格式（如 `RGBA32`、`DXT5`、`ETC2_RGBA8`、`ASTC_RGBA_4x4`、`BC7`）；不指定则保持原始 |
| `--backup` | 写回前自动备份原文件 |
| `--no-backup` | 不创建备份 |
| `--recursive` | 目录模式下递归处理子目录 |
| `--dry-run` | 预览模式，不实际写入 |
| `--keepnames` | 导出时保留原始文件名 |
| `--kd` | 保留 `.decomp` 解压缓存文件 |
| `--fd` | 强制覆盖旧的 `.decomp` 缓存 |
| `--md` | 解压到内存（不写 `.decomp`） |
| `-v, --verbose` | 详细输出 |
| `-q, --quiet` | 静默模式 |

---

## 七、支持的资源类型 (-t)

`Texture2D`、`TextAsset`、`AudioClip`、`Font`、`Mesh`、`Shader`、`MonoBehaviour`、`GameObject`、`AssetBundle` 等。

**特殊格式支持：**
| 资源类型 | 导出格式 | 导入格式 |
|----------|----------|----------|
| Texture2D | `.png`（`--format png`）或 `.dat`（raw） | `.png`（默认保持原格式编码，可 `--tex-format` 覆盖）或 `.dat` |
| TextAsset | `.txt`（`--format txt`）或 `.dat` | `.txt` 或 `.dat` |
| AudioClip | `.wav`（`--format wav`）或 `.dat` | `.dat` |
| Font | `.dat`（raw data） | `.otf` 或 `.ttf`（直接替换字体数据） |

---

## 八、完整示例

### 8.1 字体替换流程

```bash
# 1. 导出字体资源
UABEAvalonia export -f sharedassets0.assets -o ./export/ -t Font

# 2. 导出文件名示例: Arial-sharedassets0-123.dat
#    将新字体改名并保持后缀: Arial-sharedassets0-123.otf

# 3. 导入新字体
UABEAvalonia import -f sharedassets0.assets -i ./export/
```

### 8.2 批量找bundle：哪些bundle包含名字含"bg"的Texture2D

```bash
# 1. 扫描所有 bundle，使用关键词 + 类型过滤
UABEAvalonia list -d ./GameData/ -n bg -t Texture2D --recursive

# 输出示例 (只显示有匹配的 bundle):
# Bundle: ./GameData/ui.bundle
# PathID       Entry     Type                  Size Name
# ---...
# 45           ui        Texture2D            65536 bg_main_menu
#
# Total: 1 matching asset(s) across 1 file(s).

# 2. 根据上面的输出，使用对应的 bundle 路径和 PathID 进行 import
UABEAvalonia import -f ./GameData/ui.bundle -i ./replace/
```

### 8.3 批量导出 Texture2D 为 PNG

```bash
UABEAvalonia export -d ./GameData/ -o ./Textures/ -t Texture2D --format png
```

### 8.4 Bundle 解压再压缩

```bash
UABEAvalonia decompress -b resources.bundle -o resources.unpacked
# 修改解压后的文件...
UABEAvalonia compress -b resources.unpacked -o resources.bundle --method lz4
```

### 8.5 旧版命令兼容（过渡期仍可用，会输出提示）

旧版语法自动识别并降级处理：
```bash
# 旧版语法 (仍然有效，但会显示 [Info] 提示)
UABEAvalonia export bundle -b game.bundle -o ./out/
UABEAvalonia import assets -a sharedassets0.assets -i ./import/
```
