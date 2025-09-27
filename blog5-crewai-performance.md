# CrewAI框架深度解析（五）：性能优化与最佳实践

## 引言

在使用CrewAI构建多智能体应用时，性能优化是一个关键考虑因素。本文将深入探讨CrewAI的性能优化策略和最佳实践，帮助你构建高效、可扩展的多智能体系统。

## 性能优化策略

### 1. 异步执行优化

#### 1.1 理解CrewAI的执行模型

CrewAI支持多种执行模式，每种模式都有其适用的场景：

```python
from crewai import Crew, Process
import asyncio
import concurrent.futures
import time

class PerformanceOptimizedCrew(Crew):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.executor = concurrent.futures.ThreadPoolExecutor(max_workers=10)
        self.performance_metrics = {
            'execution_time': [],
            'memory_usage': [],
            'success_rate': []
        }

    async def execute_tasks_parallel(self):
        """并行执行任务"""
        start_time = time.time()

        # 识别可以并行执行的任务
        parallel_tasks = self._identify_parallel_tasks()

        # 创建异步任务
        tasks = []
        for task_group in parallel_tasks:
            group_tasks = [self._execute_task_async(task) for task in task_group]
            tasks.append(asyncio.gather(*group_tasks))

        # 执行所有任务组
        results = await asyncio.gather(*tasks)

        execution_time = time.time() - start_time
        self.performance_metrics['execution_time'].append(execution_time)

        return self._combine_results(results)

    def _identify_parallel_tasks(self):
        """识别可以并行执行的任务"""
        # 分析任务依赖关系
        dependency_graph = self._build_dependency_graph()
        parallel_groups = []

        # 简化的并行任务识别
        independent_tasks = []
        for task in self.tasks:
            if not self._has_dependencies(task, dependency_graph):
                independent_tasks.append(task)

        if independent_tasks:
            parallel_groups.append(independent_tasks)

        return parallel_groups

    def _build_dependency_graph(self):
        """构建任务依赖图"""
        graph = {}
        for task in self.tasks:
            graph[task] = task.context or []
        return graph

    def _has_dependencies(self, task, dependency_graph):
        """检查任务是否有依赖"""
        return len(dependency_graph.get(task, [])) > 0

    async def _execute_task_async(self, task):
        """异步执行单个任务"""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(self.executor, task.execute)

    def _combine_results(self, results):
        """合并执行结果"""
        combined = ""
        for group_result in results:
            if isinstance(group_result, list):
                for result in group_result:
                    combined += str(result) + "\n"
            else:
                combined += str(group_result) + "\n"
        return combined

    def get_performance_stats(self):
        """获取性能统计信息"""
        return {
            'average_execution_time': sum(self.performance_metrics['execution_time']) / len(self.performance_metrics['execution_time']) if self.performance_metrics['execution_time'] else 0,
            'total_executions': len(self.performance_metrics['execution_time']),
            'memory_usage': self.performance_metrics['memory_usage'],
            'success_rate': self._calculate_success_rate()
        }

    def _calculate_success_rate(self):
        """计算成功率"""
        if not self.performance_metrics['success_rate']:
            return 0
        return sum(self.performance_metrics['success_rate']) / len(self.performance_metrics['success_rate'])
```

#### 1.2 批处理优化

```python
class BatchProcessor:
    def __init__(self, batch_size=10):
        self.batch_size = batch_size
        self.processed_batches = []

    def process_tasks_in_batches(self, tasks):
        """批量处理任务"""
        results = []

        for i in range(0, len(tasks), self.batch_size):
            batch = tasks[i:i + self.batch_size]
            batch_result = self._process_batch(batch)
            results.extend(batch_result)
            self.processed_batches.append({
                'batch_size': len(batch),
                'processing_time': time.time(),
                'results_count': len(batch_result)
            })

        return results

    def _process_batch(self, batch):
        """处理单个批次"""
        # 这里可以实现批处理逻辑
        # 例如，批量API调用、批量数据库操作等
        return [task.execute() for task in batch]

    def get_batch_statistics(self):
        """获取批处理统计信息"""
        if not self.processed_batches:
            return {}

        total_batches = len(self.processed_batches)
        avg_batch_size = sum(b['batch_size'] for b in self.processed_batches) / total_batches
        avg_results = sum(b['results_count'] for b in self.processed_batches) / total_batches

        return {
            'total_batches': total_batches,
            'average_batch_size': avg_batch_size,
            'average_results_per_batch': avg_results,
            'processing_efficiency': avg_results / avg_batch_size if avg_batch_size > 0 else 0
        }
```

