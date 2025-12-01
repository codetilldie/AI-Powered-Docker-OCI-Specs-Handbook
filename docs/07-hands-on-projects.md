# 第七章：动手实战项目 (Chapter 7: Hands-on Projects)

> 通过三个完整项目，将理论知识转化为实践能力，构建自己的容器工具集

---

理论的价值在于实践。本章将通过三个循序渐进的项目,带你亲手构建容器生态系统的核心组件：一个迷你容器运行时、一个 AI 驱动的 Dockerfile 优化工具,以及一个简单的 OCI Registry。每个项目都附有完整代码和详细讲解。

## 7.1 Project A：用 Go 语言实现一个迷你容器 Runtime

### 7.1.1 项目目标

实现一个最小化的容器运行时,支持：
- ✅ Namespace 隔离（PID、MNT、UTS、NET）
- ✅ Cgroups 资源限制（CPU、内存）
- ✅ OverlayFS 文件系统
- ✅ 简单的命令行接口

### 7.1.2 项目结构

```
mini-container/
├── main.go              # 入口文件
├── container/
│   ├── namespace.go     # Namespace 管理
│   ├── cgroup.go        # Cgroups 管理
│   └── rootfs.go        # 文件系统管理
├── cmd/
│   ├── run.go           # run 命令
│   └── exec.go          # exec 命令
└── go.mod
```

### 7.1.3 核心代码实现

#### main.go - 命令行入口

```go
package main

import (
    "fmt"
    "os"
    
    "github.com/urfave/cli/v2"
)

func main() {
    app := &cli.App{
        Name:  "mini-container",
        Usage: "A minimalist container runtime",
        Commands: []*cli.Command{
            {
                Name:  "run",
                Usage: "Run a command in a new container",
                Flags: []cli.Flag{
                    &cli.StringFlag{
                        Name:  "mem",
                        Usage: "Memory limit (e.g., 512m)",
                        Value: "512m",
                    },
                    &cli.Float64Flag{
                        Name:  "cpus",
                        Usage: "CPU limit (e.g., 1.5)",
                        Value: 1.0,
                    },
                    &cli.StringFlag{
                        Name:  "rootfs",
                        Usage: "Root filesystem path",
                        Value: "/tmp/rootfs",
                    },
                },
                Action: runContainer,
            },
        },
    }
    
    if err := app.Run(os.Args); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

#### container/namespace.go - Namespace 隔离

```go
package container

import (
    "os"
    "os/exec"
    "syscall"
)

func NewContainer(config *Config) error {
    // 1. 创建子进程，设置 Namespace
    cmd := exec.Command("/proc/self/exe", "child", config.Command...)
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWPID |   // PID Namespace
                    syscall.CLONE_NEWNET |  // Network Namespace
                    syscall.CLONE_NEWNS |   // Mount Namespace
                    syscall.CLONE_NEWUTS |  // UTS Namespace
                    syscall.CLONE_NEWIPC,   // IPC Namespace
        Unshareflags: syscall.CLONE_NEWNS,
    }
    
    // 2. 设置 Cgroups
    if err := setupCgroups(config); err != nil {
        return err
    }
    
    // 3. 启动容器
    if err := cmd.Start(); err != nil {
        return err
    }
    
    // 4. 等待容器结束
    return cmd.Wait()
}

func RunChild(config *Config) error {
    fmt.Printf("Container PID: %d\n", os.Getpid())
    
    // 1. 设置主机名
    if err := syscall.Sethostname([]byte("container")); err != nil {
        return err
    }
    
    // 2. 设置 RootFS（OverlayFS）
    if err := setupRootFS(config.RootFS); err != nil {
        return err
    }
    
    // 3. chroot 到新根目录
    if err := syscall.Chroot(config.RootFS); err != nil {
        return err
    }
    
    // 4. 切换工作目录
    if err := os.Chdir("/"); err != nil {
        return err
    }
    
    // 5. 重新挂载 /proc
    if err := mountProc(); err != nil {
        return err
    }
    
    // 6. 执行用户命令
    argv := config.Command
    if len(argv) == 0 {
        argv = []string{"/bin/sh"}
    }
    
    return syscall.Exec(argv[0], argv, os.Environ())
}
```

#### container/cgroup.go - Cgroups 资源限制

```go
package container

