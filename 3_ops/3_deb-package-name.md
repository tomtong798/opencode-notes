# Debian 软件包 Package 名称知识点

## 概述

当使用 `apt remove <package-name>` 卸载软件时，`<package-name>` 来自 deb 包的 `DEBIAN/control` 文件中的 `Package` 字段。本文档说明如何确定和控制这个名称。

---

## 1. deb 包结构

一个标准的 deb 包由三部分组成：

```
package.deb
├── debian-binary        # deb 格式版本信息
├── control.tar.{gz|xz}  # 控制信息
│   └── control          # ← Package 名称在这里
│   └── {pre,post}inst   # 安装脚本
│   └── {pre,post}rm     # 卸载脚本
└── data.tar.{gz|xz}     # 实际安装的文件
    └── usr/
    └── opt/
    └── ...
```

### control 文件格式

```
Package: opencode
Version: 1.0.2
Section: devel
Priority: optional
Architecture: amd64
Maintainer: OPENCode <hello@opencode.ai>
Description: AI-powered code editor
```

**`Package` 字段就是 `apt remove` 时使用的名称。**

---

## 2. 解压方式对比

| 命令 | 解压内容 | 是否包含 control |
|------|----------|------------------|
| `dpkg-deb -x file.deb dir/` | 仅 data 部分 | ❌ 无 |
| `dpkg-deb -e file.deb dir/` | 仅 control | ✅ 有 |
| `dpkg-deb -R file.deb dir/` | 完整解压 | ✅ 有 |

### 实用命令

```bash
# 查看 Package 名称（不解压）
dpkg-deb -I package.deb | grep Package

# 查看完整 control 信息
dpkg-deb -I package.deb

# 仅解压 control
dpkg-deb -e package.deb /tmp/control-only
cat /tmp/control-only/control

# 完整解压
dpkg-deb -R package.deb /tmp/full-extract
cat /tmp/full-extract/DEBIAN/control

# 仅解压文件（无 control）
dpkg-deb -x package.deb /tmp/files-only
```

---

## 3. 推断 Package 名称的方法

当没有 control 文件时（如 `dpkg-deb -x` 解压），可以通过以下方式推断：

### 方法一：文档目录（最可靠）

```bash
# Debian 规范要求每个包必须有文档目录
ls /usr/share/doc/*/
# 输出: /usr/share/doc/opencode/  → Package 名是 opencode
```

### 方法二：changelog 文件

```bash
zcat /usr/share/doc/<package>/changelog.gz | head -1
# 输出: opencode (1.0.2) ...  → Package 名是 opencode
```

### 方法三：直接查看原 deb

```bash
dpkg-deb -I package.deb | grep Package
```

---

## 4. electron-builder 打包配置

electron-builder 对 deb 和 rpm 的 Package 名称控制方式不同。

### 配置示例

```typescript
// electron-builder.config.ts
const config = {
  appId: "ai.opencode.desktop",
  productName: "OPENCode",  // 显示名称
  
  linux: {
    target: ["AppImage", "deb", "rpm"],
  },
  
  // RPM: 可以显式指定 packageName
  rpm: {
    packageName: "opencode",
  },
  
  // DEB: 无直接的 packageName 选项
  // Package 名来自 productName 的转换
  deb: {
    // 可通过 fpm 参数间接控制
    fpm: ["--name", "opencode"],
  },
}
```

### Package 名称来源优先级

#### RPM 格式
1. `rpm.packageName` 配置 ✅ 显式控制
2. `productName` 转换（如未配置）

#### DEB 格式
1. `deb.fpm: ["--name", "xxx"]` 参数
2. `productName` 转换（小写、去空格）
3. `package.json` 的 `name` 字段（备选）

**重要：** electron-builder 没有 `deb.packageName` 配置项。

### productName 转换规则

| productName | 转换后的 Package 名 |
|-------------|---------------------|
| `OPENCode` | `opencode` |
| `OPENCode Dev` | `opencode-dev` 或 `opencode` |
| `My App Name` | `my-app-name` |

---

## 5. package.json 与 Package 名的关系

```json
{
  "name": "@opencode-ai/desktop-electron",
  "productName": "OPENCode",
  "version": "1.0.2"
}
```

| 字段 | 用途 | 是否影响 deb Package 名 |
|------|------|------------------------|
| `name` | npm 包名 | ❌ 不影响（electron-builder 不使用） |
| `productName` | 应用显示名 | ✅ 是主要来源 |
| `version` | 版本号 | ❌ 不影响 |

---

## 6. 多渠道/多环境配置

```typescript
// electron-builder.config.ts
function getConfig() {
  const base = { /* 基础配置 */ }
  
  switch (channel) {
    case "dev":
      return {
        ...base,
        productName: "OPENCode Dev",
        rpm: { packageName: "opencode-dev" },
        // deb 会使用 "opencode-dev" 或 "opencode"
      }
    
    case "beta":
      return {
        ...base,
        productName: "OPENCode Beta",
        rpm: { packageName: "opencode-beta" },
      }
    
    case "prod":
      return {
        ...base,
        productName: "OPENCode",
        rpm: { packageName: "opencode" },
      }
  }
}
```

---

## 7. 验证与调试

### 查看已安装包信息

```bash
# 查看已安装包的详细信息
dpkg -s opencode

# 列出包安装的文件
dpkg -L opencode

# 查看包的 control 文件
cat /var/lib/dpkg/status | grep -A 20 "^Package: opencode"
```

### 查看未安装的 deb

```bash
# 查看信息
dpkg-deb -I package.deb

# 查看文件列表
dpkg-deb -c package.deb

# 查看内容（解压）
dpkg-deb -R package.deb /tmp/extract
```

### 修改已生成的 deb

```bash
# 解压
dpkg-deb -R original.deb /tmp/work

# 修改 DEBIAN/control
sed -i 's/Package: old-name/Package: new-name/' /tmp/work/DEBIAN/control

# 重新打包
dpkg-deb -b /tmp/work modified.deb
```

---

## 8. 常见问题

### Q: 为什么 apt remove 找不到包？

**A:** Package 名可能与文件名不同。
- 文件名: `opencode-electron-linux-amd64.deb`
- Package 名: `opencode`

```bash
# 正确方式
dpkg-deb -I file.deb | grep Package
```

### Q: 如何确保 deb 和 rpm 的 Package 名一致？

**A:** 显式配置：

```typescript
const packageName = "opencode"

return {
  productName: "OPENCode",
  rpm: { packageName },
  deb: { fpm: ["--name", packageName] },
}
```

### Q: electron-builder 支持哪些 deb 配置项？

**A:** 主要配置项：

```typescript
deb: {
  depends: ["gconf2", "gconf-service"],  // 依赖
  categories: ["Development"],           // 菜单分类
  priority: "optional",                   // 优先级
  fpm: ["--name", "custom-name"],         // fpm 参数
  afterInstall: "scripts/postinst.sh",   // 安装后脚本
  afterRemove: "scripts/postrm.sh",       // 卸载后脚本
}
```

---

## 9. 参考资料

- [Debian Policy Manual - Control files](https://www.debian.org/doc/debian-policy/ch-controlfields.html)
- [electron-builder - Linux Options](https://www.electron.build/configuration/linux)
- [FPM - Effing Package Management](https://github.com/jordansissel/fpm)
- `man dpkg-deb`