### 2. 内存优化

#### 2.1 智能缓存系统

```python
import pickle
import os
import hashlib
import json
from datetime import datetime, timedelta
from typing import Any, Optional, Dict, List

class SmartCache:
    def __init__(self, cache_dir: str = "cache", max_size: int = 1000, ttl_hours: int = 24):
        self.cache_dir = cache_dir
        self.max_size = max_size
        self.ttl = timedelta(hours=ttl_hours)
        self.memory_cache = {}
        self.access_stats = {}

        # 创建缓存目录
        os.makedirs(cache_dir, exist_ok=True)

        # 启动清理线程
        self._start_cleanup_thread()

    def get(self, key: str) -> Optional[Any]:
        """获取缓存数据"""
        cache_key = self._generate_cache_key(key)

        # 检查内存缓存
        if cache_key in self.memory_cache:
            cached_item = self.memory_cache[cache_key]
            if self._is_valid(cached_item):
                self._update_access_stats(cache_key, 'hit')
                return cached_item['data']
            else:
                # 过期数据清理
                del self.memory_cache[cache_key]

        # 检查文件缓存
        file_path = self._get_cache_file_path(cache_key)
        if os.path.exists(file_path):
            try:
                with open(file_path, 'rb') as f:
                    cached_item = pickle.load(f)

                if self._is_valid(cached_item):
                    # 加载到内存缓存
                    self.memory_cache[cache_key] = cached_item
                    self._update_access_stats(cache_key, 'hit')
                    return cached_item['data']
                else:
                    # 删除过期文件
                    os.remove(file_path)
            except Exception as e:
                self._update_access_stats(cache_key, 'error')
                return None

        self._update_access_stats(cache_key, 'miss')
        return None

    def set(self, key: str, value: Any, ttl: Optional[timedelta] = None):
        """设置缓存数据"""
        cache_key = self._generate_cache_key(key)
        expiry_time = datetime.now() + (ttl or self.ttl)

        cached_item = {
            'data': value,
            'expiry_time': expiry_time,
            'created_at': datetime.now(),
            'access_count': 0,
            'size': self._estimate_size(value)
        }

        # 检查缓存大小限制
        if self._should_evict():
            self._evict_items()

        # 存储到内存缓存
        self.memory_cache[cache_key] = cached_item

        # 存储到文件缓存
        self._save_to_file(cache_key, cached_item)

        self._update_access_stats(cache_key, 'set')

    def _generate_cache_key(self, key: str) -> str:
        """生成缓存键"""
        return hashlib.md5(key.encode()).hexdigest()

    def _get_cache_file_path(self, cache_key: str) -> str:
        """获取缓存文件路径"""
        return os.path.join(self.cache_dir, f"{cache_key}.cache")

    def _is_valid(self, cached_item: Dict) -> bool:
        """检查缓存项是否有效"""
        return datetime.now() < cached_item['expiry_time']

    def _should_evict(self) -> bool:
        """检查是否需要清理缓存"""
        return len(self.memory_cache) >= self.max_size

    def _evict_items(self):
        """清理缓存项"""
        # 按LRU策略清理
        sorted_items = sorted(
            self.memory_cache.items(),
            key=lambda x: x[1].get('access_count', 0)
        )

        # 清理25%的缓存项
        items_to_remove = len(sorted_items) // 4
        for i in range(items_to_remove):
            key = sorted_items[i][0]
            del self.memory_cache[key]

            # 删除对应的文件缓存
            file_path = self._get_cache_file_path(key)
            if os.path.exists(file_path):
                os.remove(file_path)

    def _save_to_file(self, cache_key: str, cached_item: Dict):
        """保存到文件"""
        file_path = self._get_cache_file_path(cache_key)
        try:
            with open(file_path, 'wb') as f:
                pickle.dump(cached_item, f)
        except Exception as e:
            print(f"Error saving cache to file: {e}")

    def _estimate_size(self, value: Any) -> int:
        """估算对象大小"""
        return len(pickle.dumps(value))

    def _update_access_stats(self, cache_key: str, action: str):
        """更新访问统计"""
        if cache_key not in self.access_stats:
            self.access_stats[cache_key] = {
                'hits': 0,
                'misses': 0,
                'errors': 0,
                'sets': 0
            }

        if action == 'hit':
            self.access_stats[cache_key]['hits'] += 1
            if cache_key in self.memory_cache:
                self.memory_cache[cache_key]['access_count'] += 1
        elif action == 'miss':
            self.access_stats[cache_key]['misses'] += 1
        elif action == 'error':
            self.access_stats[cache_key]['errors'] += 1
        elif action == 'set':
            self.access_stats[cache_key]['sets'] += 1

    def _start_cleanup_thread(self):
        """启动清理线程"""
        import threading
        import time

        def cleanup_worker():
            while True:
                time.sleep(3600)  # 每小时清理一次
                self._cleanup_expired_items()

        cleanup_thread = threading.Thread(target=cleanup_worker, daemon=True)
        cleanup_thread.start()

    def _cleanup_expired_items(self):
        """清理过期项目"""
        current_time = datetime.now()

        # 清理内存缓存
        expired_keys = [
            key for key, item in self.memory_cache.items()
            if current_time >= item['expiry_time']
        ]

        for key in expired_keys:
            del self.memory_cache[key]

        # 清理文件缓存
        for filename in os.listdir(self.cache_dir):
            if filename.endswith('.cache'):
                file_path = os.path.join(self.cache_dir, filename)
                try:
                    with open(file_path, 'rb') as f:
                        cached_item = pickle.load(f)
                    if current_time >= cached_item['expiry_time']:
                        os.remove(file_path)
                except Exception:
                    # 删除损坏的缓存文件
                    try:
                        os.remove(file_path)
                    except:
                        pass

    def get_cache_stats(self) -> Dict:
        """获取缓存统计信息"""
        total_requests = sum(
            stats['hits'] + stats['misses']
            for stats in self.access_stats.values()
        )

        total_hits = sum(stats['hits'] for stats in self.access_stats.values())

        hit_rate = (total_hits / total_requests * 100) if total_requests > 0 else 0

        return {
            'memory_cache_size': len(self.memory_cache),
            'file_cache_count': len([f for f in os.listdir(self.cache_dir) if f.endswith('.cache')]),
            'total_requests': total_requests,
            'hit_rate': f"{hit_rate:.2f}%",
            'cache_dir': self.cache_dir,
            'max_size': self.max_size
        }
```

