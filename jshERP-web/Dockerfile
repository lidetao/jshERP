# =============================================
# 多阶段构建：Node 构建 → Nginx 运行
# =============================================

# ---- 构建阶段 ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY jshERP-web/package.json jshERP-web/yarn.lock* ./
RUN yarn config set registry https://registry.npmmirror.com && \
    yarn install
COPY jshERP-web/ .
RUN yarn build

# ---- 运行阶段 ----
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY deploy/nginx/jsh_erp.conf /etc/nginx/conf.d/jsh_erp.conf
EXPOSE 80
