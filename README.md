mcp-platform/
├── README.md                          # 項目說明文檔
├── QUICKSTART.md                      # 快速開始指南
├── Makefile                           # Make 命令集合
├── docker-compose.yml                 # 本地開發環境配置
├── .env.example                       # 環境變量範例
├── init-db.sql                        # 數據庫初始化腳本
│
├── services/                          # 微服務目錄
│   ├── mcp-server/                   # MCP 協議服務器
│   │   ├── main.py                   # 主程式
│   │   ├── Dockerfile                # Docker 配置
│   │   └── requirements.txt          # Python 依賴
│   │
│   ├── api-gateway/                  # API 網關
│   │   ├── main.py                   # 主程式
│   │   ├── Dockerfile                # Docker 配置
│   │   └── requirements.txt          # Python 依賴
│   │
│   ├── edge-gateway/                 # 邊緣網關
│   │   ├── main.py                   # 主程式
│   │   ├── Dockerfile                # Docker 配置
│   │   └── requirements.txt          # Python 依賴
│   │
│   └── web-portal/                   # Web 前端 (React)
│       └── Dockerfile                # Docker 配置
│
├── firmware/                          # MCU 韌體代碼
│   └── stm32/                        # STM32 專案
│       └── Core/                     # 核心代碼
│           ├── Inc/                  # 頭文件
│           │   └── mcp_protocol.h   # MCP 協議定義
│           └── Src/                  # 源文件
│               ├── mcp_protocol.c   # MCP 協議實現
│               ├── mcp_dispatch.c   # 命令分派
│               ├── mcp_tools.c      # 工具函數
│               └── tca953x.c        # GPIO Expander 驅動
│
├── k8s/                              # Kubernetes 配置
│   ├── 00-namespace.yaml            # 命名空間
│   ├── 01-configmap.yaml            # 配置映射
│   ├── 02-postgres.yaml             # PostgreSQL 部署
│   ├── 03-redis-rabbitmq.yaml       # Redis & RabbitMQ
│   ├── 04-mcp-server.yaml           # MCP Server 部署
│   └── 05-api-gateway.yaml          # API Gateway + Ingress
│
├── terraform/                        # AWS 基礎設施即代碼
│   ├── main.tf                      # 主配置
│   ├── variables.tf                 # 變量定義
│   ├── outputs.tf                   # 輸出定義
│   ├── vpc.tf                       # VPC 配置
│   ├── eks.tf                       # EKS 集群
│   └── data-stores.tf               # RDS, Redis, RabbitMQ
│
├── scripts/                          # 部署腳本
│   └── deploy.sh                    # 自動化部署腳本
│
└── monitoring/                       # 監控配置
    └── prometheus.yml               # Prometheus 配置
```

## 核心組件說明

### 1. 雲端服務層 (services/)

#### MCP Server (services/mcp-server/)
- **功能**: 處理 MCP 協議,管理設備連接,整合 AI (Claude)
- **技術**: Python FastAPI, SQLAlchemy, Anthropic SDK
- **端口**: 8000
- **關鍵特性**:
  - WebSocket 設備連接
  - AI 驅動的設備控制
  - 設備註冊與管理
  - 消息持久化

#### API Gateway (services/api-gateway/)
- **功能**: RESTful API,認證授權,請求路由
- **技術**: FastAPI, JWT, Redis
- **端口**: 8000
- **關鍵特性**:
  - JWT 認證
  - API 限流
  - WebSocket 代理

#### Edge Gateway (services/edge-gateway/)
- **功能**: 邊緣計算節點,協議轉換
- **技術**: Python, pySerial, WebSocket
- **關鍵特性**:
  - UART/Serial 通訊
  - MCP 協議解析
  - 本地緩存 (Redis)
  - 離線隊列

### 2. MCU 韌體層 (firmware/stm32/)

#### MCP 協議處理器
- **檔案**: mcp_protocol.c/h
- **功能**: 二進制 MCP 封包解析與生成
- **特性**:
  - UART 中斷驅動
  - 零拷貝設計
  - CRC 校驗

#### 命令分派器
- **檔案**: mcp_dispatch.c
- **功能**: JSON 命令路由到工具函數
- **支援**: cJSON 輕量級解析

#### I2C GPIO Expander 驅動
- **檔案**: tca953x.c/h
- **支援設備**: TCA9536, TCA9539, TXE8116
- **功能**:
  - 8/16-bit GPIO 控制
  - I2C 掃描
  - 位操作優化

#### 工具函數
- **檔案**: mcp_tools.c
- **實現工具**:
  - read_sensor: 讀取 ADC
  - set_gpio: 控制 GPIO
  - read_gpio_expander: 讀取 I2C GPIO
  - write_gpio_expander: 寫入 I2C GPIO

### 3. Kubernetes 部署 (k8s/)

#### 基礎設施
- **PostgreSQL**: StatefulSet + PVC (20GB gp3)
- **Redis**: Deployment
- **RabbitMQ**: Deployment + Management UI

#### 應用服務
- **MCP Server**: Deployment (3 replicas) + HPA
- **API Gateway**: Deployment (2 replicas) + LoadBalancer
- **Ingress**: AWS ALB + SSL

### 4. AWS 基礎設施 (terraform/)

#### 網路層
- **VPC**: 10.0.0.0/16
- **Subnets**: 3 AZ × (Public + Private)
- **NAT Gateway**: 高可用配置

#### 計算層
- **EKS**: Kubernetes 1.28
- **Node Groups**: 
  - General: t3.medium (2-10 nodes)
  - Spot: 降低成本

#### 數據層
- **RDS PostgreSQL 15**: Multi-AZ, 100GB gp3
- **ElastiCache Redis 7**: Cluster mode, 2 nodes
- **Amazon MQ RabbitMQ**: Cluster Multi-AZ