import (
    "fmt"
    "io/ioutil"
    "os"
    "path/filepath"
    "strconv"
)

const cgroupRoot = "/sys/fs/cgroup"

func setupCgroups(config *Config) error {
    containerID := generateContainerID()
    
    // 1. 创建 Cgroup 目录
    cgroupPath := filepath.Join(cgroupRoot, "mini-container", containerID)
    
    // 2. 限制内存
    if err := limitMemory(cgroupPath, config.MemoryLimit); err != nil {
        return err
    }
    
    // 3. 限制 CPU
    if err := limitCPU(cgroupPath, config.CPULimit); err != nil {
        return err
    }
    
    // 4. 将当前进程加入 Cgroup
    procsFile := filepath.Join(cgroupPath, "cgroup.procs")
    return ioutil.WriteFile(procsFile, []byte(strconv.Itoa(os.Getpid())), 0644)
}

func limitMemory(cgroupPath string, limitMB int) error {
    memoryPath := filepath.Join(cgroupPath, "memory")
    if err := os.MkdirAll(memoryPath, 0755); err != nil {
        return err
    }
    
    // 设置内存限制（字节）
    limitBytes := limitMB * 1024 * 1024
    limitFile := filepath.Join(memoryPath, "memory.limit_in_bytes")
    
    return ioutil.WriteFile(limitFile, []byte(strconv.Itoa(limitBytes)), 0644)
}

func limitCPU(cgroupPath string, cpus float64) error {
    cpuPath := filepath.Join(cgroupPath, "cpu")
    if err := os.MkdirAll(cpuPath, 0755); err != nil {
        return err
    }
    
    // CFS (Completely Fair Scheduler) 配置
    // quota / period = CPU 核心数
    period := 100000  // 100ms
    quota := int(cpus * float64(period))
    
    quotaFile := filepath.Join(cpuPath, "cpu.cfs_quota_us")
    periodFile := filepath.Join(cpuPath, "cpu.cfs_period_us")
    
    if err := ioutil.WriteFile(quotaFile, []byte(strconv.Itoa(quota)), 0644); err != nil {
        return err
    }
    
    return ioutil.WriteFile(periodFile, []byte(strconv.Itoa(period)), 0644)
}
```

#### container/rootfs.go - OverlayFS 文件系统

```go
package container

import (
    "fmt"
    "os"
    "path/filepath"
    "syscall"
)

func setupRootFS(rootfs string) error {
    // 1. 创建 OverlayFS 所需目录
    lower := filepath.Join(rootfs, "lower")
    upper := filepath.Join(rootfs, "upper")
    work := filepath.Join(rootfs, "work")
    merged := filepath.Join(rootfs, "merged")
    
    for _, dir := range []string{lower, upper, work, merged} {
        if err := os.MkdirAll(dir, 0755); err != nil {
            return err
        }
    }
    
    // 2. 挂载 OverlayFS
    opts := fmt.Sprintf("lowerdir=%s,upperdir=%s,workdir=%s", lower, upper, work)
    
    return syscall.Mount("overlay", merged, "overlay", 0, opts)
}

func mountProc() error {
    // 挂载 /proc 文件系统
    procPath := "/proc"
    
    // 确保目录存在
    if err := os.MkdirAll(procPath, 0755); err != nil {
        return err
    }
    
    // 挂载 proc
    return syscall.Mount("proc", procPath, "proc", 0, "")
}
```

### 7.1.4 使用示例

```bash
# 1. 准备 rootfs（使用 Docker 导出）
docker export $(docker create alpine) | tar -C /tmp/rootfs -xvf -

# 2. 编译项目
go build -o mini-container

# 3. 运行容器（需要 root）
sudo ./mini-container run --mem 512m --cpus 0.5 /bin/sh

# 在容器内
$ hostname
container

$ cat /proc/self/cgroup
# 可以看到 Cgroup 配置