#### 2.2 内存管理优化

```python
import psutil
import gc
import threading
from typing import Callable, Optional

class MemoryManager:
    def __init__(self, max_memory_percent: float = 80.0):
        self.max_memory_percent = max_memory_percent
        self.monitoring = False
        self.monitor_thread = None
        self.callbacks = []

    def start_monitoring(self):
        """启动内存监控"""
        if not self.monitoring:
            self.monitoring = True
            self.monitor_thread = threading.Thread(target=self._monitor_memory, daemon=True)
            self.monitor_thread.start()

    def stop_monitoring(self):
        """停止内存监控"""
        self.monitoring = False

    def add_memory_callback(self, callback: Callable[[float], None]):
        """添加内存回调函数"""
        self.callbacks.append(callback)

    def _monitor_memory(self):
        """监控内存使用"""
        while self.monitoring:
            try:
                memory_percent = psutil.virtual_memory().percent

                if memory_percent > self.max_memory_percent:
                    self._handle_high_memory(memory_percent)

                    # 调用回调函数
                    for callback in self.callbacks:
                        try:
                            callback(memory_percent)
                        except Exception as e:
                            print(f"Error in memory callback: {e}")

                time.sleep(5)  # 每5秒检查一次
            except Exception as e:
                print(f"Error monitoring memory: {e}")
                time.sleep(10)

    def _handle_high_memory(self, memory_percent: float):
        """处理高内存使用情况"""
        print(f"High memory usage detected: {memory_percent:.2f}%")

        # 执行垃圾回收
        gc.collect()

        # 清理不必要的缓存
        self._clear_caches()

        print(f"Memory cleanup completed. Current usage: {psutil.virtual_memory().percent:.2f}%")

    def _clear_caches(self):
        """清理缓存"""
        # 这里可以实现具体的缓存清理逻辑
        # 例如清理CrewAI的缓存、工具缓存等
        import crewai
        if hasattr(crewai, 'clear_cache'):
            crewai.clear_cache()

    def get_memory_info(self) -> Dict:
        """获取内存信息"""
        memory = psutil.virtual_memory()
        return {
            'total_memory': memory.total,
            'available_memory': memory.available,
            'used_memory': memory.used,
            'memory_percent': memory.percent,
            'max_memory_percent': self.max_memory_percent
        }
```

