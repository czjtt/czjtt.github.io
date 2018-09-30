## springboot devtools 热部署 出现的问题
遇到一个部分jar包出现  

````
“***/lib must exist and must be a directory”
````
````
“***/bin must exist and must be a directory”
````

文件夹缺少的问题。

> 本人遇到的是openoffice的jurt和juh两个jar包出现了问题。

这是由于devtools是扫描工程中的所有依赖包，然后读取jar包的META-INF文件夹下的MANIFEST.MF文件中的Class-Path内容，
过滤掉部分内容
以下是代码：

ChangeableUrls类下的


	private ChangeableUrls(URL... urls) {
		DevToolsSettings settings = DevToolsSettings.get();
		List<URL> reloadableUrls = new ArrayList<URL>(urls.length);
		for (URL url : urls) {
			if ((settings.isRestartInclude(url) || isFolderUrl(url.toString()))
					&& !settings.isRestartExclude(url)) {
				reloadableUrls.add(url);
			}
		}
		this.urls = Collections.unmodifiableList(reloadableUrls);
	}


然后返回的urls数据，当然会在代码中进行assert
FileSystemWatcher类下

	public void addSourceFolder(File folder) {
		Assert.notNull(folder, "Folder must not be null");
		Assert.isTrue(folder.isDirectory(),
				"Folder '" + folder + "' must exist and must" + " be a directory");
		synchronized (this.monitor) {
			checkNotStarted();
			this.folders.put(folder, null);
		}
	}

第二行的Assert.isTrue。

## 解决方法


* 重新打包一份jar

	1.修改springboot-devtools的jar包的源代码

	2.修改出现问题的jar包的源代码中的META-INF文件夹下的MANIFEST.MF文件中的Class-Path里不存在的路径的值

* 按照错误提示，新建几个空文件夹

两种方法当然是第二种简单有效入侵最少了。
