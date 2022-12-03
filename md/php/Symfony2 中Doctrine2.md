CreateTime:2015-12-07 11:38:07.0

###symfony2 中 根据 doctrine的entity 生成数据表

php app/console doctrine:schema:update  
这行并不会真正执行，只是计算下需要执行多少条sql语句


php app/console doctrine:schema:update --dump-sql 
将要执行的sql语句打印到命令行

php app/console doctrine:schema:update --force 
执行，这才是真正的执行



###Symfony2 Doctrine从现有Database生成Entity

生成元数据

php app/console doctrine:mapping:import --force SiteHomeBundle xml


生成Entity

php app/console doctrine:mapping:convert annotation ./src

生成getter setter

php app/console doctrine:generate:entities SiteHomeBundle --no-backup



###Symfony2 Doctrine 其他实用

doctrine:generate:crud                      
基于Doctrine实体生成增删改查（CRUD）

doctrine:schema:create                     
执行（或转储）生成数据库方案所需的SQL语句

doctrine:schema:drop                       
执行（或转储）删除数据库方案所需的SQL语句

doctrine:schema:update                   
执行（或转储）更新匹配当前映射元数据数据库方案所需的SQL语句