### 3. 网络优化

#### 3.1 连接池管理

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import threading
from typing import Dict, Any

class ConnectionPoolManager:
    def __init__(self, pool_connections: int = 10, pool_maxsize: int = 10):
        self.session = self._create_session(pool_connections, pool_maxsize)
        self.lock = threading.Lock()

    def _create_session(self, pool_connections: int, pool_maxsize: int) -> requests.Session:
        """创建带有连接池的会话"""
        session = requests.Session()

        # 配置重试策略
        retry_strategy = Retry(
            total=3,
            backoff_factor=0.1,
            status_forcelist=[429, 500, 502, 503, 504],
            allowed_methods=["HEAD", "GET", "OPTIONS", "POST", "PUT", "DELETE"]
        )

        # 创建适配器
        adapter = HTTPAdapter(
            max_retries=retry_strategy,
            pool_connections=pool_connections,
            pool_maxsize=pool_maxsize
        )

        # 为HTTP和HTTPS注册适配器
        session.mount("http://", adapter)
        session.mount("https://", adapter)

        return session

    def request(self, method: str, url: str, **kwargs) -> requests.Response:
        """发送请求"""
        with self.lock:
            try:
                response = self.session.request(method, url, **kwargs)
                response.raise_for_status()
                return response
            except requests.exceptions.RequestException as e:
                print(f"Request failed: {e}")
                raise

    def get(self, url: str, params: Dict[str, Any] = None) -> requests.Response:
        """GET请求"""
        return self.request('GET', url, params=params)

    def post(self, url: str, data: Dict[str, Any] = None, json: Dict[str, Any] = None) -> requests.Response:
        """POST请求"""
        return self.request('POST', url, data=data, json=json)

    def close(self):
        """关闭连接池"""
        self.session.close()
```

#### 3.2 请求缓存

```python
class RequestCache:
    def __init__(self, cache_manager: SmartCache, ttl_hours: int = 1):
        self.cache_manager = cache_manager
        self.ttl = timedelta(hours=ttl_hours)

    def get_cached_response(self, url: str, params: Dict[str, Any] = None) -> Optional[Dict]:
        """获取缓存的响应"""
        cache_key = self._generate_request_cache_key(url, params)
        return self.cache_manager.get(cache_key)

    def cache_response(self, url: str, params: Dict[str, Any], response: Dict):
        """缓存响应"""
        cache_key = self._generate_request_cache_key(url, params)
        self.cache_manager.set(cache_key, response, self.ttl)

    def _generate_request_cache_key(self, url: str, params: Dict[str, Any] = None) -> str:
        """生成请求缓存键"""
        key_data = {'url': url}
        if params:
            key_data['params'] = params
        return json.dumps(key_data, sort_keys=True)
