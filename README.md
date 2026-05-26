# Cordova Spatialite Storage

This is a project fork of `cordova-spatialite-storage`, based on the
`litehelpers/Cordova-sqlite-storage` API style, with Spatialite support.

此版 iOS 另外包含 `sqlite3-base64`、`sqlite3-regexp-cached`，並會在開啟 DB
後初始化 Spatialite，所以可以用 SQLite、Spatialite、regexp、base64 相關 SQL。

## 目前專案使用情境

這個 plugin 主要用來開啟 app 已下載好的 `.sqlite` / `.db` 檔案，例如離線圖資、
地籍圖資、KML/ZIP 下載後產生的資料庫。它不是 Web SQL，也不是瀏覽器的
`window.openDatabase`；請一律透過：

```js
window.sqlitePlugin.openDatabase(options, successCallback, errorCallback);
```

## 安裝

從本專案的 plugin 原始碼安裝：

```sh
cordova plugin add ./myplugins/Cordova-spatialite-storage
```

如果專案已經裝過舊版，建議先移除再加入，讓 `plugins/` 和 `platforms/` 都重新產生：

```sh
cordova plugin rm cordova-spatialite-storage
cordova plugin add ./myplugins/Cordova-spatialite-storage
cordova prepare ios
cordova prepare android
```

iOS 有改到 `src/ios/SQLitePlugin.m` 或 common C header 時，一定要重新 Xcode build
並安裝到手機；只更新 `www` 不會重新編譯 native plugin。

## 基本 API

### 開啟 DB

```js
var db = window.sqlitePlugin.openDatabase(
    {
        name: dbPath,
        location: "default"
    },
    function () {
        console.log("DB opened");
    },
    function (err) {
        console.error("DB open failed", err);
    }
);
```

`name` 可以是 DB 檔案路徑。這個 fork 的 Android 和 iOS 對路徑格式要求不同：

- Android：建議傳入 raw absolute path，例如 `/storage/emulated/0/slopesur/B01/B01.sqlite`
- iOS：可以傳入 `file://...` 或 raw absolute path，例如 `/var/mobile/.../Library/NoCloud/slopesur/pdaDB.sqlite`
- iOS 目前會處理 percent encoding，所以中文 app name 或中文路徑可以正常 decode

如果 `name` 只是一個相對檔名，plugin 會依照 `location` 做舊式 Cordova SQLite
位置處理；但本專案下載 DB 幾乎都是 full path，建議用 full path 來避免平台差異。

### Transaction 查詢

```js
db.readTransaction(function (tx) {
    tx.executeSql(
        "SELECT spatialite_version() AS version",
        [],
        function (tx, res) {
            console.log(res.rows.item(0).version);
        },
        function (tx, err) {
            console.error("SQL failed", err);
            return false;
        }
    );
});
```

寫入資料請用 `transaction`：

```js
db.transaction(function (tx) {
    tx.executeSql(
        "INSERT INTO log_table(message) VALUES (?)",
        ["hello"],
        function () {
            console.log("insert ok");
        }
    );
});
```

### 關閉 DB

```js
db.close(
    function () {
        console.log("DB closed");
    },
    function (err) {
        console.error("DB close failed", err);
    }
);
```

不要在 transaction 還在跑的時候 close，plugin 會回傳錯誤。

## 建議的跨平台開啟方式

本專案常見情境是 Cordova File 或 FileTransfer 拿到 `fileEntry.nativeURL` / `toURL()`。
建議在 app 層統一轉成 plugin 能接受的格式：

```js
function normalizeSpatialitePathForOpen(path) {
    var value = String(path || "").replace(/\\/g, "/");

    if (typeof getDeviceKind === "function" && getDeviceKind() === "ANDROID") {
        return value.replace(/^file:\/\//i, "");
    }

    if (typeof getDeviceKind === "function" && getDeviceKind() === "IOS") {
        if (value.charAt(0) === "/" && value.indexOf("file://") !== 0) {
            return "file://" + value;
        }
    }

    return value;
}

function openSpatialiteDatabase(path, onOpen, onError) {
    return window.sqlitePlugin.openDatabase(
        {
            name: normalizeSpatialitePathForOpen(path),
            location: "default"
        },
        onOpen,
        onError
    );
}
```

Android 端不要把 `file://` 直接傳進 native open；iOS 端這次改版後 `file://`
和 raw absolute path 都可支援。

## 地籍 DB 使用範例

地籍資料通常是下載完成後才存在手機內，因此流程要分成三步：

1. 先下載 DB 檔案到 app storage，例如 iOS 的 `Library/NoCloud/slopesur/...`
2. 確認檔案存在後，再用 `window.sqlitePlugin.openDatabase`
3. 保留 DB handle，後續地籍查詢直接走 SQLite transaction

範例：

```js
var cadasterDbs = {};

function openCadasterDb(code, fileUrl, done) {
    var db = openSpatialiteDatabase(
        fileUrl,
        function () {
            cadasterDbs[code] = db;
            done(null, db);
        },
        function (err) {
            done(err);
        }
    );
}

function queryCadaster(code, sql, params, done) {
    var db = cadasterDbs[code];
    if (!db) {
        done(new Error("Cadaster DB is not opened: " + code));
        return;
    }

    db.readTransaction(function (tx) {
        tx.executeSql(sql, params || [], function (tx, res) {
            var rows = [];
            for (var i = 0; i < res.rows.length; i++) {
                rows.push(res.rows.item(i));
            }
            done(null, rows);
        }, function (tx, err) {
            done(err);
            return false;
        });
    });
}
```

