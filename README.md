# svn-server

Subversion server in Docker: Apache + mod_dav_svn over http, with the iF.SVNAdmin web UI (XSS/CSRF fixed fork). Based on Alpine Linux.

以 Docker 執行的 Subversion 伺服器：透過 http(WebDAV)存取，附 iF.SVNAdmin 網頁管理介面。

本專案源自已封存的 [elleFlorio/svn-docker](https://github.com/elleFlorio/svn-docker)(MIT),之後大幅改寫：改用官方 Alpine + tini、PHP 8.4、移除 svnserve 與 s6-overlay、改用修補過 XSS/CSRF 的 iF.SVNAdmin。原作者的版權聲明保留在 [LICENSE](LICENSE)。

## 內容

| 元件 | 版本 / 說明 |
|---|---|
| 基底 | `alpine:3.24`,PID 1 為 `tini`,Apache 前景執行 |
| SVN 存取 | Apache + mod_dav_svn,`http://<主機>/svn/<repo>`,Basic Auth + 權限檔 |
| 網頁管理 | iF.SVNAdmin,`http://<主機>/svnadmin` |
| PHP | `php84`(含 CVE-2026-109618 / 109619 修補) |
| svn:// | **不提供**(svnserve 預設允許匿名讀取，已移除) |

## 取得 image 與執行

直接用 release 版本(GitHub Actions 推 `v*` tag 時自動 build、測試、發布):
```
docker pull ghcr.io/talisych/svn-server:<版本>      # 或 :latest
```
或自行 build(需含 submodule):
```
git clone --recurse-submodules https://github.com/talisych/svn-server.git
cd svn-server && docker build -t ghcr.io/talisych/svn-server:local .
```

```
docker run -d --name svn-server --restart unless-stopped -p 80:80 \
  --log-opt max-size=50m --log-opt max-file=5 \
  -v <hostpath>:/home/svn -v svn_config:/etc/subversion -v svnadmin_config:/opt/svnadmin/data \
  ghcr.io/talisych/svn-server:<版本>
```

發布新版:`git tag -a vX.Y.Z -m ... && git push origin vX.Y.Z`,流程見 `.github/workflows/release.yml`。

| 路徑 | 內容 | 建議 |
|---|---|---|
| `/home/svn` | repositories | bind mount 或 named volume |
| `/etc/subversion` | `passwd`、權限檔 `subversion-access-control` | named volume |
| `/opt/svnadmin/data` | iF.SVNAdmin 設定 | named volume |

## 設定

建立 http 存取帳號(iF.SVNAdmin 只認 MD5 `$apr1$` / SHA1 雜湊，請用 `-bm`,不要用 `-B`):
```
docker exec -t svn-server htpasswd -bm /etc/subversion/passwd <username> <password>
```

瀏覽器開 `http://localhost/svn` 應要求帳密並顯示 repo 列表。

網頁管理介面 `http://localhost/svnadmin`:第一次進入會顯示設定頁，各項按 Test 確認後儲存，再指定管理員。容器內的路徑：
- SVNAuthFile:`/etc/subversion/subversion-access-control`
- SVNUserFile:`/etc/subversion/passwd`
- SVNParentPath:`/home/svn`
- svn / svnadmin:`/usr/bin/svn`、`/usr/bin/svnadmin`

## Log

全部輸出到容器 stdout / stderr,用 `docker logs svn-server` 查看，並以 `--log-opt` 輪替。
- `svn-op ...`:每個 SVN 操作一行(checkout、commit、update、log…),含來源 IP、使用者、repo
- 其他 stdout:Apache access log,含 iF.SVNAdmin 與**瀏覽器**瀏覽 / 下載 `/svn` 的請求(svn 客戶端的原始請求不記錄，改由 `svn-op` 一行一個操作)
- stderr:Apache error log,含認證失敗

```
docker logs svn-server 2>/dev/null | grep '^svn-op'
```

## 安全

- iF.SVNAdmin 來自 [talisych/iF.SVNAdmin](https://github.com/talisych/iF.SVNAdmin) 分支 `fix/xss-csrf`(commit `b222fe0`):upstream master 1.7.0(未正式發布)+ CSRF(CVE-2019-15128)與 XSS 修補。
- 額外防護:session cookie `SameSite=Strict; HttpOnly`;`/svnadmin` 加 `X-Frame-Options`、`X-Content-Type-Options`、`Referrer-Policy`;`data/` 與 `cli.php` 禁止外部存取;隱藏 Apache / PHP 版本。
- 只在內網使用時，帳號密碼即可。若對外開放，建議再限制 `/svnadmin`(`/svn` 由 Apache 驗證、不經 PHP,不受影響):iF.SVNAdmin 登入沒有失敗次數限制，且 PHP 登入前漏洞(如 PHP2Shell)不受密碼保護。可在 `svnadmin.conf` 加 `Require ip <網段>`,或在 `/svnadmin` 前加一層 Apache Basic Auth。

iF.SVNAdmin 以 git submodule 引用 fork(`iF.SVNAdmin/`),程式修改在 fork 進行。更新到 fork 的新 commit:
```
git -C iF.SVNAdmin fetch && git -C iF.SVNAdmin checkout <commit>
git add iF.SVNAdmin && git commit -m "Update iF.SVNAdmin to <commit>"
```

## 從舊版(elleFlorio/svn-docker、s6 / PHP 8.1)升級

- svn:// 與 port 3690 已移除：先確認所有 working copy 都是 `http://`(`svn info | grep '^URL'`)。
- repos、`/etc/subversion`、iF.SVNAdmin 資料要改由 `apache` 擁有，升級後執行一次：
  ```
  docker exec svn-server chown -R apache:apache /home/svn /etc/subversion /opt/svnadmin/data
  ```
  (以 root `svnadmin create` 建立的 repo,舊版本來就無法經 http commit,chown 後即正常。)
- 舊版 iF.SVNAdmin 設定(`config.ini`、角色)可直接沿用。
- 「使用者加入群組」頁面改名為 `groupassign.php`,舊網址 `usergroupassign.php` 會 404。
- 舊版最後狀態保留為 tag `legacy-php81`。

## 授權

MIT,見 [LICENSE](LICENSE)。iF.SVNAdmin 為 GPL-2.0,見 `iF.SVNAdmin/license.txt`。
