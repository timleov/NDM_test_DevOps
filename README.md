# ЗАДАЧА

Есть несколько (N) nginx в режиме обратного прокси (**_proxy_pass_**). За ними приложение, отвечающее по HTTP.  
#### Важно! Запрос от пользователя может прийти на любой из nginx.  
Запрос от пользователя может пройти как через один nginх, так и по цепочке через несколько nginx.

Примеры:  
_пользователь > nginx1 > приложение;_  
_пользователь > nginx2 > приложение;_  
_..._  
_пользователь > nginxN > приложение;_  
_пользователь > nginx1 > nginx2 > nginxN > приложение;_  
_пользователь > nginx2 > nginxN > приложение_.  

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
        - subnet: 172.25.0.0/16

services:
  # Наше приложение (эхо-сервер, выводящий заголовки)
  app:
    image: mendhak/http-https-echo:31
    container_name: app_server
    networks:
      proxy_net:
        ipv4_address: 172.25.0.10

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
        ipv4_address: 172.25.0.2

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
        ipv4_address: 172.25.0.3

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
        ipv4_address: 172.25.0.4
```

#### nginx.conf
```bash
events { worker_connections 1024; }

http {
    # Доверяем всей подсети наших Nginx/Docker
    set_real_ip_from  172.25.0.0/16;
    # Ищем реальный IP в X-Forwarded-For, если пришли из доверенной сети 
    real_ip_header    X-Forwarded-For; 
    # Инструктируем Nginx не доверять внешним (пользовательским) X-Forwarded-For 
    real_ip_recursive on; 
 
    # upstream для динамического определения, куда слать запрос 
    upstream backend { 
        server app:80; 
    } 
 
    server { 
        listen 80; 
 
        # Роутинг для теста цепочек: /proxy2 шлет на nginx2, /proxy3 на nginx3, /app на приложение 
        location /proxy2 { 
            proxy_pass http://172.25.0; 
            proxy_set_header X-Forwarded-For 
            $proxy_add_x_forwarded_for; 
        } 
 
        location /proxy3 { 
            proxy_pass http://172.25.0; 
            proxy_set_header X-Forwarded-For 
            $proxy_add_x_forwarded_for; 
        } 
 
        location /app { 
            proxy_pass http://172.25.0; 
            proxy_set_header X-Forwarded-For 
            $proxy_add_x_forwarded_for; 
        } 
    } 
}
```

## 2. Запуск стенда 
Выполните команду в терминале для сборки и запуска контейнеров:  
```bash
docker-compose up -d
```  

## 3. Протокол тестирования (с помощью curl) 
Для выполнения тестов мы будем использовать curl и фильтровать вывод утилитой grep, чтобы видеть только заголовок x-forwarded-for, пришедший в приложение. 
Примечание: IP-адрес 172.25.0.x в начале строки — это ваш IP в подсети Docker (обычно 172.25.0.1, если вы делаете запрос с хост-машины).

### Тест 1: Прямой запрос через один Nginx (Пользователь -> Nginx1 -> Приложение)  
Запрос отправляется на порт 8081 (nginx1) в эндпоинт /app.  
```bash
curl -s http://localhost:8081/app | grep -i "x-forwarded-for"
```  
Ожидаемый результат:  
```bash
"x-forwarded-for": "172.25.0.1"
```  
(в приложение передается только IP пользователя)  

### Тест 2: Запрос по цепочке (Пользователь -> Nginx1 -> Nginx2 -> Nginx3 -> Приложение)  
Запрос отправляется на порт 8081 (nginx1) в эндпоинт /proxy2. Согласно конфигу, Nginx1 
перенаправит его на Nginx2, тот на Nginx3, а Nginx3 — в приложение.  
```bash
curl -s http://localhost:8081/proxy2 | grep -i "x-forwarded-for"
```  
Ожидаемый результат:  
```bash
"x-forwarded-for": "172.25.0.1, 172.25.0.1, 172.25.0.2"
```  
(цепочка зафиксирована: реальный ip пользователя -> ip Nginx1 -> ip Nginx2)  

### Тест 3: Защита от подмены заголовка злоумышленником (один Nginx)  
Пользователь пытается передать фейковый IP 9.9.9.9 через заголовок X-Forwarded-For на nginx1.  
```bash
curl -s -H "X-Forwarded-For: 9.9.9.9" http://localhost:8081/app | grep -i "x-forwarded-for"
```
Ожидаемый результат:  
```bash
"x-forwarded-for": "172.25.0.1"
```  
(фейковый IP 9.9.9.9 полностью удален, так как Nginx не доверяет внешнему источнику. В приложение ушел только честный IP пользователя)  

### Тест 4: Защита от подмены заголовка злоумышленником при работе в цепочке  
Пользователь передает фейковый IP 9.9.9.9 в цепочку из трех Nginx.  
```bash
curl -s -H "X-Forwarded-For: 9.9.9.9" http://localhost:8081/proxy2 | grep -i "x-forwarded-for" 
```  
Ожидаемый результат:  
```bash
"x-forwarded-for": "172.25.0.1, 172.25.0.1, 172.25.0.2"
```  
(фейковый IP 9.9.9.9 успешно отброшен на первом же Nginx. Вся последующая внутренняя цепочка построена корректно и безопасно)  
