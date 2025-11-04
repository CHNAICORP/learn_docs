# 在Windows上使用Docker安装Milvus和Neo4j的完整指南

## 1. 准备工作

### 安装Docker Desktop for Windows
1. 访问 [Docker官网](https://www.docker.com/products/docker-desktop/) 下载Docker Desktop
2. 安装时确保启用WSL 2后端（推荐）或Hyper-V
3. 完成安装后重启电脑

### 验证Docker安装
```powershell
docker --version
docker-compose --version
```

## 2. 项目结构

创建项目文件夹 `milvus_neo4j_demo`，结构如下：
```
milvus_neo4j_demo/
├── docker-compose.yml
├── test_demo.py
├── requirements.txt
└── run_demo.bat
```

## 3. Docker Compose 配置

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - milvus_etcd_data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    ports:
      - "2379:2379"

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    ports:
      - "9001:9001"
      - "9000:9000"
    volumes:
      - milvus_minio_data:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  milvus:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.3.4
    command: ["milvus", "run", "standalone"]
    security_opt:
    - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
    volumes:
      - milvus_data:/var/lib/milvus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      start_period: 90s
      timeout: 20s
      retries: 3
    ports:
      - "19530:19530"
      - "9091:9091"
    depends_on:
      - "etcd"
      - "minio"

  neo4j:
    image: neo4j:5.13
    container_name: neo4j
    environment:
      - NEO4J_AUTH=neo4j/password123
      - NEO4J_PLUGINS=["apoc"]
    ports:
      - "7474:7474"
      - "7687:7687"
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
      - neo4j_import:/var/lib/neo4j/import
      - neo4j_plugins:/plugins
    healthcheck:
      test: ["CMD", "cypher-shell", "-u", "neo4j", "-p", "password123", "RETURN 1"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  milvus_etcd_data:
  milvus_minio_data:
  milvus_data:
  neo4j_data:
  neo4j_logs:
  neo4j_import:
  neo4j_plugins:
```

## 4. Python测试代码

创建 `test_demo.py`：

```python
import time
import numpy as np
from pymilvus import connections, FieldSchema, CollectionSchema, DataType, Collection, utility
from neo4j import GraphDatabase
import logging

# 设置日志
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class MilvusNeo4jDemo:
    def __init__(self):
        # Windows Docker Desktop 使用 localhost 连接
        self.milvus_host = "localhost"
        self.milvus_port = "19530"
        self.neo4j_uri = "bolt://localhost:7687"
        self.neo4j_user = "neo4j"
        self.neo4j_password = "password123"
        
    def wait_for_milvus(self, timeout=60):
        """等待Milvus服务就绪"""
        logger.info("等待Milvus服务启动...")
        start_time = time.time()
        while time.time() - start_time < timeout:
            try:
                connections.connect("default", host=self.milvus_host, port=self.milvus_port)
                if utility.get_server_version():
                    logger.info("Milvus连接成功！")
                    connections.disconnect("default")
                    return True
            except Exception as e:
                logger.info(f"等待Milvus... ({e})")
                time.sleep(5)
        raise Exception("Milvus启动超时")
    
    def wait_for_neo4j(self, timeout=60):
        """等待Neo4j服务就绪"""
        logger.info("等待Neo4j服务启动...")
        start_time = time.time()
        while time.time() - start_time < timeout:
            try:
                driver = GraphDatabase.driver(
                    self.neo4j_uri, 
                    auth=(self.neo4j_user, self.neo4j_password)
                )
                with driver.session() as session:
                    session.run("RETURN 1")
                driver.close()
                logger.info("Neo4j连接成功！")
                return True
            except Exception as e:
                logger.info(f"等待Neo4j... ({e})")
                time.sleep(5)
        raise Exception("Neo4j启动超时")
    
    def setup_milvus(self):
        """设置Milvus向量数据库"""
        logger.info("正在设置Milvus...")
        connections.connect("default", host=self.milvus_host, port=self.milvus_port)
        
        # 检查集合是否存在，如果存在则删除
        if utility.has_collection("book_collection"):
            utility.drop_collection("book_collection")
        
        # 定义字段
        fields = [
            FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
            FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=200),
            FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=128)
        ]
        
        # 创建集合schema
        schema = CollectionSchema(fields=fields, description="图书向量搜索")
        
        # 创建集合
        self.collection = Collection(name="book_collection", schema=schema)
        
        # 创建索引
        index_params = {
            "index_type": "IVF_FLAT",
            "metric_type": "L2",
            "params": {"nlist": 128}
        }
        self.collection.create_index(field_name="embedding", index_params=index_params)
        
        logger.info("Milvus集合创建成功！")
    
    def setup_neo4j(self):
        """设置Neo4j图数据库"""
        logger.info("正在设置Neo4j...")
        self.neo4j_driver = GraphDatabase.driver(
            self.neo4j_uri, 
            auth=(self.neo4j_user, self.neo4j_password)
        )
        
        # 清空现有数据
        with self.neo4j_driver.session() as session:
            session.run("MATCH (n) DETACH DELETE n")
            logger.info("Neo4j数据库已清空")
    
    def insert_sample_data(self):
        """向两个数据库插入示例数据"""
        logger.info("正在插入示例数据...")
        
        # 向Milvus插入数据
        books = [
            "人工智能导论",
            "机器学习实战",
            "深度学习原理",
            "自然语言处理",
            "计算机视觉",
            "图神经网络",
            "数据挖掘技术",
            "Python编程"
        ]
        
        # 生成随机向量（模拟嵌入向量）
        np.random.seed(42)  # 确保可重复性
        vectors = np.random.random((len(books), 128)).tolist()
        
        # 插入数据到Milvus
        data = [
            books,
            vectors
        ]
        
        insert_result = self.collection.insert(data)
        self.collection.flush()
        
        logger.info(f"向Milvus插入了 {len(books)} 本书")
        
        # 向Neo4j插入数据
        with self.neo4j_driver.session() as session:
            # 创建图书节点
            for i, title in enumerate(books):
                session.run(
                    "CREATE (b:Book {id: $id, title: $title})",
                    id=i+1, title=title
                )
            
            # 创建作者节点
            authors = ["张三", "李四", "王五"]
            for author in authors:
                session.run("CREATE (a:Author {name: $name})", name=author)
            
            # 创建作者-图书关系
            relationships = [
                ("张三", "人工智能导论"),
                ("张三", "机器学习实战"),
                ("李四", "深度学习原理"),
                ("李四", "自然语言处理"),
                ("王五", "计算机视觉"),
                ("王五", "图神经网络")
            ]
            
            for author, book in relationships:
                session.run(
                    "MATCH (a:Author {name: $author}), (b:Book {title: $book}) "
                    "CREATE (a)-[:WROTE {year: 2023}]->(b)",
                    author=author, book=book
                )
            
            # 创建图书之间的相似关系
            session.run("""
                MATCH (b1:Book {title: '人工智能导论'}), (b2:Book {title: '机器学习实战'})
                CREATE (b1)-[:SIMILAR_TO {score: 0.85}]->(b2)
            """)
            
            session.run("""
                MATCH (b1:Book {title: '深度学习原理'}), (b2:Book {title: '图神经网络'})
                CREATE (b1)-[:SIMILAR_TO {score: 0.92}]->(b2)
            """)
            
            # 创建一些类别节点
            categories = ["AI", "编程", "数据科学"]
            for category in categories:
                session.run("CREATE (c:Category {name: $name})", name=category)
            
            # 创建图书-类别关系
            category_relations = [
                ("人工智能导论", "AI"),
                ("机器学习实战", "AI"),
                ("深度学习原理", "AI"),
                ("自然语言处理", "AI"),
                ("计算机视觉", "AI"),
                ("图神经网络", "AI"),
                ("数据挖掘技术", "数据科学"),
                ("Python编程", "编程")
            ]
            
            for book, category in category_relations:
                session.run("""
                    MATCH (b:Book {title: $book}), (c:Category {name: $category})
                    CREATE (b)-[:BELONGS_TO]->(c)
                """, book=book, category=category)
        
        logger.info("向Neo4j插入了图书、作者、类别和关系数据")
    
    def test_milvus_search(self):
        """测试Milvus向量搜索"""
        logger.info("\n=== 测试Milvus向量搜索 ===")
        
        # 加载集合
        self.collection.load()
        
        # 生成查询向量
        np.random.seed(123)
        query_vector = np.random.random((1, 128)).tolist()
        
        # 搜索参数
        search_params = {"metric_type": "L2", "params": {"nprobe": 10}}
        
        # 执行搜索
        results = self.collection.search(
            data=query_vector,
            anns_field="embedding",
            param=search_params,
            limit=3,
            output_fields=["title"]
        )
        
        print("Milvus向量搜索结果:")
        for i, hit in enumerate(results[0]):
            print(f"{i+1}. 图书: {hit.entity.get('title')}, 距离: {hit.distance:.4f}")
    
    def test_neo4j_query(self):
        """测试Neo4j图查询"""
        logger.info("\n=== 测试Neo4j图查询 ===")
        
        with self.neo4j_driver.session() as session:
            # 查询所有作者和他们写的书
            print("作者和他们的著作:")
            result = session.run("""
                MATCH (a:Author)-[w:WROTE]->(b:Book)
                RETURN a.name as author, collect(b.title) as books, w.year as year
            """)
            
            for record in result:
                print(f"作者: {record['author']}, 著作: {', '.join(record['books'])}")
            
            # 查询图书分类
            print("\n图书分类:")
            result = session.run("""
                MATCH (b:Book)-[:BELONGS_TO]->(c:Category)
                RETURN c.name as category, collect(b.title) as books
                ORDER BY category
            """)
            
            for record in result:
                print(f"类别: {record['category']}, 图书: {', '.join(record['books'])}")
            
            # 查询相似图书
            print("\n相似图书关系:")
            result = session.run("""
                MATCH (b1:Book)-[r:SIMILAR_TO]->(b2:Book)
                RETURN b1.title as book1, b2.title as book2, r.score as similarity
                ORDER BY similarity DESC
            """)
            
            for record in result:
                print(f"相似对: {record['book1']} -> {record['book2']}, 相似度: {record['similarity']}")
    
    def combined_query(self):
        """组合查询：在Milvus中找到相似图书，然后在Neo4j中查找相关作者和类别"""
        logger.info("\n=== 组合查询测试 ===")
        
        # 首先在Milvus中搜索
        self.collection.load()
        np.random.seed(456)
        query_vector = np.random.random((1, 128)).tolist()
        
        search_params = {"metric_type": "L2", "params": {"nprobe": 10}}
        results = self.collection.search(
            data=query_vector,
            anns_field="embedding",
            param=search_params,
            limit=3,
            output_fields=["title"]
        )
        
        # 获取搜索到的图书标题
        found_books = [hit.entity.get('title') for hit in results[0]]
        print(f"Milvus向量搜索找到的图书: {found_books}")
        
        # 然后在Neo4j中查询这些图书的详细信息
        with self.neo4j_driver.session() as session:
            for book_title in found_books:
                print(f"\n查询图书 '{book_title}' 的详细信息:")
                
                # 查询作者
                result = session.run("""
                    MATCH (a:Author)-[:WROTE]->(b:Book {title: $title})
                    RETURN a.name as author
                """, title=book_title)
                
                authors = [record["author"] for record in result]
                if authors:
                    print(f"  作者: {', '.join(authors)}")
                else:
                    print(f"  作者: 未找到")
                
                # 查询类别
                result = session.run("""
                    MATCH (b:Book {title: $title})-[:BELONGS_TO]->(c:Category)
                    RETURN c.name as category
                """, title=book_title)
                
                categories = [record["category"] for record in result]
                if categories:
                    print(f"  类别: {', '.join(categories)}")
                else:
                    print(f"  类别: 未分类")
    
    def cleanup(self):
        """清理资源"""
        if hasattr(self, 'neo4j_driver'):
            self.neo4j_driver.close()
        connections.disconnect("default")
        logger.info("资源已清理")

