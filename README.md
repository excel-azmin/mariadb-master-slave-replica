# mariadb-master-slave-replica

* Create Config File
  
	`master-db.cnf`

```
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
```

* create stack

  `mariadb`


  ```

version: "3.7"

services:
  backend:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network  
  
  configurator:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: none 
    entrypoint:
      - bash
      - -c
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_SOCKETIO";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: mariadb-master
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
  
  mariadb-master:
    image: mariadb:10.6
    healthcheck:
      test: mysqladmin ping -h localhost --password=VTtVNLTvqi0K9tl
      interval: 1s
      retries: 15
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed
    environment:
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
      - MARIADB_REPLICATION_MODE=master
      - MARIADB_REPLICATION_USER=repl_user
      - MARIADB_REPLICATION_PASSWORD=VTtVNLTvqi0K9tl
      - MARIADB_ROOT_PASSWORD=VTtVNLTvqi0K9tl
    volumes:
      - mariadb_master_data:/var/lib/mysql
    networks:  
      - bench-db-network 

  mariadb-slave1:
    image: mariadb:10.6
    healthcheck:
      test: mysqladmin ping -h localhost --password=VTtVNLTvqi0K9tl
      interval: 1s
      retries: 15
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed 
    environment:
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
      - MARIADB_REPLICATION_MODE=slave
      - MARIADB_REPLICATION_USER=repl_user
      - MARIADB_REPLICATION_PASSWORD=VTtVNLTvqi0K9tl
      - MARIADB_MASTER_HOST=mariadb-master
      - MARIADB_MASTER_PORT_NUMBER=3306
      - MARIADB_ROOT_PASSWORD=VTtVNLTvqi0K9tl
    volumes:
      - mariadb_slave_data1:/var/lib/mysql
    networks:  
      - bench-db-network
      
  mariadb-slave2:
    image: mariadb:10.6
    healthcheck:
      test: mysqladmin ping -h localhost --password=VTtVNLTvqi0K9tl
      interval: 1s
      retries: 15
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed 
    environment:
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
      - MARIADB_REPLICATION_MODE=slave
      - MARIADB_REPLICATION_USER=repl_user
      - MARIADB_REPLICATION_PASSWORD=VTtVNLTvqi0K9tl
      - MARIADB_MASTER_HOST=mariadb-master
      - MARIADB_MASTER_PORT_NUMBER=3306
      - MARIADB_ROOT_PASSWORD=VTtVNLTvqi0K9tl
    volumes:
      - mariadb_slave_data2:/var/lib/mysql
    networks:  
      - bench-db-network 
  
  frontend:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
      labels:
        traefik.docker.network: traefik-public
        traefik.enable: "true"
        traefik.constraint-label: traefik-public
        traefik.http.routers.erpnext--http.rule: Host(${SITES:?No sites set})
        traefik.http.routers.erpnext--http.entrypoints: http
        traefik.http.routers.erpnext--http.middlewares: https-redirect
        traefik.http.routers.erpnext--https.rule: Host(${SITES})
        traefik.http.routers.erpnext--https.entrypoints: https
        traefik.http.routers.erpnext--https.tls: "true"
        traefik.http.routers.erpnext--https.tls.certresolver: le
        traefik.http.services.erpnext-.loadbalancer.server.port: "8080"    

    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: $$host
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
      PROXY_READ_TIMEOUT: 120
      CLIENT_MAX_BODY_SIZE: 50m
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:
      - bench-db-network
      - bench-network
      - traefik-public 

  queue-default:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - bench
      - worker
      - --queue
      - default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network   

  queue-long:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - bench
      - worker
      - --queue
      - long
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network   

  queue-short:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - bench
      - worker
      - --queue
      - short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network   

  redis-queue:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure 
    volumes:
      - redis-queue-data:/data
    networks:  
      - bench-network  

  redis-cache:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure 
    volumes:
      - redis-cache-data:/data
    networks:  
      - bench-network    

  redis-socketio:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure 
    volumes:
      - redis-socketio-data:/data
    networks:  
      - bench-network  

  scheduler:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - bench
      - schedule
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network   

  websocket:
    image: excelazmin/erpnext-v14:${ERP_VERSION?Variable ERP_VERSION not set}
    deploy:
      restart_policy:
        condition: on-failure 
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - bench-db-network
      - bench-network   
      
volumes:
  mariadb_master_data:
  mariadb_slave_data1:
  mariadb_slave_data2:
  redis-queue-data:
  redis-cache-data:
  redis-socketio-data:
  sites:
  logs:

networks:
  bench-network:
    name: erpnext-v14
    external: true
  bench-db-network:
    name: bench-db-network
    external: true
  traefik-public:
    name: traefik-public
    external: true  

#SITES=`erp.example.com`    
  ```
