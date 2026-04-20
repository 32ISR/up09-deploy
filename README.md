# Деплой веб-приложения

## Задача 

Напишу попозже...

## Docker

Docker упаковывает приложение вместе с окружением в контейнер. Контейнер ведёт себя одинаково на любой машине с Docker — локально, на сервере, в облаке.

Два основных понятия: **образ** — статичный шаблон, **контейнер** — запущенный экземпляр образа. Образы состоят из слоёв, Docker кэширует каждый — неизменившиеся слои при пересборке не трогаются.

---

## Dockerfile

Инструкции для сборки образа.

```dockerfile
FROM node:20-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 3000
CMD ["node", "index.js"]
```

`package.json` копируется до исходного кода намеренно: если код изменился, а зависимости нет — слой с `node_modules` возьмётся из кэша. Поменялся `package.json` — пересборка с этого слоя.

---

## Docker Compose

Описывает всю инфраструктуру в одном файле. Удобно когда сервисов несколько.

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: mydb
      MYSQL_USER: admin
      MYSQL_PASSWORD: secret
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

`depends_on` — порядок запуска, не готовность. База может стартовать, но ещё не принимать подключения. Для этого — `healthcheck`.

`volumes` — без тома данные исчезнут при удалении контейнера.

`ports` — формат `HOST:CONTAINER`.

```bash
docker compose up -d # Поднять сервисы
docker compose down # Опустить сервисы
docker compose logs -f app # Посмотреть логи контейнера app
docker compose build # Собрать все образы из композа
```

---

## Nginx

Веб-сервер и обратный прокси.

Статику отдаёт сам — HTML, CSS, JS, картинки — без участия Node.js. API-запросы проксирует на Express.

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://app:3000;
    }
}
```

`try_files $uri $uri/ /index.html` — для SPA: если файл не найден, отдаёт `index.html`, дальше маршрутизирует фронтенд.

`proxy_pass http://app:3000` — `app` это имя сервиса из Compose, Docker резолвит его во внутренний адрес контейнера.
