# Oleksandr Bilyk - software developer, solution architect, system thinker. 

### [Secret Arch](./SecretArch/Readme.md)
*Sep 04, 2021*

Azure KeyVault is Azure service for storing secrets. Let's imagine that we need to remove secrets after their expiration that KeyVault doesn't provide. It would be nice to design KeyVault decorator service that will provide missing features. Let's use Event Sourcing and CosmosDB Change feed for such purposes.

### [F# Lazy Expiration](./FSharpLazyExpiration/Readme.md)
*Feb 13, 2021*

Comparison of OOP and Functional looks at one problem with resource usage.

### [From Service Fabric to App Service](./FromServiceFabricToAppService/Readme.md)
*Jun 30, 2021*

Service Fabric is super powerful PAAS that was not polished by open source comunity to have a good usage expirience. Article contains story of moving few microservices from Service Fabric to App Service. It is story about how importantce dependency inversion architectural princyple helps to build platform agnostic solutions.

### [Http Client Factory](./AppServiceHttpClientFactory/Readme.md)
*Feb 14, 2021*

Article describes HttpClientFactory usage best practices. Ignoring network connections allocation recommendations cause networking issues on Azure App Service and not only there. 



### [Windows Application Block](https://github.com/oleksandr-bilyk/WindowsApplicationBlock) 
*Jun 4, 2017*

My first open-source repository and article. In 2017 I was prepared for MCSD Windows Application Builder certification and after 10+ years of desktop development was very excited by Windows 10 UWP platform. Describes deep analysis of lifecycle, dependency injection, navigation. The article was attempt to write a solid Application Block for UWP MVVM fancy scenarious, dependency injection with .NET Native (withoug DI containers). 


