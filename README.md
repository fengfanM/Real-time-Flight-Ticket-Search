# 海航随心飞助手 - 深度技术解析

## 🚀 快速开始

### 本地运行

项目已经在本地启动！

**访问地址**：http://localhost:3000

### 项目信息

| 项目 | 详情 |
|------|------|
| **项目名称** | 海航随心飞助手 |
| **技术栈** | Next.js 14 + TypeScript + Tailwind CSS |
| **数据来源** | 历史航班快照 CSV |
| **航班数量** | 1994 条 |
| **覆盖城市** | 168 个 |
| **航线数量** | 1109 条 |

---

## 🛠️ 完整技术栈详解

### 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| **Next.js** | 14.2.9 | React 全栈框架，App Router |
| **React** | 18.x | UI 构建库 |
| **TypeScript** | 5.x | 类型安全 |
| **Tailwind CSS** | 3.x | 原子化 CSS 框架 |
| **shadcn/ui** | - | 组件库 |
| **高德地图 JS API** | 2.x | 地图可视化 |

### 后端技术栈

| 技术 | 用途 |
|------|------|
| **Next.js API Routes** | 后端 API 接口 |
| **Node.js** | 运行时环境 |
| **CSV 解析** | 数据加载 |
| **mtime 热更新** | 文件监控与缓存失效 |

### 开发工具

| 工具 | 用途 |
|------|------|
| **pnpm** | 包管理器 |
| **ESLint** | 代码检查 |
| **Prettier** | 代码格式化 |
| **Docker** | 容器化部署 |

---

## 🧠 核心算法深度解析

### 1. 班期位图编码算法

#### 问题背景
海航随心飞的航班班期通常表示为 "1234567"（每天）或 "135"（周一三五）这种字符串形式。直接用字符串存储和查询效率低。

#### 算法实现（`lib/flight/utils.ts`）

```typescript
/**
 * 将班期字符串转换为位图
 * 例如："135" → 0b10101 (21)
 * 位0=周一，位1=周二，...，位6=周日
 */
export function parseDowBitmap(dowStr: string): number {
  let bitmap = 0;
  for (const c of dowStr) {
    const day = parseInt(c, 10);
    if (day >= 1 && day <= 7) {
      bitmap |= 1 << (day - 1);
    }
  }
  return bitmap;
}

/**
 * 检查某天是否有航班
 * 例如：bitmap=0b10101，targetDay=1 → true（周一）
 */
export function hasFlightOnDay(bitmap: number, targetDay: number): boolean {
  return (bitmap & (1 << (targetDay - 1))) !== 0;
}
```

#### 优势
- ✅ **O(1) 查询**：无需遍历字符串，直接位运算
- ✅ **空间节省**：1 个数字存 7 天信息
- ✅ **组合灵活**：支持多班次组合查询

---

### 2. BFS 最短路径搜索算法（中转路线）

#### 问题背景
需要找到从起点城市到终点城市的所有可行中转路线，支持 0-2 次中转。

#### 算法设计（`lib/flight/search-service.ts`）

```
数据结构：
- SearchState: 当前城市、已乘坐航班、总耗时
- Priority Queue: 按总耗时排序（可选优化）

搜索流程：
1. 初始化队列，起点城市入队
2. 取出队首元素
3. 如果当前城市是终点 → 加入结果
4. 否则，遍历从当前城市出发的所有航班
5. 对于每个航班：
   - 检查中转时间是否满足（1-24 小时）
   - 检查总耗时是否超限（≤48 小时）
   - 检查中转次数是否超限（≤2）
   - 如果满足，加入队列继续搜索
6. 重复步骤 2-5，直到队列为空
```

#### 关键代码片段

