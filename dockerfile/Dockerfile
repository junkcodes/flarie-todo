FROM node:lts
WORKDIR /app
COPY package.json .
COPY src .
RUN npm install
EXPOSE 3000
CMD ["node","index.js"]