$ ps aux
PID   USER     COMMAND
1     root     /bin/sh
```

### 7.1.5 扩展练习

1. **网络支持**：使用 veth pair 连接容器网络
2. **镜像管理**：支持从 OCI 镜像启动
3. **日志收集**：捕获容器 stdout/stderr

---

## 7.2 Project B：构建基于 AI 的 Dockerfile 优化工具

### 7.2.1 项目目标

创建一个 CLI 工具,自动分析和优化 Dockerfile：
- ✅ 检测安全问题
- ✅ 优化镜像体积
- ✅ 生成优化建议
- ✅ 自动重写 Dockerfile

### 7.2.2 技术栈

- **Python 3.11+**
- **OpenAI API** (GPT-4)
- **Click**（CLI 框架）
- **Docker SDK**

### 7.2.3 项目结构

```
dockerfile-optimizer/
├── optimizer/
│   ├── __init__.py
│   ├── analyzer.py      # Dockerfile 分析
│   ├── ai_engine.py     # AI 引擎
│   └── generator.py     # 优化建议生成
├── cli.py               # 命令行入口
├── config.yaml          # 配置文件
└── tests/
```

### 7.2.4 核心代码

#### analyzer.py - Dockerfile 分析器

```python
import re
from dataclasses import dataclass
from typing import List

@dataclass
class DockerfileIssue:
    severity: str  # Critical, High, Medium, Low
    line_number: int
    category: str  # Security, Performance, Best Practice
    description: str
    suggestion: str

class DockerfileAnalyzer:
    def __init__(self, dockerfile_content: str):
        self.lines = dockerfile_content.split('\n')
        self.issues = []
        
    def analyze(self) -> List[DockerfileIssue]:
        self.check_base_image()
        self.check_user()
        self.check_secrets()
        self.check_layer_optimization()
        self.check_apt_cache()
        return self.issues
    
    def check_base_image(self):
        """检查基础镜像"""
        for i, line in enumerate(self.lines):
            if line.startswith('FROM'):
                if ':latest' in line:
                    self.issues.append(DockerfileIssue(
                        severity="High",
                        line_number=i + 1,
                        category="Best Practice",
                        description="使用 'latest' 标签导致构建不可重现",
                        suggestion="使用特定版本标签，如 'python:3.11-slim'"
                    ))
                
                if 'ubuntu' in line and 'slim' not in line:
                    self.issues.append(DockerfileIssue(
                        severity="Medium",
                        line_number=i + 1,
                        category="Performance",
                        description="基础镜像体积大",
                        suggestion="考虑使用 Alpine 或 Distroless 镜像"
                    ))
    
    def check_user(self):
        """检查是否以 root 运行"""
        has_user_directive = any('USER' in line for line in self.lines)
        
        if not has_user_directive:
            self.issues.append(DockerfileIssue(
                severity="Critical",
                line_number=len(self.lines),
                category="Security",
                description="容器以 root 用户运行",
                suggestion="添加:\nRUN useradd -m -u 1000 appuser\nUSER appuser"
            ))
    
    def check_secrets(self):
        """检查敏感信息泄露"""
        patterns = {
            r'(PASSWORD|SECRET|API_KEY)\s*=': "可能的密钥硬编码",
            r'(aws_access_key|PRIVATE_KEY)': "AWS 密钥泄露风险",
        }
        
        for i, line in enumerate(self.lines):
            for pattern, desc in patterns.items():
                if re.search(pattern, line, re.I):
                    self.issues.append(DockerfileIssue(
                        severity="Critical",
                        line_number=i + 1,
                        category="Security",
                        description=desc,
                        suggestion="使用 Docker secrets 或环境变量文件"
                    ))
    
    def check_layer_optimization(self):
        """检查层优化"""
        run_count = sum(1 for line in self.lines if line.startswith('RUN'))
        
        if run_count > 5:
            self.issues.append(DockerfileIssue(
                severity="Medium",
                line_number=0,
                category="Performance",
                description=f"过多的 RUN 指令 ({run_count}个)",
                suggestion="合并相关的 RUN 指令以减少镜像层数"
            ))
    
    def check_apt_cache(self):
        """检查 apt 缓存清理"""
        for i, line in enumerate(self.lines):
            if 'apt-get install' in line:
                # 检查同一行或后续行是否清理缓存
                next_lines = '\n'.join(self.lines[i:i+3])
                if 'rm -rf /var/lib/apt/lists/*' not in next_lines:
                    self.issues.append(DockerfileIssue(
                        severity="Medium",
                        line_number=i + 1,
                        category="Performance",
                        description="apt-get 未清理缓存",
                        suggestion="在同一个 RUN 中添加: && rm -rf /var/lib/apt/lists/*"
                    ))
