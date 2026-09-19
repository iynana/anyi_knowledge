**原理**：
+ `@SpringBootConfiguration` 是 Spring Boot 的配置类标记
+ `@EnableAutoConfiguration` 扫描`spring.factories`文件，读取声明的自动配置类，并根据在配置类中根据条件注解判断是否生效
+ `@ComponentScan` 扫描应用组件

**按需加载**：条件注解确保仅在特定条件满足时（类路径存在指定类、用户未自定义相关 Bean）才加载自动配置，避免不必要的 Bean 创建。