def main():
    demo = MilvusNeo4jDemo()
    
    try:
        logger.info("开始Milvus + Neo4j Demo")
        
        # 等待服务启动
        demo.wait_for_milvus()
        demo.wait_for_neo4j()
        
        # 设置数据库
        demo.setup_milvus()
        demo.setup_neo4j()
        
        # 插入数据
        demo.insert_sample_data()
        
        # 测试功能
        demo.test_milvus_search()
        demo.test_neo4j_query()
        demo.combined_query()
        
        print("\n" + "="*50)
        print("✅ Demo执行完成！")
        print("可以访问以下地址查看服务:")
        print("Neo4j Browser: http://localhost:7474")
        print("用户名: neo4j")
        print("密码: password123")
        print("="*50)
        
        # 等待用户输入后再退出
        input("\n按Enter键退出...")
        
    except Exception as e:
        logger.error(f"发生错误: {e}")
        import traceback
        traceback.print_exc()
    finally:
        demo.cleanup()

if __name__ == "__main__":
    main()
```

## 5. 依赖文件

创建 `requirements.txt`：

```txt
pymilvus==2.3.4
neo4j==5.13.0
numpy==1.24.3
```

## 6. Windows批处理脚本

创建 `run_demo.bat`：

```batch
@echo off
chcp 65001 > nul
echo ========================================
echo Milvus + Neo4j Windows Demo
echo ========================================

