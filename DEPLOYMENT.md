# Deployment Information

## Public URL
https://group15-402-day06.onrender.com/

## Platform
Render

## Service Description
VinLex AI — Trợ lý tư vấn quy chế học vụ VinUniversity, deployed as a web service on Render.

## Test Commands

### Home page (web UI)
```bash
curl https://group15-402-day06.onrender.com/
# Expected: HTML page with VinLex AI chat interface
```

### Login page
```bash
curl -o /dev/null -w "%{http_code}" https://group15-402-day06.onrender.com/login
# Expected: 200
```

## Environment Variables Set
- PORT (injected by Render)
- ENVIRONMENT=production

## Screenshots
- See Railway-Server_Image.png in project folder for deployment dashboard reference