```

### 4. 数据库优化

#### 4.1 批量数据库操作

```python
import sqlite3
import threading
from typing import List, Dict, Any, Optional
from contextlib import contextmanager

class DatabaseOptimizer:
    def __init__(self, db_path: str):
        self.db_path = db_path
        self.connection_pool = []
        self.pool_lock = threading.Lock()
        self.max_pool_size = 5

        # 初始化连接池
        self._initialize_pool()

    def _initialize_pool(self):
        """初始化连接池"""
        for _ in range(self.max_pool_size):
            conn = sqlite3.connect(self.db_path, check_same_thread=False)
            self.connection_pool.append(conn)

    @contextmanager
    def get_connection(self):
        """获取数据库连接"""
        conn = None
        try:
            with self.pool_lock:
                if self.connection_pool:
                    conn = self.connection_pool.pop()
                else:
                    conn = sqlite3.connect(self.db_path, check_same_thread=False)

            yield conn
        finally:
            if conn:
                with self.pool_lock:
                    if len(self.connection_pool) < self.max_pool_size:
                        self.connection_pool.append(conn)
                    else:
                        conn.close()

    def batch_insert(self, table: str, data: List[Dict[str, Any]]) -> bool:
        """批量插入数据"""
        if not data:
            return True

        with self.get_connection() as conn:
            try:
                cursor = conn.cursor()

                # 构建插入语句
                columns = list(data[0].keys())
                placeholders = ', '.join(['?' for _ in columns])
                sql = f"INSERT INTO {table} ({', '.join(columns)}) VALUES ({placeholders})"

                # 准备数据
                values = [tuple(item[col] for col in columns) for item in data]

                # 执行批量插入
                cursor.executemany(sql, values)
                conn.commit()

                return True
            except Exception as e:
                conn.rollback()
                print(f"Batch insert failed: {e}")
                return False

    def batch_update(self, table: str, updates: List[Dict[str, Any]], condition_column: str) -> bool:
        """批量更新数据"""
        if not updates:
            return True

        with self.get_connection() as conn:
            try:
                cursor = conn.cursor()

                for item in updates:
                    # 构建更新语句
                    set_clause = ', '.join([f"{k} = ?" for k in item.keys() if k != condition_column])
                    condition_value = item[condition_column]
                    values = [v for k, v in item.items() if k != condition_column] + [condition_value]

                    sql = f"UPDATE {table} SET {set_clause} WHERE {condition_column} = ?"
                    cursor.execute(sql, values)

                conn.commit()
                return True
            except Exception as e:
                conn.rollback()
                print(f"Batch update failed: {e}")
                return False

    def optimize_database(self):
        """优化数据库"""
        with self.get_connection() as conn:
            try:
                cursor = conn.cursor()

                # 分析数据库
                cursor.execute("ANALYZE")

                # 重建索引
                cursor.execute("REINDEX")

                # 清理数据库
                cursor.execute("VACUUM")

                conn.commit()
                print("Database optimization completed")
            except Exception as e:
                print(f"Database optimization failed: {e}")
