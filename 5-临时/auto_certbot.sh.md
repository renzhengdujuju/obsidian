cat << 'EOF' > /tmp/auto_certbot.sh
#!/usr/bin/env bash
set -euo pipefail

# ================= 配置区 =================
DOMAIN="renzhengdejuju.online"
EMAIL="3313270365@qq.com"
WEBROOT="/usr/share/nginx/html"
# ==========================================

GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}>>> [1/5] 环境预检...${NC}"
PUBLIC_IP=$(curl -s --max-time 5 https://api.ipify.org || echo "未知")
DOMAIN_IP=$(getent hosts "$DOMAIN" | awk '{ print $1 }' | head -n 1)
echo "服务器公网 IP: $PUBLIC_IP"
echo "域名解析 IP:   $DOMAIN_IP"
if [ "$PUBLIC_IP" != "$DOMAIN_IP" ]; then
    echo -e "${YELLOW}警告：IP 不一致，请确保域名没有接入未放行验证路径的 CDN/WAF。${NC}"
fi

echo -e "\n${GREEN}>>> [2/5] 备份 Nginx 配置...${NC}"
sudo cp -a /etc/nginx "/etc/nginx.backup.$(date +%Y%m%d%H%M%S)"

echo -e "\n${GREEN}>>> [3/5] 安装与修复 Certbot...${NC}"
if ! command -v snap &> /dev/null; then
    sudo dnf install -y snapd
    sudo systemctl enable --now snapd.socket
    [ ! -e /snap ] && sudo ln -s /var/lib/snapd/snap /snap
    sudo snap wait system seed.loaded || true
fi
sudo snap install core 2>/dev/null || true
sudo snap refresh core 2>/dev/null || true
sudo snap install --classic certbot 2>/dev/null || true
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
hash -r

# 检查插件
if ! sudo certbot plugins | grep -q "nginx"; then
    echo "尝试修复 nginx 插件..."
    sudo snap set certbot trust-plugin-with-root=ok 2>/dev/null || true
    sudo snap install certbot-nginx 2>/dev/null || true
fi

echo -e "\n${GREEN}>>> [4/5] 注入 Nginx 放行规则并重启...${NC}"
# 自动修改 nginx.conf 里的 80 端口配置，放行 acme-challenge
# 这一步使用 sed 精确插入，以免破坏你现有的跳转规则
sudo sed -i "/server_name.*$DOMAIN.*;/ {
    /listen 80;/,/}/ {
        /location \^~ \/\.well-known\/acme-challenge\//!b
        a \        location ^~ /.well-known/acme-challenge/ { root $WEBROOT; default_type "text/plain"; }
    }
}" /etc/nginx/nginx.conf 2>/dev/null || true

# 稳妥起见，直接创建验证目录
sudo mkdir -p "$WEBROOT/.well-known/acme-challenge"
echo "certbot-test" | sudo tee "$WEBROOT/.well-known/acme-challenge/test.txt" > /dev/null

# 检查 Nginx 语法并重载
if sudo nginx -t; then
    sudo systemctl reload nginx
else
    echo -e "${RED}Nginx 配置测试失败，请检查！${NC}"
    exit 1
fi

echo -e "\n${GREEN}>>> [5/5] 申请证书...${NC}"
echo "注意：如果这里报 403 错误，说明阿里云备案拦截依然生效，无法通过 HTTP-01 验证。"
echo "正在执行申请..."
sudo certbot --nginx -d "$DOMAIN" --email "$EMAIL" --agree-tos --no-eff-email --redirect || {
    echo -e "\n${RED}申请失败。如果你确定安全组已放行 80 端口，但依然报 403，说明被阿里云备案网关拦截（Server: Beaver）。${NC}"
    echo -e "${YELLOW}解决方案：1. 等待备案通过；2. 使用 DNS-01 验证（需要 API）；3. 换用 443 端口的 TLS-ALPN-01 验证。${NC}"
    exit 1
}

echo -e "\n${GREEN}>>> 配置自动续签重载钩子...${NC}"
sudo install -d -m 755 /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'HOOK'
#!/usr/bin/env bash
set -euo pipefail
/usr/sbin/nginx -t
/usr/bin/systemctl reload nginx
HOOK
sudo chmod 750 /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh

echo -e "\n${GREEN}>>> 测试模拟续签...${NC}"
sudo certbot renew --dry-run --run-deploy-hooks

echo -e "\n${GREEN}========================================${NC}"
echo -e "${GREEN} 自动化部署流程执行完毕！${NC}"
echo -e " 请执行 'curl -I https://$DOMAIN' 检查是否生效。"
echo -e "${GREEN}========================================${NC}"
EOF

# 执行脚本
sudo bash /tmp/auto_certbot.sh