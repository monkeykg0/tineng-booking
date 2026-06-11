# 服务器初始化手册（Ubuntu 24.04 / 阿里云 2c4g）

用途：从零初始化生产服务器，或灾难恢复时照此重建（目标 1 小时内恢复）。
域名：goumiaomu.cn（API：api.goumiaomu.cn，后台：admin.goumiaomu.cn）；服务器 IP 116.62.34.224。
前置：系统已重装为 Ubuntu 24.04 并完成 `apt update && apt upgrade`。

## 1. 创建部署用户 + SSH 加固

```bash
# 以 root 登录执行
adduser deploy
usermod -aG sudo deploy
```

本地电脑上把公钥传上去（没有密钥先 `ssh-keygen -t ed25519` 生成）：

```bash
ssh-copy-id deploy@116.62.34.224
ssh deploy@116.62.34.224   # 确认免密能登录后再继续！
```

回服务器禁用密码登录和 root 登录：

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

⚠️ 禁密码前必须先验证密钥登录成功，否则会把自己锁在外面。

阿里云控制台安全组：只放行 22、80、443 三个入方向端口。不装 ufw（和安全组双层防火墙容易锁死自己，KISS）。

## 2. 加 2G swap（4G 内存兜底）

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 3. 安装 Docker（阿里云镜像源）

官方源 download.docker.com 在国内被重置（curl 报 Connection reset by peer），用阿里云镜像源替代。阿里云 ECS 内网地址为 mirrors.cloud.aliyuncs.com（免流量）；非阿里云机器改用 mirrors.aliyun.com。

```bash
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL http://mirrors.cloud.aliyuncs.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  http://mirrors.cloud.aliyuncs.com/docker-ce/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker deploy   # 如未创建 deploy 用户（直接用 root）则跳过此行
docker run --rm hello-world      # 验证
```

⚠️ **国内服务器拉 Docker Hub 镜像必超时**（registry-1.docker.io 被墙），必须配置镜像加速器：

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run"
  ]
}
EOF
sudo systemctl restart docker
docker run --rm hello-world   # 重新验证
```

公共加速器可能失效。若都拉不动，去阿里云控制台 → 容器镜像服务 → 镜像工具 → 镜像加速器，拿到账号专属的 `https://xxxx.mirror.aliyuncs.com` 地址加到数组第一位，重启 docker 再试。

## 4. 域名解析

⚠️ goumiaomu.cn 由**另一个阿里云账号**持有，解析操作要登录那个账号：云解析 DNS → goumiaomu.cn → 解析设置 → 添加两条 A 记录（已有记录一律不动，旧机器业务不受影响）：

| 主机记录 | 类型 | 记录值 | TTL |
| --- | --- | --- | --- |
| api | A | 116.62.34.224 | 默认 |
| admin | A | 116.62.34.224 | 默认 |

验证：`ping api.goumiaomu.cn` 返回 116.62.34.224。

## 5. acme.sh 签证书（HTTP standalone 方式，无需域名账号的 AccessKey）

前提：第 4 步解析已生效；80 端口空闲（compose 尚未启动时即满足）。

```bash
curl https://get.acme.sh | sh -s email=你的邮箱
source ~/.bashrc

# pre/post hook：将来自动续期时临时停起 nginx 腾出 80 端口（每 60 天约几秒中断）
acme.sh --issue --standalone -d api.goumiaomu.cn \
  --pre-hook  "cd ~/tineng-booking/deploy && docker compose stop nginx || true" \
  --post-hook "cd ~/tineng-booking/deploy && docker compose start nginx || true"
acme.sh --issue --standalone -d admin.goumiaomu.cn \
  --pre-hook  "cd ~/tineng-booking/deploy && docker compose stop nginx || true" \
  --post-hook "cd ~/tineng-booking/deploy && docker compose start nginx || true"
```

（备选：若能在域名所在账号建 RAM 子用户并授权 AliyunDNSFullAccess，可改用 `--dns dns_ali` 方式，无 80 端口依赖。）

安装证书到部署目录（acme.sh 会记住此配置，每 60 天自动续期并执行 reloadcmd）：

```bash
mkdir -p ~/tineng-booking/deploy/certs
acme.sh --install-cert -d api.goumiaomu.cn \
  --key-file       ~/tineng-booking/deploy/certs/api.key \
  --fullchain-file ~/tineng-booking/deploy/certs/api.crt \
  --reloadcmd "cd ~/tineng-booking/deploy && docker compose restart nginx || true"
acme.sh --install-cert -d admin.goumiaomu.cn \
  --key-file       ~/tineng-booking/deploy/certs/admin.key \
  --fullchain-file ~/tineng-booking/deploy/certs/admin.crt \
  --reloadcmd "cd ~/tineng-booking/deploy && docker compose restart nginx || true"
```

## 6. 上传代码并部署

本地电脑，把项目同步到服务器（建好 GitHub 私有仓库后可改为服务器上 git pull）：

```bash
rsync -av --exclude node_modules --exclude .git \
  ~/Desktop/后端技能学习/tineng-booking/ deploy@116.62.34.224:~/tineng-booking/
```

服务器上：

```bash
cd ~/tineng-booking/deploy
cp .env.example .env && vim .env        # 改 POSTGRES_PASSWORD 为强密码
mkdir -p admin-dist                      # 后台还没构建，先建空目录占位
docker compose up -d --build
docker compose ps                        # 三个容器都应为 running/healthy
```

## 7. 全链路验证

```bash
curl https://api.goumiaomu.cn/healthz
# 期望输出：{"code":0,"message":"ok"}
```

通过即部署管道全通：域名 → HTTPS → Nginx → Go API → （PostgreSQL 已就绪）。

## 8. 收尾检查清单

- [ ] `ssh root@IP` 已被拒绝、密码登录已被拒绝
- [ ] 安全组只开 22/80/443，`docker compose ps` 确认 postgres 没有映射公网端口
- [ ] `free -h` 看到 2G swap
- [ ] `acme.sh --list` 显示两张证书及下次续期时间
- [ ] 重启服务器后 `docker compose ps` 三容器自动拉起（restart: unless-stopped）

## 后续（开始写业务后再做）

- 每日数据库备份到 OSS：cron 跑 `docker compose exec -T postgres pg_dump -U tineng tineng | gzip > 备份文件`，ossutil 上传，保留 30 天。
- GitHub Actions 自动部署（替代手动 rsync）。
