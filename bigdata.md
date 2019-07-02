# 云计算大数据部分
1. xueqing-serve项目解读
   > 该项目是比赛中需要编写服务代码的项目，作用是对整个大数据得分模块功能数据的操作。主要操作有对数据的爬取，对数据的清戏，对数据中岗位的聚类，对岗位的分类推荐

2. 爬虫
   >前提条件 已经导入mongodb数据库，habse数据库，启动了大数据所需服务(mongodb,hadoop,zooker,hbase)，这些部分在所给资源包中有说明，请自行配置启动

    * 修改配置文件
    ![config.gif](/pic/config.gif)
    **修改该config文件下的需要配置的配置文件(修改ip地址,使用hbase虚拟机的ip地址,可通过在虚拟机中ipconfig指令来查询)**

    -------------------------------------------------------------------------------------------------------------
    * 编写爬虫代码
    ```java
    	/**
	 * 爬虫单个页面的处理方法
	 */
	@Override
	public void process(Page page) {
		// Init select and urls
		Selectable select = null;
		List<String> urls = null;
		// 列表页面，这里是通过搜索来确定
		try {
			if (!page.getUrl().toString().contains("html")) {
				// 列表区域
				select = page.getHtml().xpath("//p[@class='PositionName']");
				urls = select.links().all();
				page.addTargetRequests(urls);
				System.out.println(page);
				// 分页区域
				select = page.getHtml().xpath("//ul[@id='pagination']");
				urls = select.links().all();
				// 遍历是否存在了
				Iterator<String> it = urls.iterator();

				while (it.hasNext()) {
					String x = it.next();
					if (x.equals(breakUrl)) {
						it.remove();
					}
				}
				// 收集下一级的url
				page.addTargetRequests(urls);
			}
			// 岗位页面
			else{
				// 检查本页面是否爬过？岗位ID
				Matcher matcher = pattern.matcher(page.getUrl().toString());
				String pageID = null;
				while (matcher.find()) {
					pageID = matcher.group(1);
				}
				// 如果没有ID，不处理
				if (pageID == null) {
					return;
				}
				Map<String, Object> map = new HashMap<>();
				String JsonContext = ReadFile
						.ReadFile(System.getProperty("user.dir") + "/configuration/job_config.json");
				JSONObject jsonObject = new JSONObject(JsonContext);
				JSONArray wbsites = jsonObject.getJSONArray("wbsites");
				for (int i = 0; i < wbsites.length(); i++) {
					JSONObject wbsite = wbsites.getJSONObject(i);
					String wbsitename = wbsite.getString("wbsitename");
						// 设置来源
						map.put("resource", wbsitename);
						JSONArray fields = wbsite.getJSONArray("fields");
						for (int j = 0; j < fields.length(); j++) {
							JSONObject field = fields.getJSONObject(j);
							String chinesename = field.getString("chinesename");// 标签的中文名称
							String name = field.getString("name");// 英文名称
							String path = field.getString("path");// 爬虫网页标签路径
							// ?
							if (path.startsWith("//")) {
								
								String objectStr = Format_transform.gb2312ToUtf8(page.getHtml().xpath(path).toString());// 性能很差
								map.put(name, Format_transform.change(objectStr));
							} else {
								String objectStr = Format_transform.gb2312ToUtf8(page.getHtml().regex(path).toString());
								logger.info(chinesename + ":" + objectStr);
								map.put(name, Format_transform.change(objectStr));
							}
							// ?
							if (name.equals("companymess")) {
								String companymess = Format_transform
										.gb2312ToUtf8(page.getHtml().xpath(path).toString());
								String[] strs = UtilTools.parseCompony(companymess);
								map.put("nature", Format_transform.change(strs[0]));
								map.put("scale", Format_transform.change(strs[1]));
								map.put("industry", Format_transform.change(strs[2]));
							}
						}
				}
				// 设置url
				map.put("url", page.getUrl().toString());

				// 保存到HBase中，并设置结束日期为明天    && isExist == false
				if (pageID != null ) {
					map.put("id", pageID);
					// HDFS存储位置
					map.put(HDFS, job_rawdata_path + pageID + Suffix);
					// 插入数据
					jobDataReposity.insertData("job_internet", map);
					// 下载文件，存入HDFS
					saveToHdfs(page.getUrl().toString(), job_rawdata_path);
				} else {
					// 找到这个岗位，设置它的结束日期为今天+1，每个岗位的开始日期和结束日期，目的为了统计持续周期
					jobDataReposity.insertEndTime("job_internet", map);
				}
			} 
		} catch (Exception exp) {
			logger.error(exp.toString());
		}
	}

	/**
	 * 保存数据到hdfs
	 * 
	 * @param urlname
	 * @throws IOException
	 */
	private void saveToHdfs(String urlname, String savepath) throws IOException {
		URL url = new URL(urlname);
		Pattern pattern1 = Pattern.compile("/([0-9]+)\\.html");
		Matcher matcher1 = pattern1.matcher(urlname);
		String num = null;
		while (matcher1.find()) {
			num = matcher1.group(1);
		}
		FileOutputStream fos = null;
		InputStream is;
		is = url.openStream();
		HdfsClient hdfsClient = HdfsClient.getInstance();
		HdfsClient.uploadByIo(is, savepath + num + ".html");	
	}
    ```
    
    **这两个方法在WYjobPageCrawler.java下。具体含义自行参照[webmagic](https://github.com/code4craft/webmagic)框架**
    ----------------------------------------------------------------------------------
    * 启动爬虫服务
        - 在启动服务前应该先开启我们爬虫对象的站点

            在tomcat中加载xueqing-web的war包或者在eclipse中import xueqing-web的项目然后启动tomcat运行项目
        - 启动爬虫服务

            ```java
                @Override
	            public ServiceState start() {

                //1. 启动收集服务（虫）
                Service jobCollector = new JobCollectService(server, this);
                jobCollector.start();
                // 2. 启动清洗服务（清洗）
                //Service jobCleaner = new JobCleanService(server, this);
                //jobCleaner.start();
                // 3. 启动分析服务（聚类）
                //Service jobCluster = new JobClusterService(server, this);
                //jobCluster.start();
                return ServiceState.STATE_RUNNING;
            }
            ```
            **JobAnalysisService.java中只解除启动收集服务的注解**
            **然后运行EduInsightServer.java启动服务**

3. 数据清洗 

	>主要是通过JobCleanService.java process()方法和完成JobCleanUtils代码来完成清洗

	* 书写process方法
		```java
			/**
		* 具体的实现。
		*/
		public synchronized ServiceState process() {
			try {
				isTaskDone.set(false);
				// 清洗数据
				logger.info("清洗数据开始");
				//清洗所有互联网信息			
				jobDataReposity.cleanJobData();
				//进行岗位分类保存
				String table = hbaseclassify.getProperty("hbasetable");//分类
				String[] strArray = null;
				strArray = table.split(",");
				String JsonContext = ReadFile
						.ReadFile(System.getProperty("user.dir") + "/configuration/hbaseclassify.json");
				JSONObject jsonObject;
				jsonObject = new JSONObject(JsonContext);
				//如何根据名称先分大类
				for (int i = 0; i < strArray.length; i++) {
					JSONArray jsonArray = jsonObject.getJSONArray(strArray[i]);
					//对保存的数据进行分类统计IT
					//	[{"name":"云计算"},{"name":"cloud"},{"name":"openstack"},{"name":"kvm"},{"name":"vmware"},{"name":"ceph"},{"name":"sdn"},{"name":"云"},{"name":"阿里云"},{"name":"腾讯云"},{"name":"云存储"}]
					jobDataReposity.classify(jsonArray, "job_" + strArray[i], strArray[i]);
				}
			
			} catch (Exception e) {
				logger.error(e.toString());
			}
			finally {
				isTaskDone.set(true);
			}
			logger.info("清洗数据结束");
			return ServiceState.STATE_RUNNING;
		}
		```
		**该方法是对具体的数据进行分类**
	* 书写cleanJobData方法

		```java
		/** 
		*初中,高中,中技,中专,大专,本科,硕士,博士
		* @Title: ${cleanEducation} 
		* @Description: ${清洗工作学历}
		* @param ${scale：爬取的工作学历}  
		* @return ${清洗过的工作学历}
		* @throws 
		*/
		public static String cleanEducation(String education) {//清洗学历 至初中,高中,中技,中专,大专,本科,硕士,博士,其他 其一
			if (education.contains("初中")) {
				return "初中";
			}
			else if(education.contains("高中")) {
				return "高中";
			}
			else if(education.contains("中专")) {
				return "中专";
			}
			else if(education.contains("中技")) {
				return "中技";
			}
			else if(education.contains("大专")) {
				return "大专";
			}
			else if(education.contains("本科")) {
				return "本科";
			}
			else if(education.contains("硕士")) {
				return "硕士";
			}
			else if(education.contains("博士")) {
				return "博士";
			}
			return "不限";
		}
		```
		**该方法是具体对学历的清洗操作**
	* 启动服务
  
		```java
		@Override
		public ServiceState start() {

			//1. 启动收集服务（虫）
			//Service jobCollector = new JobCollectService(server, this);
			//jobCollector.start();
			// 2. 启动清洗服务（清洗）
			Service jobCleaner = new JobCleanService(server, this);
			jobCleaner.start();
			// 3. 启动分析服务（聚类）
			//Service jobCluster = new JobClusterService(server, this);
			//jobCluster.start();
			return ServiceState.STATE_RUNNING;
		}
		```
	**JobAnalysisService.java中只解除启动清洗服务的注解**
	**然后运行EduInsightServer.java启动服务**



4. 岗位聚类
   >岗位聚类主要有两部分第一部分是对学历和地区信息数据的处理还有一部分是对岗位数据的处理聚类
   * 第一部分
	- 添加HbaseStatistics.java中代码
	```java	
		/**
		* 学历分布
		* 
		* @param stmt
		* @param tableName
		* @throws SQLException
		* @throws IOException
		*/
		public void countEducationDistribution(String tableName, String mongoTable, String family) throws SQLException, IOException {/////
			MongoClient mongoClient = mongodbstorage.setUp();
			// String sql = "select LOCATION,count(1) as count,sum(AMOUNT) from " +
			// tableName
			// + " where ISPERCEPTED = 'no' group by LOCATION order by count desc";
			// ResultSet results = stmt.executeQuery(sql);

			// 建立表的连接
			Table table = connection.getTable(TableName.valueOf(tableName));
			// 创建一个空的Scan实例
			Scan scan1 = new Scan();
			// 可以指定具体的列族列
			scan1.addColumn(Bytes.toBytes(family), Bytes.toBytes("EDUCATION")).addColumn(Bytes.toBytes(family),
					Bytes.toBytes("AMOUNT"));
			scan1.setCaching(60);
			scan1.setMaxResultSize(1 * 1024 * 1024); // 100k （MB1 * 1024 * 1024）
			scan1.setFilter(new PageFilter(1000));

			// 在行上获取遍历器
			ResultScanner scanner1 = table.getScanner(scan1);

			Map map = mongodbstorage.create(mongoTable, "job_education_distribution", mongoClient);// mongodb集合,key,value
			MongoCollection<Document> collection = (MongoCollection<Document>) map.get("collection");
			Document document = (Document) map.get("document");
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			java.util.Date date = new java.util.Date();
			String da = sdf.format(date);
			mongodbstorage.appendString(document, "date", da);
			int[] cu = new int[34];
			int[] nu = new int[34];
			Vector<Document> vec = new Vector<Document>();
			for (Result res : scanner1) {
				String loc = (new String(CellUtil.cloneValue(res.rawCells()[1]))).split("-")[0];
				int total = Integer.parseInt(new String(CellUtil.cloneValue(res.rawCells()[0])));
				int num = 1;
				if (!cities.containsKey(loc)) {
				} else {
					// String prov = jobanalysisreposity.appendProvinceDistribution(document, loc,
					// total, mongoClient);
					String prov = cities.get(loc);
					switch (prov) {
					case "初中":
						cu[0] += total;
						nu[0] += num;
						break;
					case "高中":
						cu[1] += total;
						nu[1] += num;
						break;
					case "中技":
						cu[2] += total;
						nu[2] += num;
						break;
					case "中专":
						cu[3] += total;
						nu[3] += num;
						break;
					case "大专":
						cu[4] += total;
						nu[4] += num;
						break;
					case "本科":
						cu[5] += total;
						nu[5] += num;
						break;
					case "硕士":
						cu[6] += total;
						nu[6] += num;
						break;
					case "博士":
						cu[7] += total;
						nu[7] += num;
						break;
					default: 
						cu[8] += total;
						nu[8] += num;
						break;

					}
				}
				logger.info(loc + ":" + total);
			}
			Document[] doc = new Document[34];
			for (int i = 0; i <= 33; i++) {
				doc[i] = new Document();
			}
			mongodbstorage.appendExperience(doc[0], "初中", cu[0], nu[0]);
			mongodbstorage.appendExperience(doc[1], "高中", cu[1], nu[1]);
			mongodbstorage.appendExperience(doc[2], "中技", cu[2], nu[2]);
			mongodbstorage.appendExperience(doc[3], "中专", cu[3], nu[3]);
			mongodbstorage.appendExperience(doc[4], "大专", cu[4], nu[4]);
			mongodbstorage.appendExperience(doc[5], "本科", cu[5], nu[5]);
			mongodbstorage.appendExperience(doc[6], "硕士", cu[6], nu[6]);
			mongodbstorage.appendExperience(doc[7], "博士", cu[7], nu[7]);
			mongodbstorage.appendExperience(doc[8], "不限", cu[8], nu[8]);

			for (int i = 0; i <= 8; i++) {
				vec.add(doc[i]);
			}
			mongodbstorage.appendArray(document, "category", vec);
			mongodbstorage.insertOne(collection, document);	
		}
	```
   * 第二部分

	- 添加JobClusterService.java中代码

		```java
			public ServiceState process() {/////
				try {
					isTaskDone.set(false);
					// 链接存储和Hive，用于聚类分析。
		//			Statement stmt = HiveOperator.getInstance().setUp();
					mongoClient = mongodbstorage.setUp();
					// 1. 读取配置
					// 2 启动聚类，存到MongoDB分析库
					List<Map<String, Object>> lists = new ArrayList<>();
					JobCluster cluster = new JobCluster();
					String JsonContext = ReadFile
							.ReadFile(System.getProperty("user.dir") + "/configuration/job_industry_skills_config.json");
					// json to Object
					JSONObject jsonObject = new JSONObject(JsonContext);
					JSONArray jsonArray = jsonObject.getJSONArray("industry");
					for (int i = 0; i < jsonArray.length(); i++) {
						Map<String, Object> map = new HashMap<String, Object>();
						JSONObject object = jsonArray.getJSONObject(i);
						// 获取name（云计算、大数据、ai）
						String name = object.getString("industry-id");
						String[] names = name.split("_");
						map.put("industry-id", names[1]);
						// 获取小类
						String category = object.getString("category");
						String[] categorys_ = category.split(",");
						List<String> categorys = new ArrayList<>();
						for (String str : categorys_) {
							categorys.add(str);
						}
						map.put("category", categorys);
						// 获取技能名称
						String skill = object.getString("skills");
						String[] skills_ = skill.split(",");
						List<String> skills = new ArrayList<>();
						for (String str : skills_) {
							skills.add(str);
						}
						map.put("skills", skills);
						// 加入map list
						lists.add(map);
					}
					
					// 岗位聚类
					for (Map<String, Object> map : lists) {
						String str = (String) map.get("industry-id");
						for (String str1 : (List<String>) map.get("category"))// 岗位子分类
						{
							for (int i = 0; i <= clusterCount.length-1; i++) {
								cluster.cluster(str, str1, (List<String>) map.get("skills"),clusterCount[i], mongoClient);
							}
						}
					}
					
					//数据统计
					HbaseStatistics hbaseStatics =new HbaseStatistics();
					hbaseStatics.doDataStatistics();

				} catch (Exception e) {
					logger.error(e.toString());
				} finally {
					isTaskDone.set(true);
				}
				return ServiceState.STATE_RUNNING;
			}

		```
	- 添加JobCluster.java中代码
		```java
			public List<Integer> kMeans(List<double[]> des1, int cluster) {
				String masterName = sparkProperties.getProperty("spark_master");
				SparkConf conf = new SparkConf().setAppName("JobCluseter").setMaster(masterName);
				JavaSparkContext jsc = new JavaSparkContext(conf);
				List<Vector> preData = new ArrayList<>();
				for (int i = 0; i < des1.size(); i++) {
					Vector sv = Vectors.dense(des1.get(i));
					preData.add(sv);
				}
				JavaRDD<Vector> data = jsc.parallelize(preData);
				int numClusters = cluster;
				int numIterations = 5000;
				int runs = 300;
				KMeansModel clusters = KMeans.train(data.rdd(), numClusters, numIterations, runs);
				JavaRDD<Integer> clusterResult = clusters.predict(data);
				List<Integer> indexs = new ArrayList<>();
				Vector[] centers = clusters.clusterCenters();
				clusterResult.collect();
				for (int i = 0; i < clusterResult.collect().size(); i++) {
					indexs.add(clusterResult.collect().get(i));
				}
				jsc.close();

				return indexs;
			}
		```
	- 启动服务
		```java
		@Override
		public ServiceState start() {

			//1. 启动收集服务（虫）
			//Service jobCollector = new JobCollectService(server, this);
			//jobCollector.start();
			// 2. 启动清洗服务（清洗）
			Service jobCleaner = new JobCleanService(server, this);
			jobCleaner.start();
			// 3. 启动分析服务（聚类）
			//Service jobCluster = new JobClusterService(server, this);
			//jobCluster.start();
			return ServiceState.STATE_RUNNING;
		}
		```
	**JobAnalysisService.java中只解除启动清洗服务的注解**
	**然后运行EduInsightServer.java启动服务**
--------------------------
>以上2,3,4部分的题目数据已经完成，然后需要运行xueqing-client项目来显示效果(用户名:admin,密码:123456)
>需要注解掉xueqing-client项目中controller层指定请求控制代码中的一些从mongodb获取数据的操作,具体请自行分析项目源码
1. 岗位推荐

