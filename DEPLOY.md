# 部署说明(内部私有 NuGet 源)

基于 loicsharma/BaGet 的 fork,含本地修复。

文中占位符约定:
- `<内网地址>`:BaGet 容器所在机器的内网 IP 与端口(如 `10.0.0.x:5555`)
- `<公网地址>`:经 nginx/端口映射对外暴露的地址(如 `example.com:45555` 或公网 IP:端口)

## 与上游的差异

- `src/BaGet.Core/Extensions/PackageArchiveReaderExtensions.cs`:
  nuspec 中 readme/icon 路径按 Windows 打包惯例含反斜杠(如 `docs\PACKAGE.md`),
  而 zip 条目名一律为正斜杠;NuGet.Packaging 查条目不做路径归一化(实测 5.10.0 与 6.8.2 皆如此),
  导致解析 nuget.org 官方包抛 FileNotFoundException,推送被误判为无效包(HTTP 400)。
  修复:读取前统一将反斜杠替换为正斜杠(此问题无依赖升级捷径,只能靠本补丁)。

- `src/Directory.Build.props`:NuGet.* 依赖 5.10.0 → 6.8.2,消除 NU1903 高危 / NU1901 低危漏洞告警。

## 构建镜像(服务器上)

```bash
git clone https://github.com/iamddatm/BaGet /opt/baget-src
cd /opt/baget-src
docker build -t baget:fixed .
```

## 运行容器

```bash
docker run -d --name baget --restart unless-stopped -p <内网端口>:80 \
  -e AllowPackageOverwrites=true \
  -e ApiKey=<API密钥> \
  -e Database__ConnectionString="Data Source=/var/baget/baget.db" \
  -e Storage__Path=/var/baget \
  -e Mirror__Enabled=true \
  -e Mirror__PackageSource=https://nuget.azure.cn/v3/index.json \
  -v <数据卷路径>:/var/baget \
  baget:fixed
```

## 前置 nginx 反代(必须)

```nginx
proxy_set_header Host $http_host;
```

BaGet 按请求 Host 生成服务索引里的绝对 URL;不加这条时索引公布的是内网地址,
外网客户端 restore 与推送全部超时。修改后所有老客户端需执行一次
`dotnet nuget locals http-cache --clear`,否则继续用缓存的旧索引。

## 踩过的坑(重建容器/排障时必读)

1. **`Database__ConnectionString` 必须指向卷内路径(如 `/var/baget/baget.db`)**。
   默认值是相对路径 `baget.db`,落在容器层里——每次 `docker rm` 重建都会清空数据库,
   而存储卷文件还在,两边不一致后推送报 500(CreateNew 撞上无主残留文件)。若已发生:
   清掉卷内 `packages/` 目录后重推。
2. **`AllowPackageOverwrites=true`**:重复版本包直接推送返回 409,开启后覆盖(先删后写)。
3. **客户端 NuGet 要求 HTTPS**:HTTP 源必须在其 NuGet.Config 中显式
   `allowInsecureConnections="true"`,且推送/还原用**源名**引用(裸 URL 匹配不到该配置)。
4. **API 密钥无法明文存进 NuGet.Config**:`apikeys` 节存的是 DPAPI 加密值,
   手写明文会报 "not a valid Base-64 string";`dotnet` CLI 无 setApiKey 命令,
   每次推送带 `-k <密钥>` 即可(或装完整版 nuget.exe 执行一次 setApiKey)。

## 日常使用

客户端注册源(一次性):

```powershell
dotnet nuget add source http://<公网地址>/v3/index.json --name BaGet --allow-insecure-connections
```

推送包:

```powershell
dotnet nuget push -s BaGet -k <API密钥> <包路径>.nupkg
# 或直接 URL(NuGet 会按 URL 匹配到已配置源的 allowInsecureConnections)
dotnet nuget push -s http://<公网地址>/v3/index.json -k <API密钥> <包路径>.nupkg
```

验证推送成功且可下载:

```bash
curl -o /dev/null -w "%{http_code} %{size_download}B\n" \
  http://<公网地址>/v3/package/<小写包id>/<版本>/<小写包id>.<版本>.nupkg
```

## 已知遗留

- 运行时为 .NET Core 3.1(已 EOL),上游停更;内网只读场景暂可接受,
  长期建议评估迁移维护活跃的 fork BaGetter。
- 镜像包(nuget.azure.cn 回源)包文件不落盘;需完全离线的包请手动 push 进本地存储。
- `<数据卷路径>` 现含数据库与全部已推包,需纳入备份。
- `AllowPackageOverwrites=true` 意味着持密钥者可覆盖任意已推包;迁移稳定后可关闭以防误覆盖。