```

## 最佳实践

### 1. 智能体设计最佳实践

#### 1.1 角色定义优化

```python
class OptimizedAgent(Agent):
    def __init__(self, *args, **kwargs):
        # 设置合理的默认值
        kwargs.setdefault('max_iter', 25)
        kwargs.setdefault('max_rpm', 60)
        kwargs.setdefault('memory', True)
        kwargs.setdefault('verbose', False)

        super().__init__(*args, **kwargs)

        # 性能优化配置
        self._setup_performance_optimizations()

    def _setup_performance_optimizations(self):
        """设置性能优化"""
        # 启用结果缓存
        self.enable_result_caching = True
        self.cache_ttl = timedelta(hours=1)

        # 配置工具使用策略
        self.tool_usage_strategy = 'lazy'  # 'eager', 'lazy', 'adaptive'

        # 设置内存限制
        self.max_memory_usage = 100 * 1024 * 1024  # 100MB

    def optimize_task_execution(self, task):
        """优化任务执行"""
        if self.enable_result_caching:
            cache_key = f"{self.role}_{task.description}_{hash(str(task.context))}"
            cached_result = self._get_cached_result(cache_key)
            if cached_result:
                return cached_result

        # 执行任务
        result = super().execute_task(task)

        # 缓存结果
        if self.enable_result_caching:
            self._cache_result(cache_key, result)

        return result

    def _get_cached_result(self, cache_key):
        """获取缓存结果"""
        # 实现缓存逻辑
        pass

    def _cache_result(self, cache_key, result):
        """缓存结果"""
        # 实现缓存逻辑
        pass
```

#### 1.2 工具使用优化

```python
class OptimizedTool(BaseTool):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.usage_stats = {
            'total_calls': 0,
            'successful_calls': 0,
            'failed_calls': 0,
            'average_execution_time': 0
        }
        self.cache = {}
        self.rate_limiter = RateLimiter(calls_per_minute=30)

    def _run(self, **kwargs):
        """优化的工具执行"""
        start_time = time.time()

        # 检查速率限制
        if not self.rate_limiter.can_execute():
            raise Exception("Rate limit exceeded")

        # 检查缓存
        cache_key = self._generate_cache_key(kwargs)
        if cache_key in self.cache:
            return self.cache[cache_key]

        try:
            # 执行实际逻辑
            result = self.execute_tool(**kwargs)

            # 更新统计信息
            self.usage_stats['total_calls'] += 1
            self.usage_stats['successful_calls'] += 1

            # 缓存结果
            self.cache[cache_key] = result

            # 更新执行时间
            execution_time = time.time() - start_time
            self._update_average_execution_time(execution_time)

            return result

        except Exception as e:
            self.usage_stats['total_calls'] += 1
            self.usage_stats['failed_calls'] += 1
            raise e

    def execute_tool(self, **kwargs):
        """实际工具执行逻辑"""
        raise NotImplementedError

    def _generate_cache_key(self, kwargs):
        """生成缓存键"""
        return hashlib.md5(str(sorted(kwargs.items())).encode()).hexdigest()

    def _update_average_execution_time(self, execution_time):
        """更新平均执行时间"""
        total_calls = self.usage_stats['total_calls']
        current_avg = self.usage_stats['average_execution_time']
        new_avg = (current_avg * (total_calls - 1) + execution_time) / total_calls
        self.usage_stats['average_execution_time'] = new_avg

    def get_usage_stats(self):
        """获取使用统计"""
        success_rate = (self.usage_stats['successful_calls'] /
                       self.usage_stats['total_calls'] * 100) if self.usage_stats['total_calls'] > 0 else 0

        return {
            **self.usage_stats,
            'success_rate': f"{success_rate:.2f}%",
            'cache_size': len(self.cache)
        }
