FROM node:lts AS build1
WORKDIR /webapp
COPY . .
WORKDIR /webapp/frontend
RUN npm install
RUN npm run build

FROM python:3.10 AS final
WORKDIR /webapp
COPY --from=build1 /webapp /webapp
WORKDIR /webapp/backend
RUN python -m pip install -r ./requirements.txt

EXPOSE 80
CMD [ "python", "app.py" ]

# Docker ビルド
# sudo docker build -t app .
# コンテナ内ポート 80 をローカルポート 8080 へ接続
# sudo docker run -p:8080:80 app
# イメージデバッグ (コンテナ内を参照)
# sudo docker run --rm -i -t -p:8080:80 app /bin/bash