```typescript
export class FlightSearchService {
  private readonly MAX_CONNECTIONS = 2;
  private readonly MIN_CONNECT_MINS = 60;
  private readonly MAX_CONNECT_MINS = 1440;
  private readonly MAX_TOTAL_HOURS = 48;

  searchFlights(params: SearchParams): { routes: Route[], restriction?: string } {
    // 1. 参数校验
    const { origin_city, dest_city, date, max_connections = 2 } = params;

    // 2. 检查节假日限制（666版本）
    if (this.isDateRestricted(date, params.version)) {
      return { routes: [], restriction: '666版本节假日限制期间' };
    }

    // 3. BFS 初始化
    const queue: SearchState[] = [{
      city: origin_city,
      flights: [],
      arrivalTime: null,
      connections: 0
    }];
    const results: Route[] = [];

    // 4. BFS 主循环
    while (queue.length > 0 && results.length < 200) {
      const state = queue.shift()!;

      // 到达终点，记录结果
      if (state.city === dest_city && state.flights.length > 0) {
        results.push({
          flights: [...state.flights],
          totalDuration: this.calculateTotalDuration(state.flights),
          connections: state.connections
        });
        continue;
      }

      // 未到达终点，继续搜索
      const weekDay = this.getWeekDay(date, state.flights.length);
      const outgoingFlights = this.dataService.getFlightsFromCity(state.city, weekDay);

      for (const flight of outgoingFlights) {
        // 检查中转时间约束
        if (state.arrivalTime) {
          const connectMins = this.calculateConnectTime(state.arrivalTime, flight.departureTime);
          if (connectMins < this.MIN_CONNECT_MINS || connectMins > this.MAX_CONNECT_MINS) {
            continue;
          }
        }

        // 检查中转次数
        const newConnections = state.flights.length > 0 ? state.connections + 1 : 0;
        if (newConnections > max_connections) {
          continue;
        }

        // 加入队列
        queue.push({
          city: flight.dest_city,
          flights: [...state.flights, flight],
          arrivalTime: flight.arrivalTime,
          connections: newConnections
        });
      }
    }

    return { routes: results };
  }
}
```

#### 算法复杂度分析

| 维度 | 复杂度 |
|------|--------|
| **时间复杂度** | O(V + E)，V=城市数，E=航线条数 |
| **空间复杂度** | O(V)，队列和结果存储 |
| **最坏情况** | O((C * F)^K)，C=城市数，F=平均航班数，K=中转次数 |

#### 优化策略

1. **剪枝策略**：
   - 中转时间不在 1-24 小时 → 剪枝
   - 总耗时超过 48 小时 → 剪枝
   - 中转次数超限 → 剪枝

2. **去重策略**：
   - 避免重复访问同一城市（或记录最优路径）
   - 限制结果数量（最多 200 条）

3. **索引优化**：
   - 按 `出港城市 + 星期` 建立索引
   - 查询时间从 O(n) 降到 O(1)

---

### 3. 航班数据索引构建

#### 索引结构（`lib/flight/data-service.ts`）

```typescript
type FlightIndex = Map<
  string,                    // 出港城市
  Map<
    number,                  // 星期（1-7）
    Flight[]                 // 该城市该星期的所有航班
  >
>;

class FlightDataService {
  private flightIndex: FlightIndex = new Map();

  buildIndex(flights: Flight[]): void {
    for (const flight of flights) {
      // 解析班期位图
      const dowBitmap = parseDowBitmap(flight.dow);
      
      // 为每一天建立索引
      for (let day = 1; day <= 7; day++) {
        if (hasFlightOnDay(dowBitmap, day)) {
          this.addToIndex(flight.origin_city, day, flight);
        }
      }
    }
  }

  getFlightsFromCity(city: string, weekDay: number): Flight[] {
    return this.flightIndex.get(city)?.get(weekDay) || [];
  }
}
```

---

## 📐 查询设计逻辑与规则

### 1. 查询参数设计（`lib/flight/types.ts`）

```typescript
export interface SearchParams {
  origin_city: string;           // 起点城市
  dest_city: string;             // 终点城市
  date: string;                  // 出发日期（YYYY-MM-DD）
  max_connections?: number;      // 最大中转次数（0-2）
  time_filter?: 'early' | 'late'; // 时间筛选
  version?: '666' | '2666';     // 会员版本
}
```

