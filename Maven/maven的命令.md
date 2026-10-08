+ mvn clean
+ mvn validate
+ mvn compile
+ mvn test
+ mvn package
+ mvn verify
+ mvn install
+ mvn deploy
上述顺序也是maven的生命周期

````
# 直接启动项目
mvn spring-boot:run

# 打包后运行
mvn clean package
java -jar target/app.jar
```