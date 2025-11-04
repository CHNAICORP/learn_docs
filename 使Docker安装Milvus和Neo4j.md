我来为你创建一个完整的最小化测试demo，使用Docker安装Milvus和Neo4j，并用Python进行数据操作。

## 1. Docker Compose 配置文件

创建 `docker-compose.yml`：

```yaml
version: '3.5'

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
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3

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
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
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
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
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
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/neo4j/data:/data
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/neo4j/logs:/logs
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/neo4j/import:/var/lib/neo4j/import
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/neo4j/plugins:/plugins
    healthcheck:
      test: ["CMD", "cypher-shell", "-u", "neo4j", "-p", "password123", "RETURN 1"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  milvus:
  neo4j:
```

## 2. Python 测试代码

创建 `test_demo.py`：

```python
import time
import numpy as np
from pymilvus import connections, FieldSchema, CollectionSchema, DataType, Collection, utility
from neo4j import GraphDatabase

class MilvusNeo4jDemo:
    def __init__(self):
        # 连接信息
        self.milvus_host = "localhost"
        self.milvus_port = "19530"
        self.neo4j_uri = "bolt://localhost:7687"
        self.neo4j_user = "neo4j"
        self.neo4j_password = "password123"
        
    def setup_milvus(self):
        """设置Milvus向量数据库"""
        print("正在连接Milvus...")
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
        
        print("Milvus集合创建成功！")
    
    def setup_neo4j(self):
        """设置Neo4j图数据库"""
        print("正在连接Neo4j...")
        self.neo4j_driver = GraphDatabase.driver(
            self.neo4j_uri, 
            auth=(self.neo4j_user, self.neo4j_password)
        )
        
        # 清空现有数据
        with self.neo4j_driver.session() as session:
            session.run("MATCH (n) DETACH DELETE n")
            print("Neo4j数据库已清空")
    
    def insert_sample_data(self):
        """向两个数据库插入示例数据"""
        print("正在插入示例数据...")
        
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
        
        # 生成随机向量
        vectors = np.random.random((len(books), 128)).tolist()
        
        # 插入数据到Milvus
        entities = [
            books,
            vectors
        ]
        
        insert_result = self.collection.insert(entities)
        self.collection.flush()
        
        print(f"向Milvus插入了 {len(books)} 本书")
        
        # 向Neo4j插入数据
        with self.neo4j_driver.session() as session:
            # 创建图书节点
            for i, title in enumerate(books):
                session.run(
                    "CREATE (b:Book {id: $id, title: $title})",
                    id=i+1, title=title
                )
            
            # 创建作者节点和关系
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
                    "CREATE (a)-[:WROTE]->(b)",
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
        
        print("向Neo4j插入了图书、作者和关系数据")
    
    def test_milvus_search(self):
        """测试Milvus向量搜索"""
        print("\n=== 测试Milvus向量搜索 ===")
        
        # 加载集合
        self.collection.load()
        
        # 生成查询向量
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
        
        print("搜索结果:")
        for i, hit in enumerate(results[0]):
            print(f"{i+1}. 图书: {hit.entity.get('title')}, 距离: {hit.distance:.4f}")
    
    def test_neo4j_query(self):
        """测试Neo4j图查询"""
        print("\n=== 测试Neo4j图查询 ===")
        
        with self.neo4j_driver.session() as session:
            # 查询所有作者和他们写的书
            print("作者和他们的著作:")
            result = session.run("""
                MATCH (a:Author)-[:WROTE]->(b:Book)
                RETURN a.name as author, collect(b.title) as books
            """)
            
            for record in result:
                print(f"作者: {record['author']}, 著作: {', '.join(record['books'])}")
            
            # 查询相似图书
            print("\n相似图书关系:")
            result = session.run("""
                MATCH (b1:Book)-[r:SIMILAR_TO]->(b2:Book)
                RETURN b1.title as book1, b2.title as book2, r.score as similarity
            """)
            
            for record in result:
                print(f"相似对: {record['book1']} -> {record['book2']}, 相似度: {record['similarity']}")
    
    def combined_query(self):
        """组合查询：在Milvus中找到相似图书，然后在Neo4j中查找相关作者"""
        print("\n=== 组合查询测试 ===")
        
        # 首先在Milvus中搜索
        self.collection.load()
        query_vector = np.random.random((1, 128)).tolist()
        
        search_params = {"metric_type": "L2", "params": {"nprobe": 10}}
        results = self.collection.search(
            data=query_vector,
            anns_field="embedding",
            param=search_params,
            limit=2,
            output_fields=["title"]
        )
        
        # 获取搜索到的图书标题
        found_books = [hit.entity.get('title') for hit in results[0]]
        print(f"Milvus搜索到的图书: {found_books}")
        
        # 然后在Neo4j中查询这些图书的作者
        with self.neo4j_driver.session() as session:
            for book_title in found_books:
                result = session.run("""
                    MATCH (a:Author)-[:WROTE]->(b:Book {title: $title})
                    RETURN a.name as author
                """, title=book_title)
                
                authors = [record["author"] for record in result]
                if authors:
                    print(f"图书 '{book_title}' 的作者: {', '.join(authors)}")
                else:
                    print(f"图书 '{book_title}' 没有找到作者信息")
    
    def cleanup(self):
        """清理资源"""
        if hasattr(self, 'neo4j_driver'):
            self.neo4j_driver.close()
        connections.disconnect("default")
        print("\n资源已清理")

def wait_for_services():
    """等待服务启动"""
    print("等待服务启动...")
    time.sleep(30)

def main():
    demo = MilvusNeo4jDemo()
    
    try:
        # 等待服务完全启动
        wait_for_services()
        
        # 设置数据库
        demo.setup_milvus()
        demo.setup_neo4j()
        
        # 插入数据
        demo.insert_sample_data()
        
        # 测试功能
        demo.test_milvus_search()
        demo.test_neo4j_query()
        demo.combined_query()
        
        print("\n✅ Demo执行完成！")
        
    except Exception as e:
        print(f"❌ 发生错误: {e}")
    finally:
        demo.cleanup()

if __name__ == "__main__":
    main()
```