echo 正在启动Docker服务...
docker-compose down
docker-compose up -d

echo 等待服务启动（30秒）...
timeout /t 30 /nobreak > nul

echo 检查服务状态...
docker ps

echo 安装Python依赖...
pip install -r requirements.txt

echo 运行Demo...
python test_demo.py

echo Demo完成！
echo.
echo 可以访问：
echo Neo4j Browser: http://localhost:7474
echo 用户名: neo4j
echo 密码: password123
echo.
echo 按任意键停止服务并退出...
pause > nul

echo 停止Docker服务...
docker-compose down
```

## 7. 运行步骤

### 方法一：使用批处理脚本（推荐）
1. 打开命令提示符或PowerShell
2. 导航到项目目录：
   ```cmd
   cd milvus_neo4j_demo
   ```
3. 运行批处理脚本：
   ```cmd
   run_demo.bat
   ```

### 方法二：手动运行
1. 启动Docker服务：
   ```cmd
   docker-compose up -d
   ```
2. 等待30秒让服务完全启动
3. 安装Python依赖：
   ```cmd
   pip install -r requirements.txt
   ```
4. 运行Python脚本：
   ```cmd
   python test_demo.py
   ```

## 8. 常见问题解决

### Docker Desktop问题
1. **WSL 2未安装**：
   - 以管理员身份运行PowerShell：
     ```powershell
     wsl --install
     ```
   - 重启电脑

2. **端口冲突**：
   - 检查端口19530、7474、7687是否被占用
   - 修改`docker-compose.yml`中的端口映射

3. **内存不足**：
   - 在Docker Desktop设置中增加内存分配（建议至少4GB）

### 连接问题
1. **Milvus连接失败**：
   - 等待更长时间让服务完全启动
   - 检查Milvus日志：`docker logs milvus-standalone`

2. **Neo4j连接失败**：
   - 访问 http://localhost:7474 验证Neo4j是否运行
   - 检查Neo4j日志：`docker logs neo4j`

## 9. 验证安装

成功运行后，你应该看到：
- Milvus向量搜索结果显示相似的图书
- Neo4j图查询显示作者、类别和关系信息
- 组合查询展示两种数据库的协同工作

访问 http://localhost:7474 可以在Neo4j浏览器中可视化图数据。

这个配置在Windows Docker Desktop上经过测试，应该可以顺利运行！