# Деплой на Hetzner (Ubuntu + Nginx)

## Що потрібно

Всього 2 файли:
- `Index.html` — головна сторінка (~92 KB)
- `data.json` — дані (~3.9 MB)

## Варіант 1: Субдомен + Nginx (рекомендовано)

### 1. Створити DNS запис

В панелі Hetzner (або де у вас DNS):
```
rada.yourdomain.com  →  A  →  IP_ВАШОГО_СЕРВЕРА
```

### 2. Створити папку на сервері

```bash
sudo mkdir -p /var/www/rada
sudo chown $USER:$USER /var/www/rada
```

### 3. Закинути файли

З локальної машини (PowerShell / Git Bash):
```bash
scp Index.html data.json user@YOUR_SERVER:/var/www/rada/
```

### 4. Налаштувати Nginx

```bash
sudo nano /etc/nginx/sites-available/rada
```

Вставити:
```nginx
server {
    listen 80;
    server_name rada.yourdomain.com;
    root /var/www/rada;
    index Index.html;

    # Gzip для швидшого завантаження data.json
    gzip on;
    gzip_types application/json text/html application/javascript;
    gzip_min_length 1000;

    # Кешування
    location ~* \.json$ {
        expires 1h;
        add_header Cache-Control "public, immutable";
    }

    location / {
        try_files $uri $uri/ /Index.html;
    }
}
```

### 5. Увімкнути сайт

```bash
sudo ln -s /etc/nginx/sites-available/rada /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 6. HTTPS (Let's Encrypt)

```bash
sudo certbot --nginx -d rada.yourdomain.com
```

Готово! Сайт доступний на `https://rada.yourdomain.com`

---

## Варіант 2: Без субдомену (підпапка)

Якщо вже є сайт на сервері і хочете додати як підпапку:

```bash
# Копіюємо файли
sudo mkdir -p /var/www/html/rada
sudo cp Index.html data.json /var/www/html/rada/
```

Додати в існуючий nginx конфіг:
```nginx
location /rada/ {
    alias /var/www/rada/;
    index Index.html;
}
```

Сайт буде на `https://yourdomain.com/rada/`

---

## Варіант 3: Швидкий тест (без Nginx)

Для швидкої перевірки прямо на сервері:

```bash
cd /var/www/rada
python3 -m http.server 8080
```

Відкрити: `http://YOUR_SERVER_IP:8080`

---

## Оновлення

Просто перезаливаємо файли:
```bash
scp Index.html data.json user@YOUR_SERVER:/var/www/rada/
```

Nginx підхопить нові файли автоматично — без перезапуску.

---

## Порівняння з Google Apps Script

| | Hetzner | Google Apps Script |
|---|---|---|
| Швидкість | Дуже швидка (gzip, CDN-ready) | Повільна (cold start 3-5 сек) |
| Розмір файлів | Без обмежень | Max 50 MB проект |
| URL | Свій домен | script.google.com/... |
| HTTPS | Let's Encrypt (безкоштовно) | Автоматично |
| Оновлення | scp 2 файли | Копіювати в редактор |
| Кешування | Nginx gzip + cache | Обмежене |

**Рекомендація:** Hetzner значно краще для цього проекту — швидше, простіше, свій домен.
