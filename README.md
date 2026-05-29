# sedcore-config-repo

[sedcore-config-server](https://github.com/skot-oganization/sedcore-config-server) tarafından servis edilen merkezi konfigürasyon.

## Yapı

```
<app>.yml             → base/env-agnostic config (port, app name, env-var pattern'lar)
<app>-dev.yml         → DEV override (localhost DB, ddl-auto=create/update, SQL DEBUG, seed)
<app>-prod.yml        → PROD override (env-var DB, ddl-auto=validate, SQL WARN, {cipher} secrets)
application.yml       → tüm servisler için ortak base
application-dev.yml   → ortak dev override (log seviyesi DEBUG, actuator details=always)
application-prod.yml  → ortak prod override (log WARN, actuator details=when-authorized)
```

Spring Cloud Config bunları **birleştirir** (soldan sağa override):

```
application.yml ← <app>.yml ← application-<profile>.yml ← <app>-<profile>.yml
```

**Profile aktivasyonu**:
- Production compose: `SPRING_PROFILES_ACTIVE=prod` env-var (her client servisinde)
- Developer IDE: `--spring.profiles.active=dev` veya `SPRING_PROFILES_ACTIVE=dev`
- Profile yokken: sadece base yml'ler döner — `ddl-auto` yok (Spring default `none`, conservative)

## Servisler

| App name | Repo | Port |
|---|---|---|
| `security` | sedcore-security | 8002 |
| `pos-product-manager` | sedcore-pos-product-manager | 8001 |
| `api-manager` | sedcore-api-manager | 8080 |
| `notification` | sedcore-notification-service | 8004 |

`spring.application.name` ile eşleşmeli — client servisindeki bu değer config-server'a hangi yml'leri çekeceğini söyler.

## Secret Encryption Workflow

Hassas alanlar (`spring.datasource.password`, `jwt.secret`, API token'lar) **plaintext olarak commit edilmez**. `{cipher}...` formatında encrypt edilir.

### Encrypt etmek için

```bash
# config-server up ve ulaşılabilir olmalı
curl -u $CONFIG_USER:$CONFIG_PASS \
  -X POST http://config-server:8888/encrypt \
  -H "Content-Type: text/plain" \
  -d 'mySecretValue'

# Çıktı: AQAxxxxxx...
```

YAML'a yapıştır:

```yaml
spring:
  datasource:
    password: '{cipher}AQAxxxxxx...'   # tırnak ZORUNLU (YAML parser için)
```

Client servisi config'i çekerken config-server **otomatik decrypt** eder.

### Hangi alanlar encrypt edilmeli (prod yml'lerde)

| Alan | Servis |
|---|---|
| `spring.datasource.password` | security, pos-product-manager, notification |
| `jwt.secret` | security, api-manager, notification (aynı değer) |
| `spring.mail.password` | notification (Sprint 2026-05-27 itibarıyla burada; eskiden pos-product-manager) |
| `twilio.auth-token` | notification (Sprint 2026-05-27 itibarıyla burada) |
| `slack.webhook.url` | notification (Sprint 2026-05-27 itibarıyla burada) |

Dev yml'lerde plaintext OK — fake credentials (`ekalem` / dev-only JWT secret).

## Dev Workflow (Developer Makinesi)

Dev'de config-server **opsiyonel** — developer'lar isteğe bağlı olarak lokal config-server'ı ayağa kaldırabilir veya skip edebilir.

### Opsiyon A — Config-server ile (önerilen, prod paten ile aynı)

```bash
# 1. Local Postgres çalışıyor olsun (port 5432, db ekalem, user ekalem)
# 2. Config-repo'yu config-server'a göstermek için file:// URI veya GitHub HTTPS
docker compose up -d config-server   # veya java -jar ile manuel
# 3. Servisi IDE'den çalıştır
mvn spring-boot:run -Dspring-boot.run.profiles=dev
# veya IDE run config'inde: SPRING_PROFILES_ACTIVE=dev
```

Servis boot'unda config-server'a sorar: "security/dev için config ver" → JSON merge döner → ddl-auto=create, localhost DB, SQL DEBUG.

### Opsiyon B — Config-server'sız (lokal hızlı dev)

Client servisinin `application.properties`'inde `spring.config.import=optional:configserver:` zaten optional. Config-server kapalıysa modül fallback ile boot eder; sadece kendi minimal `application.properties` + env-var'larla çalışır (config-repo'daki dev defaults uygulanmaz, modülün kendi default'ları kullanılır).

### Verify

```bash
# Dev profile değerlerini gör
curl -u $CONFIG_USER:$CONFIG_PASS http://localhost:8888/security/dev | jq .propertySources

# Effective ddl-auto = create (security-dev.yml override)
```

## Yeni servis eklemek

1. 3 yml ekle: `<service>.yml` + `<service>-dev.yml` + `<service>-prod.yml`
2. README'deki tabloya satır ekle
3. Client servisinde:
   - `spring.application.name: <service>` (aynı isim)
   - `pom.xml`'e `spring-cloud-starter-config` + `spring-cloud-dependencies` BOM
   - `spring.config.import: optional:configserver:http://config-server:8888`

## Audit Trail

Tüm değişiklikler git history'de. `git log --oneline <file>` ile bir config dosyasının kim/ne zaman değiştirdiği görünür.

## Refresh (config değişti, restart etmeden uygula)

Spring Cloud Bus kullanmıyoruz şu an. Manuel yöntem:

```bash
# Her client servisinde:
curl -X POST http://<service>:<port>/actuator/refresh
# @RefreshScope'lu bean'ler yeniden oluşturulur
```

Çoğu config (datasource, JWT secret) `@RefreshScope`'lu değil — değişimi uygulamak için **servis restart** gerekir.