### 2. 约束规则体系

| 规则类型 | 规则内容 | 目的 |
|----------|----------|------|
| **中转时间约束** | 最小 60 分钟，最大 1440 分钟 | 确保转机合理 |
| **总耗时约束** | ≤ 48 小时 | 避免过长路线 |
| **中转次数约束** | ≤ 2 次 | 用户体验 |
| **结果数量限制** | ≤ 200 条 | 性能优化 |
| **节假日限制** | 666 版本节假日不可用 | 产品规则 |

### 3. 节假日限制逻辑

```typescript
// 666 版本节假日限制日期
const RESTRICTED_DATES_666 = new Set([
  '2024-01-01', '2024-01-29', '2024-01-30',
  '2024-01-31', '2024-02-01', '2024-02-02',
  '2024-02-03', '2024-02-04', '2024-02-05',
  '2024-02-06', '2024-02-07', '2024-02-08',
  '2024-02-09', '2024-02-10', '2024-02-11',
  '2024-02-12', '2024-02-13', '2024-02-14',
  '2024-02-15', '2024-02-16', '2024-02-17',
  '2024-04-04', '2024-04-05', '2024-04-06',
  '2024-05-01', '2024-05-02', '2024-05-03',
  '2024-05-04', '2024-05-05', '2024-06-10',
  '2024-09-15', '2024-09-16', '2024-09-17',
  '2024-10-01', '2024-10-02', '2024-10-03',
  '2024-10-04', '2024-10-05', '2024-10-06',
  '2024-10-07', '2024-10-08'
]);

isDateRestricted(date: string, version?: string): boolean {
  if (version === '666' && RESTRICTED_DATES_666.has(date)) {
    return true;
  }
  return false;
}
```

---

## 🏗️ 完整架构设计

### 分层架构图

```
┌─────────────────────────────────────────────────────────┐
│                        用户层                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   浏览器    │  │  移动端     │  │  第三方调用  │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
└─────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼──────────┐
│                      前端层 (Next.js)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Hero 组件   │  │  搜索结果    │  │  航线地图    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼──────────┐
│                    API 层 (Next.js API Routes)            │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │ /api/search      │  │ /api/flight-data │               │
│  │ /api/flight-cities│ │ /api/flight-stats│               │
│  └────────┬─────────┘  └────────┬─────────┘               │
└──────────┼────────────────────────┼─────────────────────────┘
           │                        │
┌──────────▼────────────────────────▼─────────────────────────┐
│                      业务逻辑层 (lib/flight)                  │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │ FlightSearchService│ │  FlightDataService│               │
│  │  - BFS 搜索      │ │  - 索引构建       │               │
│  │  - 约束检查      │ │  - 数据查询       │               │
│  └────────┬─────────┘  └────────┬─────────┘               │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │  FlightLoader    │ │     Utils         │               │
│  │  - CSV 解析      │ │  - 时间转换       │               │
│  │  - mtime 热更新  │ │  - 班期位图       │               │
│  └────────┬─────────┘  └────────┬─────────┘               │
└──────────┼────────────────────────┼─────────────────────────┘
           │                        │
┌──────────▼────────────────────────▼─────────────────────────┐
│                        数据层                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              data/airport.csv (航班快照)               │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 数据热更新机制（`lib/flight/flight-loader.ts`）

```typescript
// 核心设计：基于文件 mtime 的缓存失效
interface CacheBundle {
  flights: Flight[];
  mtimeMs: number;
}

let cache: CacheBundle | null = null;
let pendingReload: Promise<CacheBundle> | null = null;

async function ensureLoaded(): Promise<CacheBundle> {
  // 1. 获取当前文件 mtime
  const current = await statMtime();
  
  // 2. 检查缓存是否有效
  if (cache && current === cache.mtimeMs) {
    return cache;
  }
  
  // 3. 如果正在加载，复用 Promise（防并发）
  if (pendingReload) {
    return pendingReload;
  }
  
  // 4. 开始加载
  pendingReload = readAndParse(current);
  
  try {
    cache = await pendingReload;
    return cache;
  } finally {
    pendingReload = null;
  }
}