```

#### ai_engine.py - AI 优化引擎

```python
import openai
from typing import List

class AIOptimizer:
    def __init__(self, api_key: str):
        openai.api_key = api_key
        
    def generate_optimized_dockerfile(
        self, 
        original: str, 
        issues: List[DockerfileIssue]
    ) -> str:
        """使用 AI 生成优化后的 Dockerfile"""
        
        issues_summary = "\n".join([
            f"- [{issue.severity}] Line {issue.line_number}: {issue.description}"
            for issue in issues
        ])
        
        prompt = f"""
你是一个 Dockerfile 优化专家。请根据以下问题重写 Dockerfile。

原始 Dockerfile:
```dockerfile
{original}
```

已发现的问题:
{issues_summary}

要求:
1. 修复所有 Critical 和 High 级别问题
2. 优化镜像体积（使用多阶段构建、精简基础镜像）
3. 遵循最佳实践（非 root 用户、健康检查、最小权限）
4. 保持功能不变
5. 添加必要的注释

请只输出优化后的 Dockerfile，不要任何解释。
"""
        
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "你是一个 Dockerfile 优化专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3,
        )
        
        return response.choices[0].message.content.strip()
    
    def explain_optimization(self, original: str, optimized: str) -> str:
        """解释优化原因"""
        prompt = f"""
对比以下两个 Dockerfile，解释优化后的版本做了哪些改进：

原始版本:
{original}

优化版本:
{optimized}

请列出关键改进点（最多 5 条），每条包括：
1. 改进内容
2. 带来的好处
3. 潜在的注意事项（如有）

以 Markdown 格式输出。
"""
        
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5,
        )
        
        return response.choices[0].message.content
```

#### cli.py - 命令行接口

```python
import click
import os
from pathlib import Path
from .analyzer import DockerfileAnalyzer
from .ai_engine import AIOptimizer

@click.group()
def cli():
    """AI-powered Dockerfile Optimizer"""
    pass

@cli.command()
@click.argument('dockerfile', type=click.Path(exists=True))
@click.option('--output', '-o', help='Output file for optimized Dockerfile')
@click.option('--auto-fix', is_flag=True, help='Automatically apply fixes')
def analyze(dockerfile, output, auto_fix):
    """Analyze and optimize a Dockerfile"""
    
    # 1. 读取 Dockerfile
    with open(dockerfile) as f:
        content = f.read()
    
    # 2. 分析问题
    analyzer = DockerfileAnalyzer(content)
    issues = analyzer.analyze()
    
    # 3. 显示问题
    click.echo(f"\n📋 Found {len(issues)} issues:\n")
    for issue in sorted(issues, key=lambda x: x.severity):
        color = {
            "Critical": "red",
            "High": "yellow",
            "Medium": "blue",
            "Low": "white"
        }[issue.severity]
        
        click.secho(f"[{issue.severity}] Line {issue.line_number}", fg=color, bold=True)
        click.echo(f"  Category: {issue.category}")
        click.echo(f"  Issue: {issue.description}")
        click.echo(f"  Suggestion: {issue.suggestion}\n")
    
    # 4. AI 优化（如果启用）
    if auto_fix:
        api_key = os.getenv('OPENAI_API_KEY')
        if not api_key:
            click.secho("Error: OPENAI_API_KEY not set", fg="red")
            return
        
        click.echo("🤖 Generating optimized Dockerfile with AI...")
        
        optimizer = AIOptimizer(api_key)
        optimized = optimizer.generate_optimized_dockerfile(content, issues)
        
        # 5. 保存优化结果
        output_path = output or f"{dockerfile}.optimized"
        with open(output_path, 'w') as f:
            f.write(optimized)
        
        click.secho(f"\n✅ Optimized Dockerfile saved to: {output_path}", fg="green")
        
        # 6. 显示优化说明
        explanation = optimizer.explain_optimization(content, optimized)
        click.echo("\n📖 Optimization Explanation:\n")
        click.echo(explanation)

