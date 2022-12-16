# Spatialite extension fork
This is a fork of the https://github.com/litehelpers/Cordova-sqlite-storage project.

此版在 ios 加入 sqlite3-base64、sqlite3-regexp-cached ( From : cordova-sqlite-ext )

Usage:

switch (getDeviceKind()) {
	case "ANDROID":
		{
			var dbpath = str_replace("file://", "", cordova.file.externalRootDirectory) + '/' + window['APP_PATH'] + '/' + osmData + '/' + osmData + '.db';
			dbpath = str_replace("//", "/", dbpath);
			dbpath = str_replace("//", "/", dbpath);
			//console.log(dbpath);
			var op = {
				name: dbpath,
				location: 'default'
			};
			dbObj = window.sqlitePlugin.openDatabase(op);
		}
		break;
	case "IOS":
		{
			var dbpath = cordova.file.documentsDirectory + '/' + window['APP_PATH'] + '/' + osmData + '/' + osmData + '.db';
			dbpath = str_replace("//", "/", dbpath);
			dbpath = str_replace("//", "/", dbpath);
			//console.log(dbpath);
			var op = {
				name: dbpath,
				location: 2,
				createFromLocation: 1
			};
			dbObj = window.sqlitePlugin.openDatabase(op);
		}
		break;
}
