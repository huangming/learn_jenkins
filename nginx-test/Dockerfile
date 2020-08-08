FROM alpine:3.7
WORKDIR /
COPY index.html  .
RUN echo " " >> index.html
RUN echo \| build on `date` >> index.html

FROM dogrich/my-nginx:v1
WORKDIR /usr/share/nginx/html
COPY --from=0 /index.html .
# 开放80端口
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