@cli.command()
@click.argument('dockerfile', type=click.Path(exists=True))
def benchmark(dockerfile):
    """Benchmark image size before/after optimization"""
    
    import docker
    client = docker.from_env()
    
    # 构建原始镜像
    click.echo("Building original image...")
    original_image, _ = client.images.build(path=".", tag="original:latest")
    original_size = original_image.attrs['Size'] / (1024 * 1024)  # MB
    
    # 构建优化后镜像
    click.echo("Building optimized image...")
    optimized_image, _ = client.images.build(
        path=".",
        dockerfile=f"{dockerfile}.optimized",
        tag="optimized:latest"
    )
    optimized_size = optimized_image.attrs['Size'] / (1024 * 1024)  # MB
    
    # 对比结果
    reduction = ((original_size - optimized_size) / original_size) * 100
    
    click.echo("\n📊 Benchmark Results:\n")
    click.echo(f"Original Size:  {original_size:.2f} MB")
    click.echo(f"Optimized Size: {optimized_size:.2f} MB")
    click.secho(f"Reduction:      {reduction:.1f}%", fg="green", bold=True)

if __name__ == '__main__':
    cli()
```

### 7.2.5 使用示例

```bash
# 1. 安装依赖
pip install click openai docker

# 2. 设置 API Key
export OPENAI_API_KEY="sk-..."

# 3. 分析 Dockerfile
python -m optimizer.cli analyze Dockerfile

# 4. 自动优化
python -m optimizer.cli analyze Dockerfile --auto-fix

# 5. 对比镜像大小
python -m optimizer.cli benchmark Dockerfile
```

---

## 7.3 Project C：实现一个简单的 OCI Registry

### 7.3.1 项目目标

实现一个符合 OCI Distribution Spec 的最小化 Registry:
- ✅ 支持 Push/Pull 镜像
- ✅ Manifest 和 Blob 存储
- ✅ 基础认证
- ✅ RESTful API

### 7.3.2 技术栈

- **Go 1.21+**
- **Gin**（Web 框架）
- **Badger**（KV 数据库）

### 7.3.3 API 端点实现

#### main.go

```go
package main

import (
    "github.com/gin-gonic/gin"
    "mini-registry/handlers"
    "mini-registry/storage"
)

func main() {
    // 初始化存储
    store, err := storage.NewBadgerStore("./registry-data")
    if err != nil {
        panic(err)
    }
    defer store.Close()
    
    // 创建路由
    r := gin.Default()
    
    // 注册 API 端点
    v2 := r.Group("/v2")
    {
        // 版本检查
        v2.GET("/", handlers.CheckVersion)
        
        // Manifest 操作
        v2.GET("/:name/manifests/:reference", handlers.GetManifest(store))
        v2.PUT("/:name/manifests/:reference", handlers.PutManifest(store))
        v2.DELETE("/:name/manifests/:reference", handlers.DeleteManifest(store))
        
        // Blob 操作
        v2.HEAD("/:name/blobs/:digest", handlers.CheckBlob(store))
        v2.GET("/:name/blobs/:digest", handlers.GetBlob(store))
        
        // Blob 上传
        v2.POST("/:name/blobs/uploads/", handlers.StartBlobUpload(store))
        v2.PATCH("/:name/blobs/uploads/:uuid", handlers.UploadBlobChunk(store))
        v2.PUT("/:name/blobs/uploads/:uuid", handlers.FinishBlobUpload(store))
        
        // 目录
        v2.GET("/_catalog", handlers.GetCatalog(store))
        v2.GET("/:name/tags/list", handlers.GetTags(store))
    }
    
    r.Run(":5000")
}
```

#### handlers/manifest.go

```go
package handlers

import (
    "crypto/sha256"
    "fmt"
    "io/ioutil"
    "net/http"
    
    "github.com/gin-gonic/gin"
    "mini-registry/storage"
)

func GetManifest(store storage.Store) gin.HandlerFunc {
    return func(c *gin.Context) {
        name := c.Param("name")
        reference := c.Param("reference")
        
        // 从存储读取 Manifest
        key := fmt.Sprint("manifest:%s:%s", name, reference)
        data, err := store.Get(key)
        
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{
                "errors": []gin.H{
                    {"code": "MANIFEST_UNKNOWN", "message": "manifest unknown"},
                },
            })
            return
        }
        
        // 返回 Manifest
        c.Header("Content-Type", "application/vnd.oci.image.manifest.v1+json")
        c.Header("Docker-Content-Digest", calculateDigest(data))
        c.Data(http.StatusOK, "application/json", data)
    }
}

