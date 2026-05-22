# ЗАДАЧА

Есть несколько (N) nginx в режиме обратного прокси (proxy_pass...). За ними приложение, отвечающее по HTTP.
#### Важно! Запрос от пользователя может прийти на любой из nginx.
Запрос от пользователя может пройти как через один nginх, так и по цепочке через несколько nginx. 

Примеры:
пользователь > nginx1 > приложение;
пользователь > nginx2 > приложение; 
... 
пользователь > nginxN > приложение;
пользователь > nginx1 > nginx2 > nginxN > приложение;
пользователь > nginx2 > nginxN > приложение.

Приложение проверяет заголовок X-Forwarded-For. 
* Требуется, чтобы приложение в заголовке получило IP адрес пользователя и всех nginx (всю цепочку), через которые прошел запрос. 
* Требуется, чтобы приложение не получило заголовок X-Forwarded-For, который недобросовестный пользователь может добавить в свой запрос. 
* Решение подготовить в виде тестового стенда для запуска в docker-compose минимум из трех серверов nginx и любого готового или своего приложения выводящего заголовок X-Forwarded-For. 
* Предоставить протокол тестирования на выполнение всех требований задания с помощью утилиты curl.

# РЕШЕНИЕ

## 1. Файлы тестового стенда
Создаём пустую директорию и размещаем в ней следующие три файла.
#### docker-compose.yml
```bash
networks: 
  proxy_net: 
    ipam: 
      config:
        - subnet: 172.20.0.0/16

services: 
  # Наше приложение (эхо-сервер, выводящий заголовки) 
  app: 
    image: mendhak/http-https-echo:3.5 
    container_name: app_server 
  networks: 
    proxy_net: 
      ipv4_address: 172.20.0.10 
 
  # Первый прокси-сервер 
  nginx1: 
    image: nginx:1.25-alpine 
    container_name: nginx1 
    volumes: 
      - ./nginx.conf:/etc/nginx/nginx.conf:ro 
    ports: 
      - "8081:80" 
    networks: 
      proxy_net: 
        ipv4_address: 172.20.0.1 
 
  # Второй прокси-сервер 
  nginx2: 
    image: nginx:1.25-alpine 
    container_name: nginx2 
    volumes: 
      - ./nginx.conf:/etc/nginx/nginx.conf:ro 
    ports: 
      - "8082:80" 
    networks: 
      proxy_net: 
        ipv4_address: 172.20.0.2 
 
  # Третий прокси-сервер 
  nginx3: 
    image: nginx:1.25-alpine 
    container_name: nginx3 
    volumes: 
      - ./nginx.conf:/etc/nginx/nginx.conf:ro 
    ports: 
      - "8083:80" 
    networks: 
      proxy_net: 
        ipv4_address: 172.20.0.3
```

#### nginx.conf
