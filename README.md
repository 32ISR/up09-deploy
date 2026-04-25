# Деплой веб-приложения

## Задача 

_ваш\_домен_ - это который вам выдал Codespace

- Развернуть на 80 порту (то есть `{{ваш_домен}}/`) статику с вашей лабораторной работы `up09-render-lab`
- На `{{ваш_домен}}/api` развернуть express проект из `up09-express`
- На `{{ваш_домен}}/api-v2` развернуть express проект из `up09-express-own`

_Не забудьте удалить `node_modules` после того, как склонировали_

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

  frontend:
    container_name: frontend
    image: nginx
    ports:
      - "80:80"
    environment:
      - NGINX_PORT=80
    volumes:
      - ./frontend:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/nginx.conf
    restart: unless-stopped
    networks:
      - my-app-network

  backend:
    container_name: backend
    build: ./backend/.
    restart: unless-stopped
    networks:
      - my-app-network

networks:
  my-app-network:
```

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
events {}

http {
	include /etc/nginx/mime.types;

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
}
```

`try_files $uri $uri/ /index.html` — для SPA: если файл не найден, отдаёт `index.html`, дальше маршрутизирует фронтенд.

`proxy_pass http://app:3000` — `app` это имя сервиса из Compose, Docker резолвит его во внутренний адрес контейнера.
