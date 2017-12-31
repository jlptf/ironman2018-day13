FROM nginx
ARG HTML

# Change nginx config
RUN rm /usr/share/nginx/html/index.html
COPY ./$HTML /usr/share/nginx/html/index.html