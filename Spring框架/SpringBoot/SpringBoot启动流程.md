1. 启动入口：调用 SpringApplication.run()

2. 准备环境：创建并配置 Environment（读取 application.yml、系统变量等）

3. 创建容器：根据应用类型（Web/非Web）创建对应的 ApplicationContext

4. 刷新容器（核心）：
   - 加载Bean（@ComponentScan、@Configuration 等）
   - 自动配置生效（@EnableAutoConfiguration）
   - 实例化并初始化单例 Bean
   
4. 如果是Web应用，创建并启动内嵌 Web 服务器

5. 发布启动完成事件，应用开始对外提供服务