如果不知道地籍 DB 的實際 table / geometry 欄位，可以先查 `geometry_columns`：

```js
db.readTransaction(function (tx) {
    tx.executeSql(
        "SELECT f_table_name, f_geometry_column FROM geometry_columns LIMIT 1",
        [],
        function (tx, res) {
            var row = res.rows.item(0);
            console.log(row.f_table_name, row.f_geometry_column);
        }
    );
});
```

常用的 Spatialite 查詢：

```sql
SELECT AsText(Geometry) AS wkt
FROM B
WHERE town_id = ?
```

不同地籍 DB 的實際 table name 不一定叫 `B`，請以 `geometry_columns` 查到的
`f_table_name` 為準。

## iOS 下載 DB 的注意事項

iOS 下載後的檔案路徑常會長這樣：

```text
file:///var/mobile/Containers/Data/Application/<UUID>/Library/NoCloud/slopesur/pdaDB.sqlite
```

或 raw path：

```text
/var/mobile/Containers/Data/Application/<UUID>/Library/NoCloud/slopesur/pdaDB.sqlite
```

這次改版後 iOS native open 會：

- 接受 `file://` URL
- 接受 raw absolute path
- 對 `%E5%B1%B1...` 這類 percent encoded 中文路徑做 decode
- 使用 `fileSystemRepresentation` 交給 SQLite，避免中文路徑編碼問題
- 如果 DB 不存在，回傳 error callback，而不是 silent return

所以 app 層要確保「下載完成且檔案存在」後再 open DB。若 console 看到：

```text
Unable to find database
new transaction is waiting for open operation
```

通常代表 DB path 錯、下載還沒完成、或 app storage 位置和 open 時的位置不同。

## 這次改版內容

### `src/ios/SQLitePlugin.m`

本次 iOS DB open 修正：

- 原本只用 `NSURL URLWithString(...).path`，遇到 raw absolute path 可能失敗
- 現在先判斷 `file://`，只有 URL path 才走 `NSURL`
- 支援 percent decoding，修正中文路徑 / 中文 app name 問題
- `sqlite3_open` 改用 `fileSystemRepresentation`
- 找不到 DB 時會送出 Cordova error result，不會直接 `return`
- open 失敗或 `sqlite_master` 檢查失敗會 close DB handle
- 只有 DB 真正 open 成功後才初始化 Spatialite connection

這些修正主要影響 iOS；Android native open 沒有改。

### `src/common/spatialite/gg_advanced.h`

修正一個 C header typo：

```diff
- gaiaTriangularGrid (gaiaGeomCollPtr geom,,
+ gaiaTriangularGrid (gaiaGeomCollPtr geom,
```

這個 typo 會造成新版 Xcode/clang build fail。它是 common header，修正後對
Android 也安全。

## Android 相容性

這次 native 行為改動主要在 iOS `.m`。Android 目錄沒有改。

Android 使用時請注意：

- `name` 建議傳 raw absolute path，不要帶 `file://`
- 如果檔案不存在，Android 目前會嘗試建立空 DB
- 如果 app 要共用 iOS 的下載流程，請確認 Android 端也有安裝需要的下載 plugin
  或保留 Android 原本下載機制

## iOS build / 更新 plugin 注意事項

如果只是 app 的 `www/js` 改動，更新 www 後重新 build/install app 即可。

如果改到這個 plugin 的 native 檔案，例如：

- `src/ios/SQLitePlugin.m`
- `src/common/spatialite/*.h`
- `src/ios/native/*.a`

需要讓 Cordova 重新把 plugin 複製到 platform，再用 Xcode build：

```sh
cordova plugin rm cordova-spatialite-storage
cordova plugin add ./myplugins/Cordova-spatialite-storage
cordova prepare ios
```

在本專案也可以用 Xcode workspace build：

```sh
xcodebuild \
  -workspace platforms/ios/App.xcworkspace \
  -scheme App \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  build
```

實機測試時，請確認手機上的 app 已重新安裝，否則舊的 native plugin 仍會留在舊 app bundle。

## Troubleshooting

### `Unable to find database`

確認：

- DB 是否真的下載完成
- open 時的 path 是否和下載 target 一樣
- iOS 是否傳入 `file://...` 或 absolute path
- Android 是否拿掉 `file://`

### `new transaction is waiting for open operation`

通常是 open 失敗後仍然排入 transaction。請一定要加 open error callback，
並在 error callback 中停止後續 SQL。

### iOS build 找不到 `libarclite`

這不是 plugin API 問題，是 Xcode SDK / deployment target 問題。提高 iOS
minimum deployment target 後重新 build。

### Xcode 編譯 `gg_advanced.h` 失敗

確認 `src/common/spatialite/gg_advanced.h` 裡沒有 `geom,,`。本 fork 已修正。
