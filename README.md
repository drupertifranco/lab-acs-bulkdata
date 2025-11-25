# Laboratorio ACS con CPE simulados, Filebrowser y Vector

Este repositorio levanta un laboratorio de GenieACS con:

- GenieACS (`drumsergio/genieacs:1.2.13`) + MongoDB
- Simulador de CPE (`drumsergio/genieacs-sim`)
- Filebrowser para explorar artefactos/JSON de pruebas
- Vector como colector de Bulk Data (HTTP) hacia `./data/bulkdata/`


## Componentes y puertos

- GenieACS: `10.64.1.16`
	- `3010/tcp` UI
	- `7547/tcp` CWMP (ACS)
	- `7557/tcp` NBI
- MongoDB: `10.64.1.17` (`27017/tcp`)
- Filebrowser: `10.64.1.19` (`8081:80`)
- Vector (Bulk Collector): `10.64.1.20` (`8080/tcp`)
- CPE(s) simulados: desde `10.64.1.21` en adelante


## Requisitos

- Docker y Docker Compose v2
- Puertos locales libres: `3010`, `7547`, `7557`, `8081`, `8080`, `27017`


## Red y direccionamiento

Todos los servicios comparten la red bridge `genieacs_net` (subred `10.64.1.0/24`).

- `docker-compose.yml` define la red con IPAM y asigna IPs fijas a los servicios base.
- `cpe-services.yaml` usa la red como `external: true` para poder superponer servicios CPE a gran escala. Por eso, antes de usarlo, crea la red manualmente si no existe:

```bash
docker network create --subnet 10.64.1.0/24 genieacs_net || true
```


## Despliegue base

Levanta GenieACS, MongoDB, Filebrowser, Vector y un CPE (`cpe1`):

```bash
docker compose up -d
```

URLs útiles:
- GenieACS UI: http://localhost:3010/
- Filebrowser: http://localhost:8081/ (raíz mapeada a `./data`)
- Vector Healthcheck: http://localhost:8080/health

Notas:
- El simulador `cpe1` ejecuta: `timeout 5 ./genieacs-sim -u http://10.64.1.16:7547/ -s 111111` y termina a los 5s; por `restart: unless-stopped` se reinicia en bucle.
- Si quieres que permanezca “vivo” tras simular, cambia el comando a: `... -s 111111; tail -f /dev/null`.


## Escalar CPEs (50 y 150)

Ya hay un archivo de overlay para 50 CPE: `cpe-services.yaml` (seriales `111111`→`111160`; IPs `10.64.1.21`→`10.64.1.70`).

Levantar base + 50 CPE:
```bash
docker compose -f docker-compose.yml -f cpe-services.yaml up -d
```

Bajar todo lo levantado con ambos archivos:
```bash
docker compose -f docker-compose.yml -f cpe-services.yaml down
```

Ver solo contenedores CPE:
```bash
docker ps --filter "name=cpe-"
```

Escalar a 150 CPE (opción rápida generando un overlay nuevo):

```bash
# Genera cpe-services-150.yaml (CPE 1..150, serial 111111..111260, IP 10.64.1.21..10.64.1.170)
cat > cpe-gen-150.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
base_ip=21         # empieza en 10.64.1.21
count=150
serial_base=111111
outfile=cpe-services-150.yaml

{
	echo "version: '3.8'"
	echo "services:"
	for i in $(seq 1 "$count"); do
		ip_octet=$((base_ip + i - 1))
		serial=$((serial_base + i - 1))
		cat <<YAML
	cpe${i}:
		image: drumsergio/genieacs-sim:latest
		container_name: cpe-${i}
		depends_on: [genieacs]
		command: ["sh","-c","timeout 5 ./genieacs-sim -u http://10.64.1.16:7547/ -s ${serial}"]
		volumes:
			- ./configs/ont/data_model_202BC1-BM632w-8KA8WA1151100043.csv:/opt/genieacs-sim/data_model_202BC1-BM632w-8KA8WA1151100043.csv:ro
		restart: unless-stopped
		networks:
			genieacs_net:
				ipv4_address: 10.64.1.${ip_octet}
YAML
	done
	cat <<YAML
networks:
	genieacs_net:
		external: true
YAML
} > "$outfile"

echo "Generado $outfile"
EOF

chmod +x cpe-gen-150.sh
./cpe-gen-150.sh

# Levantar base + 150 CPE
docker compose -f docker-compose.yml -f cpe-services-150.yaml up -d
```

Importante:
- Cambia el rango de IP si tu red ya ocupa `10.64.1.0/24`.
- El `timeout 5` causará reinicios continuos; añade `; tail -f /dev/null` si quieres contenedores en ejecución estable tras simular.


## Bulk Data con Vector

- Configuración en `vector-1.yaml` (montado en `/etc/vector/vector.yaml`).
- Vector expone `/health` en `8080` y escribe JSON de bulk en `./data/bulkdata/` (montado en el contenedor).

Comprobar Vector:
```bash
curl -s http://localhost:8080/health | jq . || curl -s http://localhost:8080/health
```


## Filebrowser

- UI: http://localhost:8081/ 
- Directorio raíz expuesto: `./data` (en el contenedor `/srv`).
- Primer inicio: crea usuario/contraseña desde la UI si es necesario (ver `filebrowser/config/settings.json`).


## Comandos útiles

Estado:
```bash
docker compose ps
docker ps --filter "name=cpe-"
```

Logs:
```bash
docker logs -f genieacs
docker logs -f bulk-collector
docker logs -f cpe-1
```

Detener / bajar:
```bash
# Solo detener (deja contenedores creados)
docker compose stop

# Bajar stack base
docker compose down

# Bajar base + overlay (p.ej. 50 CPE)
docker compose -f docker-compose.yml -f cpe-services.yaml down

# Eliminar huérfanos si quitaste servicios del YAML
docker compose -f docker-compose.yml -f cpe-services.yaml down --remove-orphans

# Limpiar volúmenes anónimos (más agresivo)
docker compose down -v
```

Red externa (si la creaste manualmente):
```bash
docker network rm genieacs_net
```


## Estructura del repositorio

- `docker-compose.yml`: stack base (GenieACS, MongoDB, Filebrowser, Vector, 1 CPE)
- `cpe-services.yaml`: overlay con 50 CPE predefinidos
- `configs/ont/*.csv`: modelo de datos para el simulador de CPE
- `data/`: datos compartidos (incluye `bulkdata/`)
- `vector-1.yaml`: configuración de Vector
- `web/`: activos web auxiliares (si aplica)
- `filebrowser/config/`: configuración de Filebrowser


## Notas y buenas prácticas

- Si cambias el ancho de pruebas o RTT de red, ajusta `timeout` y/o la cadencia de simulación según tus objetivos.
- Revisa credenciales de GenieACS según la imagen utilizada y variables de entorno definidas en `docker-compose.yml`.
- Mantén separados los overlays (50, 150, etc.) para facilitar levantar/bajar lotes.
