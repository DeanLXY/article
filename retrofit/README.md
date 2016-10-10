# Retrofit #
* 主页: [https://github.com/square/retrofit](https://github.com/square/retrofit)
* 注意: 使用Retrofit的前提是**服务器端代码遵循REST规范 !!!!!**
* 功能: 
	* 效率非常高
	* 可以直接将结果转换称Java类
	* 主要是配合RxJava一起使用
* 配置: 
	* 添加Retrofit依赖: compile 'com.squareup.retrofit2:retrofit:2.0.2'
	* 添加数据解析依赖,根据实际情况进行选择
		* Gson : com.squareup.retrofit2:converter-gson:2.0.2
		* Jackson : com.squareup.retrofit2:converter-jackson:2.0.2
		* Moshi : com.squareup.retrofit2:converter-moshi:2.0.2
		* Protobuf : com.squareup.retrofit2:converter-protobuf:2.0.2
		* Wire : com.squareup.retrofit2:converter-wire:2.0.2
		* Simple XML : com.squareup.retrofit2:converter-simplexml:2.0.2

* 使用步骤: 
	1. 利用[http://www.jsonschema2pojo.org/](http://www.jsonschema2pojo.org/)创建数据模型
	2. 创建REST API 接口
		* 常用注解:
			* 请求方法:@GET / @POST / @PUT / @DELETE / @HEAD
			* URL处理
				* @Path - 替换参数
				
						@GET("/group/{id}/users")
						public Call<List<User>> groupList(@Path("id") int groupId);
				* @Query - 添加查询参数
				
						@GET("/group/{id}/users")
						public Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);
				* @QueryMap - 如果有多个查询参数，把它们放在Map中

						@GET("/group/{id}/users")
						public Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
		* 示例代码:

				public interface NetAPI {
				    @GET("/users/{user}")
				    public Call<GitModel> getFeed(@Path("user") String user);
				
				    @GET("/service/getIpInfo.php")
				    public Call<IPModel> getWeather(@Query("city")String city);
				}
	3. 创建Retrofit对象, 并发起请求.示例代码:

	        // 构建Retrofit实例
	        Retrofit retrofit = new Retrofit.Builder().
	                baseUrl(API2).
	                addConverterFactory(GsonConverterFactory.create()).
	                build();
	        // 构建接口的实现类
	        IpAPI weatherAPI = retrofit.create(IpAPI.class);
	        // 调用接口定义的方法
	        Call<IPModel> weatherCall = weatherAPI.getWeather("8.8.8.8");
	        // 异步执行请求
	        weatherCall.enqueue(new Callback<IPModel>() {
	            @Override
	            public void onResponse(Call<IPModel> call, Response<IPModel> response) {
	                IPModel model = response.body();
	                System.out.println("country:" + model.getData().getCountry());
	            }
	
	            @Override
	            public void onFailure(Call<IPModel> call, Throwable t) {
	                System.out.println(t.toString());
	            }
	        });