# Hello World

Program

```c#
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using System.IO;

namespace Hello
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    var configurationBuilder = new ConfigurationBuilder()
                        .SetBasePath(Directory.GetCurrentDirectory())
                        .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                        .AddJsonFile("appsettings.Development.json", true, false)
                        .AddJsonFile("appsettings.Production.json", true, false);

                    var hostingConfig = configurationBuilder.Build();
                    //var urls = hostingConfig["serverAddress"].Split(',');

                    webBuilder.UseContentRoot(Directory.GetCurrentDirectory())
                        .UseConfiguration(hostingConfig)
                        //.UseUrls(urls)
                        .UseStartup<Startup>();
                });
    }
}
```

Startup

```c#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.DataProtection;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using StackExchange.Redis;
using System;

namespace Hello
{
    public class Startup
    {
        public IConfiguration Configuration { get; }

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        // This method gets called by the runtime. Use this method to add services to the container.
        // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
        public void ConfigureServices(IServiceCollection services)
        {
            services.Configure<CookiePolicyOptions>(options =>
            {
                // This lambda determines whether user consent for non-essential cookies is needed for a given request.
                options.CheckConsentNeeded = context => true;
                options.MinimumSameSitePolicy = SameSiteMode.None;
            });
            // ��ʹ��session֮ǰҪע��cacheing����Ϊsession������cache���д洢
            services.AddDistributedMemoryCache();

            #region ʹ��Redis����Session
            var redisConn = Configuration["Redis:Connection"];
            var redisInstanceName = Configuration["Redis:InstanceName"];
            //Session ����ʱ������
            var sessionOutTime = Configuration.GetValue("Session:TimeOut", 30);

            var redis = ConnectionMultiplexer.Connect(redisConn);
            services.AddDataProtection().PersistKeysToStackExchangeRedis(redis, "DataProtection-Test-Keys");
            //services.AddDistributedRedisCache(option =>
            //{
            //    //redis �����ַ���
            //    option.Configuration = redisConn;
            //    //redis ʵ����
            //    option.InstanceName = redisInstanceName;
            //});
            #endregion

            // session ����
            services.AddSession(options =>
            {
                // �ٷ��ĵ� https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/app-state
                // ���� Session ����ʱ��
                options.IdleTimeout = TimeSpan.FromDays(90);
                //options.CookieHttpOnly = true;
            });

            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_3_0);
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseExceptionHandler("/Home/Error");
            }

            app.UseStaticFiles();
            app.UseCookiePolicy();
            // ���Session����Ҫ�����MVC����ǰ�棬��ΪMVC����Ҫ��
            app.UseSession();

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync("Hello World!");
                });
            });
        }
    }
}
```



Dockerfile

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/aspnet:2.1-stretch-slim AS base
WORKDIR /app
ENTRYPOINT ["dotnet", "LeadChina.Hello.API.dll"]
```

���ɾ���

```sh
docker build -t hello .
```

��������

```sh
docker run -d -v /data/sftp/mysftp/upload/hello/:/app --name hello hello
```

## ��������

```json
// ����
"RateLimitOptions": {
    "ClientWhitelist": [],
    "EnableRateLimiting": true,  # �Ƿ�����
    "Period": "1s",  # ͳ��ʱ���
    "PeriodTimespan": 1,  # �������ͻ���������
    "Limit": 1  # ͳ��ʱ�������������Ĵ���
}

// �۶�
"QoSOptions": {
    "ExceptionsAllowedBeforeBreaking":3,  # ������ٸ��쳣����
    "DurationOfBreak":5,  # �۶�ʱ��
    "TimeoutValue":5000  # �����������Ĵ���ʱ�䳬����ֵ���Զ�����������Ϊ��ʱ
}

// ���ؾ���
{
    "DownstreamPathTemplate": "/api/posts/{postId}",
    "DownstreamScheme": "https",
    "DownstreamHostAndPorts": [
            {
                "Host": "10.0.1.10",
                "Port": 5000,
            },
            {
                "Host": "10.0.1.11",
                "Port": 5000,
            }
        ],
    "UpstreamPathTemplate": "/posts/{postId}",
    "LoadBalancer": "RoundRobin",  # LeastConnection�����������٣���RoundRobin���ַ�����NoLoadBalance�������ã�
    "UpstreamHttpMethod": [ "Put", "Delete" ]
}

// ����ۺ�
{
    "ReRoutes": [
        {
            "DownstreamPathTemplate": "/",
            "UpstreamPathTemplate": "/laura",
            "UpstreamHttpMethod": [
                "Get"
            ],
            "DownstreamScheme": "http",
            "DownstreamHostAndPorts": [
                {
                    "Host": "localhost",
                    "Port": 51881
                }
            ],
            "Key": "Laura"
        },
        {
            "DownstreamPathTemplate": "/",
            "UpstreamPathTemplate": "/tom",
            "UpstreamHttpMethod": [
                "Get"
            ],
            "DownstreamScheme": "http",
            "DownstreamHostAndPorts": [
                {
                    "Host": "localhost",
                    "Port": 51882
                }
            ],
            "Key": "Tom"
        }
    ],
    "Aggregates": [
        {
            "ReRouteKeys": [
                "Tom",
                "Laura"
            ],
            "UpstreamPathTemplate": "/"
        }
    ]
}
```



