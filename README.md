# Telegram ReVanced 自动构建部署教程

本仓库使用 GitHub Actions 自动完成：
- 获取最新 `ReVanced/revanced-cli` 和 `Aunali321/ReVancedExperiments` release
- 下载 Telegram 官方 APK（`https://telegram.org/dl/android/apk`）
- 使用自定义签名密钥进行 patch 与签名
- 上传 artifact，并发布/覆盖 `telegram-patched-latest` Release 资产

工作流文件：`.github/workflows/patch-telegram.yml`

## 1. 生成签名密钥（只做一次，必须是 BKS）

`revanced-cli` 当前签名库按 `BKS` 读取 keystore。  
如果使用 `PKCS12/JKS`，会报：`java.io.IOException: Wrong version of key store.`

在本地执行：

```bash
# 下载 BouncyCastle Provider（用于 keytool 处理 BKS）
curl -fL -o bcprov.jar \
  https://repo1.maven.org/maven2/org/bouncycastle/bcprov-jdk18on/1.78.1/bcprov-jdk18on-1.78.1.jar

keytool -genkeypair \
  -v \
  -keystore release.bks \
  -storetype BKS \
  -alias telegramrevanced \
  -keyalg RSA \
  -keysize 4096 \
  -validity 36500 \
  -providerclass org.bouncycastle.jce.provider.BouncyCastleProvider \
  -providerpath bcprov.jar
```

说明：
- `-alias` 可自定义，但后续要和 Secret `ANDROID_KEY_ALIAS` 完全一致。
- `keystore password` 与 `key password` 可以相同（推荐）。

### 如果你已经生成了 `release.keystore`（PKCS12），可直接转换为 BKS

```bash
curl -fL -o bcprov.jar \
  https://repo1.maven.org/maven2/org/bouncycastle/bcprov-jdk18on/1.78.1/bcprov-jdk18on-1.78.1.jar

keytool -importkeystore \
  -srckeystore release.keystore \
  -srcstoretype PKCS12 \
  -srcalias telegramrevanced \
  -destkeystore release.bks \
  -deststoretype BKS \
  -destalias telegramrevanced \
  -providerclass org.bouncycastle.jce.provider.BouncyCastleProvider \
  -providerpath bcprov.jar
```

转换完成后，后续都用 `release.bks`。

## 2. 把 keystore 转为 base64

### Linux
```bash
base64 -w 0 release.bks > keystore.b64
```

### macOS
```bash
base64 release.bks | tr -d '\n' > keystore.b64
```

## 3. 配置 GitHub Secrets

进入：`Repository -> Settings -> Secrets and variables -> Actions`

新增以下 4 个 Secrets：

1. `ANDROID_KEYSTORE_B64`
- 值：`keystore.b64` 文件内容（整行）

2. `ANDROID_KEYSTORE_PASSWORD`
- 值：keystore 密码

3. `ANDROID_KEY_ALIAS`
- 值：创建 keystore 时使用的 alias（例如 `telegramrevanced`）

4. `ANDROID_KEY_PASSWORD`
- 值：alias 对应 key 的密码

## 4. 触发构建

1. 提交并推送仓库代码
2. 打开 `Actions -> Patch Telegram APK`
3. 点击 `Run workflow`

成功后产物位置：
- `Actions` 的 `artifact`
- `Releases` 中固定 tag：`telegram-patched-latest`
  - 资产名：`telegram-patched-latest.apk`

## 5. 更新行为说明

- 只要持续使用同一个 keystore + alias + 密码，安装时可直接覆盖更新。
- 如果以前安装的是“其他签名”的包，第一次切换到新签名时可能需要先卸载旧包。

## 6. 常见问题

### Q1: `test -n "${ANDROID_KEYSTORE_B64}"` 失败
未配置 Secret 或名称拼写错误。

### Q2: `Keystore was tampered with, or password was incorrect`
`ANDROID_KEYSTORE_PASSWORD` 错误，或 `ANDROID_KEYSTORE_B64` 内容损坏。

### Q2.1: `Wrong version of key store`
你上传的是 `PKCS12/JKS`，不是 `BKS`。按本 README 的转换步骤改成 `release.bks` 后重新上传 Secret。

### Q3: `Cannot recover key` / alias 相关错误
`ANDROID_KEY_ALIAS` 或 `ANDROID_KEY_PASSWORD` 不匹配。

### Q4: patch 成功但找不到输出 APK
当前 workflow 已显式输出到：`work/telegram-patched-latest.apk`。

## 7. 安全建议

- 不要提交 `release.bks`、`release.keystore`、`keystore.b64` 到公开仓库。
- 建议仅在本地临时保存，配置完 Secrets 后删除：

```bash
rm -f release.bks release.keystore keystore.b64 bcprov.jar
```