func PutManifest(store storage.Store) gin.HandlerFunc {
    return func(c *gin.Context) {
        name := c.Param("name")
        reference := c.Param("reference")
        
        // 读取 Manifest 数据
        data, err := ioutil.ReadAll(c.Request.Body)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        // 验证 Manifest 格式
        if !isValidManifest(data) {
            c.JSON(http.StatusBadRequest, gin.H{
                "errors": []gin.H{
                    {"code": "MANIFEST_INVALID"},
                },
            })
            return
        }
        
        // 存储 Manifest
        key := fmt.Sprintf("manifest:%s:%s", name, reference)
        digest := calculateDigest(data)
        
        if err := store.Put(key, data); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        
        // 返回 Digest
        c.Header("Docker-Content-Digest", digest)
        c.Status(http.StatusCreated)
    }
}

func calculateDigest(data []byte) string {
    hash := sha256.Sum256(data)
    return fmt.Sprintf("sha256:%x", hash)
}
```

#### handlers/blob.go

```go
package handlers

import (
    "fmt"
    "io"
    "net/http"
    
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

func GetBlob(store storage.Store) gin.HandlerFunc {
    return func(c *gin.Context) {
        digest := c.Param("digest")
        
        key := fmt.Sprintf("blob:%s", digest)
        data, err := store.Get(key)
        
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{
                "errors": []gin.H{
                    {"code": "BLOB_UNKNOWN"},
                },
            })
            return
        }
        
        c.Data(http.StatusOK, "application/octet-stream", data)
    }
}

func StartBlobUpload(store storage.Store) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 生成上传会话 ID
        uploadID := uuid.New().String()
        
        // 返回上传 URL
        location := fmt.Sprintf("/v2/%s/blobs/uploads/%s", c.Param("name"), uploadID)
        
        c.Header("Location", location)
        c.Header("Docker-Upload-UUID", uploadID)
        c.Status(http.StatusAccepted)
    }
}

func FinishBlobUpload(store storage.Store) gin.HandlerFunc {
    return func(c *gin.Context) {
        uploadID := c.Param("uuid")
        digest := c.Query("digest")
        
        // 读取上传数据
        data, err := io.ReadAll(c.Request.Body)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        
        // 验证 Digest
        actualDigest := calculateDigest(data)
        if digest != actualDigest {
            c.JSON(http.StatusBadRequest, gin.H{
                "errors": []gin.H{
                    {"code": "DIGEST_INVALID"},
                },
            })
            return
        }
        
        // 存储 Blob
        key := fmt.Sprintf("blob:%s", digest)
        if err := store.Put(key, data); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        
        c.Header("Docker-Content-Digest", digest)
        c.Status(http.StatusCreated)
    }
}
```

### 7.3.4 使用示例

```bash
# 1. 启动 Registry
go run main.go
# Listening on :5000

# 2. 推送镜像
docker tag nginx:latest localhost:5000/nginx:latest
docker push localhost:5000/nginx:latest

# 3. 拉取镜像
docker pull localhost:5000/nginx:latest

# 4. 查看目录
curl http://localhost:5000/v2/_catalog
# {"repositories":["nginx"]}
```

---

## 总结

通过三个实战项目,你已经：

1. **Project A**：理解了容器运行时的核心机制（Namespace、Cgroups、OverlayFS）
2. **Project B**：掌握了 AI 辅助开发的实践方法
3. **Project C**：实现了 OCI Distribution Spec 的基础功能

**下一步建议**：
- 🚀 将这些项目部署到生产环境
- 🔧 为项目添加更多功能（如网络、日志）
- 📚 深入研究 containerd、CRI-O 等生产级实现
- 🤝 参与开源社区（runc、OCI spec）

**恭喜你完成了整个手册的学习！**

---

**[<< 返回目录](../README.md)**

---

**贡献者欢迎**: 如果您对本章节有内容补充或建议，欢迎提交 PR 或 Issue！