## 3. 运行脚本

创建 `run_demo.sh`：

```bash
#!/bin/bash

echo "启动 Milvus 和 Neo4j 服务..."
docker-compose up -d

echo "等待服务启动..."
sleep 40

echo "运行 Python 测试脚本..."
python test_demo.py

echo "Demo 完成！"
echo "可以访问:"
echo "Neo4j Browser: http://localhost:7474 (用户名: neo4j, 密码: password123)"
```

## 4. 依赖文件

创建 `requirements.txt`：

```txt
pymilvus==2.3.4
neo4j==5.13.0
numpy==1.24.3
```

## 5. 运行说明

1. **启动服务**：
```bash
chmod +x run_demo.sh
./run_demo.sh
```

2. **或者手动运行**：
```bash
# 启动Docker服务
docker-compose up -d

# 安装Python依赖
pip install -r requirements.txt

# 运行测试
python test_demo.py
```

## 6. 验证服务

- **Neo4j Browser**: http://localhost:7474
  - 用户名: `neo4j`
  - 密码: `password123`

- **Milvus管理**: 可通过Python客户端验证

## 功能说明

这个demo演示了：

1. **Milvus功能**：
   - 创建向量集合
   - 插入向量数据
   - 向量相似度搜索

2. **Neo4j功能**：
   - 创建图节点和关系
   - 图查询遍历
   - 关系分析

3. **组合查询**：
   - 用Milvus找到相似内容
   - 用Neo4j分析关联关系

这个最小化demo展示了两种数据库的典型使用场景，你可以根据需要扩展功能！