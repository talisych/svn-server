# svn-server 維運手冊

適用：以 docker compose 部署、資料放在 `./data/` 的 svn-server。以下指令都在 compose 目錄(例如 `/opt/svn-server`)以 root 執行。

## 部署範本

`docker-compose.yml`:
```yaml
services:
  svn-server:
    container_name: svn-server
    image: ghcr.io/talisych/svn-server:<版本>
    ports:
      - "80:80"
    volumes:
      - ./data/svn:/home/svn
      - ./data/svn_config:/etc/subversion
      - ./data/svnadmin_config:/opt/svnadmin/data
    restart: unless-stopped
    logging:
      driver: journald
      options:
        tag: svn-server
```

- log 交給主機 journald:重建容器、升級都不會遺失(主機需有 `/var/log/journal` 才會跨重開機保留)。
- 不要再掛 `dav_svn.conf`,會蓋掉 image 內的設定(UTF-8、操作 log)。

## 啟動 / 停止 / 狀態

```
docker compose up -d                 # 啟動(或套用 compose 變更)
docker compose restart               # 重啟
docker compose down                  # 停止並移除容器(資料在 ./data,不受影響)
docker ps --filter name=svn-server   # 狀態
```

## 升級 / 回退

```
sed -i 's#svn-server:<舊版>#svn-server:<新版>#' docker-compose.yml
docker compose pull && docker compose up -d
docker image inspect ghcr.io/talisych/svn-server:<新版> --format '{{index .RepoDigests 0}}'
```
回退：把版本改回舊版，再 `docker compose up -d`。版本列表見 GitHub Releases。

## 帳號

```
docker exec svn-server htpasswd -bm /etc/subversion/passwd <帳號> <密碼>   # 新增或改密碼
docker exec svn-server htpasswd -D /etc/subversion/passwd <帳號>           # 刪除
docker exec svn-server cut -d: -f1 /etc/subversion/passwd                  # 列出
```
- 一定要用 `-bm`(MD5)。`-B`(bcrypt)建的帳號 svn 能用，但無法登入 iF.SVNAdmin。
- 群組與權限(`./data/svn_config/subversion-access-control`)建議用網頁 `http://<主機>/svnadmin` 管理，改完立即生效。

## Repository

建立(擇一):
- 網頁：svnadmin → Repositories → Create
- 指令:`docker exec -u apache svn-server svnadmin create /home/svn/<名稱>`

**不要用 root 建立**(沒加 `-u apache`):repo 會屬於 root,透過 http 無法 commit(`Permission denied`)。若已發生:
```
docker exec svn-server chown -R apache:apache /home/svn/<名稱>
```

查看:
```
docker exec svn-server svnlook youngest /home/svn/<名稱>   # 最新 revision
du -sh ./data/svn/*                                         # 各 repo 大小
```

## Log

```
journalctl -t svn-server -f -o cat                                       # 即時看
journalctl -t svn-server -o cat --since today | grep '^svn-op'           # 今天的 SVN 操作
journalctl -t svn-server -o cat --since "2026-11-01" --until "2026-11-08" | grep '^svn-op'
```

| 內容 | 怎麼找 |
|---|---|
| SVN 客戶端操作(checkout、commit、update、log…,含 TortoiseSVN) | `grep '^svn-op'`,格式:`svn-op 時間 來源IP 帳號 repo 動作` |
| 某人的操作 | `grep '^svn-op' \| grep ' <帳號> '` |
| 某 repo 的 commit | `grep '^svn-op' \| grep ' <repo> commit'` |
| 瀏覽器瀏覽 / 下載 | `grep 'GET /svn/'`(Apache access log 格式，含帳號與狀態碼) |
| iF.SVNAdmin 網頁操作 | `grep '/svnadmin/'` |
| 登入失敗 | `grep 'authentication failure'` |
| Apache 錯誤 | `journalctl -t svn-server -p err --since today` |

access log 裡的中文路徑是網址編碼(`%e4%b8%8d...`),不是亂碼。要看中文:
```
journalctl -t svn-server -o cat --since today | grep 'GET /svn/' | python3 -c "import sys,urllib.parse;[print(urllib.parse.unquote(l),end='') for l in sys.stdin]"
```

容量:
```
journalctl --disk-usage      # journal 佔用(預設上限為檔案系統 10%,最多 4G,滿了自動刪最舊)
```
`docker logs svn-server` 也可用，但只看得到目前這個容器(升級前的紀錄要用 journalctl)。

## 備份 / 還原

整台 VM 有備份時通常不需另外處理。要單獨備份某個 repo(正式機運作中也安全):
```
mkdir -p ./backup
docker run --rm -v "$PWD/data/svn":/home/svn:ro -v "$PWD/backup":/backup --entrypoint svnadmin \
  ghcr.io/talisych/svn-server:<版本> hotcopy /home/svn/<名稱> /backup/<名稱>-$(date +%Y%m%d)
```
還原：停止服務，把備份資料夾放回 `./data/svn/<名稱>`,`docker exec svn-server chown -R apache:apache /home/svn/<名稱>` 後啟動。

設定檔備份:`cp -a ./data/svn_config ./data/svnadmin_config ./backup/`

## 常見問題

| 狀況 | 處理 |
|---|---|
| commit 出現 `Permission denied`(`txn-current-lock`) | repo 不屬於 apache,執行上面的 `chown` |
| 網頁顯示「The form has expired or is invalid」 | 頁面開太久或在別的分頁重新登入過，重新整理再送出 |
| 從 email / 其他網站點連結進 svnadmin 要重新登入 | 正常(cookie `SameSite=Strict`,防 CSRF) |
| 帳號 svn 能用、svnadmin 登不進去 | 密碼是 bcrypt,用 `htpasswd -bm` 重設 |
| `svn://` 連不上 | 已移除，改用 `http://<主機>/svn/<repo>`;舊 working copy 用 `svn relocate` |
| 書籤 `usergroupassign.php` 404 | 已改名 `groupassign.php`,從選單進入 |

## 發版(維護者)

```
git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z
```
GitHub Actions 會 build、smoke test、推 `ghcr.io/talisych/svn-server:X.Y.Z` 與 `latest`、建立 Release。

iF.SVNAdmin 以 submodule 引用 [talisych/iF.SVNAdmin](https://github.com/talisych/iF.SVNAdmin),修改在 fork 進行，再更新指標:
```
git -C iF.SVNAdmin fetch && git -C iF.SVNAdmin checkout <commit>
git add iF.SVNAdmin && git commit -m "Update iF.SVNAdmin to <commit>"
```
