FROM nginx:1.19.5-alpine

RUN apk --no-cache add shadow && \
    usermod -u 1001 nginx && \
    groupmod -g 1001 nginx

RUN mkdir /app
RUN chown -R nginx:nginx /app && \
        chown -R nginx:nginx /var/cache/nginx && \
        chown -R nginx:nginx /var/log/nginx && \
        chown -R nginx:nginx /etc/nginx/conf.d && \
        touch /var/run/nginx.pid && \
        chown -R nginx:nginx /var/run/nginx.pid


COPY default.conf /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/nginx.conf




USER nginx
EXPOSE 8000