// 使用示例
export async function getFlights(): Promise<Flight[]> {
  const bundle = await ensureLoaded();
  return bundle.flights;
}
```

**优势**：
- ✅ **零停机更新**：修改 CSV 后无需重启服务器
- ✅ **并发安全**：多个请求共享同一个加载 Promise
- ✅ **容错降级**：加载失败时沿用旧缓存

---

## 📱 产品功能详解

### 1. 航班搜索功能

| 功能 | 说明 |
|------|------|
| **单程搜索** | 一次行程搜索 |
| **往返搜索** | 去程 + 返程搜索 |
| **中转次数选择** | 0 次（直飞）、1 次、2 次 |
| **时间筛选** | 早班（00:00-08:59）、晚班（20:00-23:59） |
| **会员版本** | 666（节假日限制）、2666（无限制） |

### 2. 航线地图可视化

基于高德地图 JS API，实现：
- 航线绘制
- 机场标记
- 城市定位
- 缩放平移交互

### 3. 航班数据表

全量航班快照浏览，支持：
- 按城市筛选
- 按时间排序
- 分页展示
- 导出功能

### 4. 航班统计

多维度统计面板：
- 按出港城市统计
- 按机场统计
- 按航空公司统计
- 热门航线 Top 10

---

## 🔧 API 接口设计

### 1. 搜索接口 `POST /api/search`

**请求**：
```json
{
  "origin_city": "北京",
  "dest_city": "上海",
  "date": "2024-04-15",
  "max_connections": 2,
  "time_filter": null,
  "version": "2666"
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "routes": [
      {
        "flights": [
          {
            "carrier_name": "海南航空",
            "flight_no": "HU7801",
            "origin_city": "北京",
            "dest_city": "上海",
            "departureTime": "08:00",
            "arrivalTime": "10:30",
            "dow": "1234567"
          }
        ],
        "totalDuration": 150,
        "connections": 0
      }
    ],
    "restriction": null
  }
}
```

### 2. 城市列表接口 `GET /api/flight-cities`

**响应**：
```json
{
  "success": true,
  "data": ["北京", "上海", "广州", "深圳", ...]
}
```

### 3. 航班数据接口 `GET /api/flight-data`

**响应**：
```json
{
  "success": true,
  "data": {
    "flights": [...],
    "stats": {
      "totalFlights": 1994,
      "totalCities": 168,
      "totalRoutes": 1109
    }
  }
}
```

---

## 📱 功能说明

### 1. 航班搜索

- **单程/往返**：支持两种行程类型
- **中转次数**：0 次（直飞）、1 次、2 次
- **时间筛选**：
  - 早班：00:00-08:59
  - 晚班：20:00-23:59
- **会员版本**：666 或 2666

### 2. 航线地图

基于高德地图的航线可视化（需要配置 API Key）

### 3. 航班数据表

全量航班历史快照浏览，支持筛选、排序

### 4. 航班统计

按出港城市、机场维度的统计面板

---

## ⚠️ 重要提示

### 数据说明

- ❌ **不是实时数据**：这是历史快照，不随实际航班变化
- ❌ **不是官方数据**：与海南航空无合作关系
- ❌ **仅供学习参考**：严禁用于真实出行决策

### 实时查询飞飞乐

如果需要实时查询飞飞乐航线和库存，需要：

1. 获取飞飞乐官方 API Key
2. 接入实时 API
3. 替换本地 CSV 数据源

---

## 📝 开发命令

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build

# 启动生产服务器
npm start
```

---

## 🔗 相关链接

- **原项目仓库**：https://github.com/yixinmeng/sxfroute
- **在线体验**：https://sxfroute.com

---

## 📄 许可证

MIT License