```

### 2. 任务管理最佳实践

#### 2.1 任务依赖优化

```python
class TaskDependencyOptimizer:
    def __init__(self, tasks: List[Task]):
        self.tasks = tasks
        self.dependency_graph = self._build_dependency_graph()

    def _build_dependency_graph(self):
        """构建任务依赖图"""
        graph = {}
        for task in self.tasks:
            graph[task] = task.context or []
        return graph

    def optimize_execution_order(self):
        """优化执行顺序"""
        # 拓扑排序
        sorted_tasks = self._topological_sort()

        # 识别并行任务
        parallel_groups = self._identify_parallel_groups(sorted_tasks)

        return parallel_groups

    def _topological_sort(self):
        """拓扑排序"""
        in_degree = {task: 0 for task in self.tasks}
        for task in self.tasks:
            for dependency in self.dependency_graph[task]:
                if dependency in in_degree:
                    in_degree[dependency] += 1

        queue = [task for task, degree in in_degree.items() if degree == 0]
        result = []

        while queue:
            current = queue.pop(0)
            result.append(current)

            for neighbor in self.tasks:
                if current in self.dependency_graph[neighbor]:
                    in_degree[neighbor] -= 1
                    if in_degree[neighbor] == 0:
                        queue.append(neighbor)

        return result

    def _identify_parallel_groups(self, sorted_tasks):
        """识别并行任务组"""
        groups = []
        current_group = []
        completed_tasks = set()

        for task in sorted_tasks:
            dependencies_met = all(
                dep in completed_tasks for dep in self.dependency_graph[task]
            )

            if dependencies_met:
                current_group.append(task)
            else:
                if current_group:
                    groups.append(current_group)
                    completed_tasks.update(current_group)
                    current_group = [task]
                else:
                    current_group.append(task)

        if current_group:
            groups.append(current_group)

        return groups
```

### 3. 监控和日志最佳实践

#### 3.1 性能监控

```python
import logging
from datetime import datetime
from typing import Dict, Any, List

