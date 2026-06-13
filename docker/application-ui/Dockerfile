# syntax=docker/dockerfile:1

FROM node:20-bookworm AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY public public
COPY src src
COPY tsconfig.json ./
ARG REACT_APP_API_BASE_URL=http://localhost:8088
ARG REACT_APP_INITIATOR_LOGIN=operator
ENV REACT_APP_API_BASE_URL=$REACT_APP_API_BASE_URL
ENV REACT_APP_INITIATOR_LOGIN=$REACT_APP_INITIATOR_LOGIN
RUN npm run build

FROM nginx:1.27-bookworm
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
