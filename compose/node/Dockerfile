FROM node:12-alpine
WORKDIR /code
COPY ./package.json ./server.js /code/ 
COPY ./static /code/static
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
	&& apk update && apk add yarn \
	&& yarn config set registry https://registry.npm.taobao.org/ \
	&& yarn 
CMD ["npm", "start"]
	