class PerformanceMonitor:
    def __init__(self, log_file: str = "performance.log"):
        self.logger = logging.getLogger("PerformanceMonitor")
        self.logger.setLevel(logging.INFO)

        # 配置日志
        handler = logging.FileHandler(log_file)
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)

        self.metrics = {
            'task_execution_times': [],
            'agent_performance': {},
            'system_resources': [],
            'error_rates': []
        }

    def log_task_execution(self, task: Task, execution_time: float, success: bool):
        """记录任务执行"""
        self.metrics['task_execution_times'].append({
            'task': task.description,
            'agent': task.agent.role,
            'execution_time': execution_time,
            'success': success,
            'timestamp': datetime.now()
        })

        if not success:
            self.metrics['error_rates'].append({
                'task': task.description,
                'agent': task.agent.role,
                'timestamp': datetime.now()
            })

        self.logger.info(
            f"Task '{task.description}' executed by {task.agent.role} "
            f"in {execution_time:.2f}s - {'Success' if success else 'Failed'}"
        )

    def log_agent_performance(self, agent: Agent, tasks_completed: int,
                             total_tasks: int, avg_execution_time: float):
        """记录智能体性能"""
        agent_key = agent.role
        if agent_key not in self.metrics['agent_performance']:
            self.metrics['agent_performance'][agent_key] = []

        performance_data = {
            'tasks_completed': tasks_completed,
            'total_tasks': total_tasks,
            'completion_rate': tasks_completed / total_tasks if total_tasks > 0 else 0,
            'avg_execution_time': avg_execution_time,
            'timestamp': datetime.now()
        }

        self.metrics['agent_performance'][agent_key].append(performance_data)

        self.logger.info(
            f"Agent {agent.role}: {tasks_completed}/{total_tasks} tasks "
            f"completed ({performance_data['completion_rate']:.2%}), "
            f"avg time: {avg_execution_time:.2f}s"
        )

    def log_system_resources(self):
        """记录系统资源使用"""
        import psutil

        memory = psutil.virtual_memory()
        cpu = psutil.cpu_percent()

        resource_data = {
            'memory_percent': memory.percent,
            'cpu_percent': cpu,
            'available_memory': memory.available,
            'timestamp': datetime.now()
        }

        self.metrics['system_resources'].append(resource_data)

        self.logger.info(
            f"System Resources - Memory: {memory.percent:.1f}%, "
            f"CPU: {cpu:.1f}%, Available: {memory.available / 1024 / 1024:.1f}MB"
        )

    def generate_performance_report(self) -> Dict[str, Any]:
        """生成性能报告"""
        report = {
            'summary': {},
            'task_analysis': {},
            'agent_analysis': {},
            'system_analysis': {},
            'recommendations': []
        }

        # 任务执行分析
        if self.metrics['task_execution_times']:
            total_tasks = len(self.metrics['task_execution_times'])
            successful_tasks = sum(1 for task in self.metrics['task_execution_times'] if task['success'])
            avg_execution_time = sum(task['execution_time'] for task in self.metrics['task_execution_times']) / total_tasks

            report['task_analysis'] = {
                'total_tasks': total_tasks,
                'successful_tasks': successful_tasks,
                'success_rate': successful_tasks / total_tasks if total_tasks > 0 else 0,
                'average_execution_time': avg_execution_time
            }

        # 智能体性能分析
        for agent_role, performances in self.metrics['agent_performance'].items():
            if performances:
                latest = performances[-1]
                report['agent_analysis'][agent_role] = {
                    'completion_rate': latest['completion_rate'],
                    'avg_execution_time': latest['avg_execution_time'],
                    'tasks_completed': latest['tasks_completed']
                }

        # 系统资源分析
        if self.metrics['system_resources']:
            latest_resources = self.metrics['system_resources'][-1]
            report['system_analysis'] = {
                'current_memory_usage': latest_resources['memory_percent'],
                'current_cpu_usage': latest_resources['cpu_percent'],
                'available_memory': latest_resources['available_memory']
            }

        # 生成建议
        report['recommendations'] = self._generate_recommendations(report)

        return report

    def _generate_recommendations(self, report: Dict[str, Any]) -> List[str]:
        """生成优化建议"""
        recommendations = []

        # 基于任务执行的建议
        if 'task_analysis' in report:
            if report['task_analysis'].get('success_rate', 1.0) < 0.9:
                recommendations.append("任务失败率较高，建议检查错误处理逻辑")

            if report['task_analysis'].get('average_execution_time', 0) > 30:
                recommendations.append("任务执行时间较长，建议优化算法或启用并行处理")

        # 基于智能体性能的建议
        for agent_role, stats in report.get('agent_analysis', {}).items():
            if stats.get('completion_rate', 1.0) < 0.8:
                recommendations.append(f"智能体 {agent_role} 完成率较低，建议优化其配置")

            if stats.get('avg_execution_time', 0) > 60:
                recommendations.append(f"智能体 {agent_role} 执行时间较长，建议减少其工作负载")

        # 基于系统资源的建议
        if 'system_analysis' in report:
            if report['system_analysis'].get('current_memory_usage', 0) > 80:
                recommendations.append("内存使用率较高，建议优化内存管理或增加系统内存")

            if report['system_analysis'].get('current_cpu_usage', 0) > 70:
                recommendations.append("CPU使用率较高，建议优化算法或减少并发任务")

        return recommendations
```

## 总结

通过本文的详细介绍，我们了解了CrewAI的性能优化策略和最佳实践：

1. **异步执行优化**：通过并行处理和批处理提高执行效率
2. **内存优化**：实现智能缓存和内存管理
3. **网络优化**：使用连接池和请求缓存
4. **数据库优化**：批量操作和连接池管理
5. **智能体设计优化**：合理的角色定义和工具使用策略
6. **任务管理优化**：依赖关系分析和执行顺序优化
7. **监控和日志**：全面的性能监控和分析

这些优化策略可以显著提高CrewAI应用的性能和稳定性。在实际应用中，需要根据具体的业务场景和需求，选择合适的优化方案。

在下一篇文章中，我们将探讨CrewAI的高级特性，包括自定义工具、自定义流程等。

---

**系列预告：**
- 第六篇：CrewAI高级特性：自定义工具和流程
- 第七篇：CrewAI在企业的实际应用案例
- 第八篇：CrewAI与其他多智能体框架的深度对比

感谢关注我们的CrewAI深度解析系列！