cat << 'EOF' > /tmp/auto_acme_alpn.sh
#!/usr/bin/env bash
set -euo pipefail

# ================= 配置区 =================
DOMAIN="renzhengdejuju.online"
EMAIL="3313270365@qq.com"
CERT_DIR="/etc/nginx/ssl"            # 证书存放目录
NGINX_CONF="/etc/nginx/nginx.conf"   # Nginx 主配置文件路径
# ==========================================

GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}>>> [1/6] 正在安装 acme.sh...${NC}"
if [ ! -f "$HOME/.acme.sh/acme.sh" ]; then
    curl -s https://get.acme.sh | sh -s email="$EMAIL"
fi
ACME="$HOME/.acme.sh/acme.sh"

echo -e "\n${GREEN}>>> [2/6] 正在备份并清理 Nginx 配置...${NC}"
sudo cp -a /etc/nginx "/etc/nginx.backup.$(date +%Y%m%d%H%M%S)"

# 自动修复你之前配置里的两个重复的空 443 server 块
# 通过正则匹配把那个只有 server_name 没有 ssl_certificate 的空块删掉
sudo python3 -c "
import re
with open('$NGINX_CONF', 'r') as f:
    content = f.read()
# 匹配空的 443 server 块并替换掉（解决之前的配置冲突）
content = re.sub(r'server\s*\{\s*listen\s+443\s+ssl;\s*server_name\s+[^;]+;\s*\}', '', content)
with open('$NGINX_CONF', 'w') as f:
    f.write(content)
"
echo "Nginx 配置清理完成。"

echo -e "\n${GREEN}>>> [3/6] 正在暂停 Nginx 以释放 443 端口...${NC}"
sudo systemctl stop nginx

echo -e "\n${GREEN}>>> [4/6] 正在通过 443 端口 (TLS-ALPN-01) 申请证书...${NC}"
# 设置默认 CA 为 Let's Encrypt
"$ACME" --set-default-ca --server letsencrypt

# 使用 alpn 模式申请（走 443）
if ! "$ACME" --issue --alpn -d "$DOMAIN" --standalone; then
    echo -e "${RED}证书申请失败！请检查 443 端口是否对公网开放（阿里云安全组）。${NC}"
    sudo systemctl start nginx
    exit 1
fi

echo -e "\n${GREEN}>>> [5/6] 安装证书并自动修改 Nginx 路径...${NC}"
sudo mkdir -p "$CERT_DIR"

# 安装证书，并设置续签成功后的重载命令
"$ACME" --install-cert -d "$DOMAIN" \
    --key-file       "$CERT_DIR/$DOMAIN.key" \
    --fullchain-file "$CERT_DIR/$DOMAIN.fullchain.cer" \
    --reloadcmd      "systemctl reload nginx"

# 自动修改 Nginx 配置里的证书路径（使用 sed 替换所有旧的 ssl_certificate 行）
sudo sed -i "s|ssl_certificate .*|ssl_certificate $CERT_DIR/$DOMAIN.fullchain.cer;|g" "$NGINX_CONF"
sudo sed -i "s|ssl_certificate_key .*|ssl_certificate_key $CERT_DIR/$DOMAIN.key;|g" "$NGINX_CONF"

echo "Nginx 证书路径已自动更新为 acme.sh 生成的证书。"

echo -e "\n${GREEN}>>> [6/6] 启动 Nginx 并验证...${NC}"
if sudo nginx -t; then
    sudo systemctl start nginx
    echo -e "${GREEN}Nginx 已成功启动！${NC}"
else
    echo -e "${RED}Nginx 配置测试失败，请手动检查 /etc/nginx/nginx.conf。${NC}"
    exit 1
fi

echo -e "\n${GREEN}========================================${NC}"
echo -e "${GREEN} 全自动闭环部署完毕！${NC}"
echo -e " 证书路径: $CERT_DIR/$DOMAIN.fullchain.cer"
echo -e " 自动续签: 已配置 (cron 任务)"
echo -e " 续签重载: 已配置 (reload nginx)"
echo -e " 你可以现在去浏览器测试: https://$DOMAIN"
echo -e "${GREEN}========================================${NC}"
EOF

# 执行脚本
sudo bash /tmp/auto_acme_alpn.sh
