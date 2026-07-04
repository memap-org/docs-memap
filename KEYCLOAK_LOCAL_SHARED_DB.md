# Keycloak local dung chung DB

Tai lieu nay dung cho truong hop ban muon chay Keycloak local de dang nhap nhanh hon, nhung van dung chung DB voi node Keycloak o server.

## Gioi han an toan

Chi nen dung theo mo hinh thay phien:

- Dung node Keycloak tren server xa.
- Bat node Keycloak local.
- Frontend tro sang URL Keycloak local.

Khong nen de node remote va node local cung chay dong thoi tren cung DB vi server hien tai dang dung `KC_CACHE=local`, khong co cache invalidation giua nhieu node.

## Tep da them

- `keycloak-local/.env.example`
- `keycloak-local/docker-compose.shared-db.yml`

## Cach dung

1. Tao file env:

```bash
cp keycloak-local/.env.example keycloak-local/.env
```

2. Sua `keycloak-local/.env`:

- `KC_DB_HOST`: mac dinh da dat `84.247.150.108` theo cac file `.env` san co trong repo
- `KC_DB_PORT`: cong MySQL
- `KC_DB_DATABASE`: mac dinh `keycloak`
- `KC_DB_USERNAME`, `KC_DB_PASSWORD`
- `KC_HOSTNAME`: `localhost` neu mo trinh duyet bang `http://localhost:8080`

IP `84.247.150.108` duoc thay the tu cac file dang dung ha tang chung trong repo, vi du:

- `profile-service/.env` -> `jdbc:mysql://84.247.150.108:3306/profile_service`
- `storage-service/.env` -> `jdbc:mysql://84.247.150.108:3306/file-service-db`
- `memap-frontend/.env` -> turn server `84.247.150.108`

Neu DB chi mo ben trong server, hay tao SSH tunnel roi dat:

```bash
KC_DB_HOST=host.docker.internal
KC_DB_PORT=3307
```

va mo tunnel o may local:

```bash
ssh -L 3307:127.0.0.1:3306 user@your-server
```

File compose da kem san:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

de ban dung SSH tunnel tren Linux de hon.

3. Bat Keycloak local:

```bash
docker compose --env-file keycloak-local/.env -f keycloak-local/docker-compose.shared-db.yml up -d
```

4. Xem log:

```bash
docker compose --env-file keycloak-local/.env -f keycloak-local/docker-compose.shared-db.yml logs -f
```

5. Truy cap admin:

```text
http://localhost:8080/admin
```

## Bien moi truong frontend

Cho `memap-frontend`:

```env
VITE_KEYCLOAK_LOCAL_URL=http://localhost:8080
VITE_KEYCLOAK_PROD_URL=https://keycloak.memap.id.vn
VITE_KEYCLOAK_URL=http://localhost:8080
VITE_KEYCLOAK_REALM=<realm-cua-ban>
VITE_KEYCLOAK_CLIENT_ID=<client-id-cua-ban>
```

Cho `memap-admin-frontend`:

```env
NEXT_PUBLIC_KEYCLOAK_LOCAL_URL=http://localhost:8080
NEXT_PUBLIC_KEYCLOAK_PROD_URL=https://keycloak.memap.id.vn
NEXT_PUBLIC_KEYCLOAK_URL=http://localhost:8080
NEXT_PUBLIC_KEYCLOAK_REALM=<realm-cua-ban>
NEXT_PUBLIC_KEYCLOAK_CLIENT_ID=<client-id-cua-ban>
```

## Script chay nhanh

Keycloak local:

```bash
bash scripts/dev-with-keycloak-local.sh
```

Keycloak prod:

```bash
bash scripts/dev-with-keycloak-prod.sh
```

## Luu y quan trong

- Dung dung version `26.0.0` de tranh drift schema/hanh vi.
- Neu realm dang dung theme `memap`, local Keycloak cung can co theme do.
- Neu ban dang sua provider SPI tuy bien tren server, local cung phai nap cung bo provider.
- Neu thay doi realm/client/user tren local, khong bat lai node remote cung luc local con dang chay.

## Dung lai

```bash
docker compose --env-file keycloak-local/.env -f keycloak-local/docker-compose.shared-db.yml down
```
