## Self-Hosting PostgreSQL on RHEL with Docker and Nginx Reverse Proxy

Learn how to set up PostgreSQL in a Docker container on a RHEL VM, configure Nginx as a reverse proxy, and make your database accessible from another machine securely. This guide covers each step from creating a Docker Compose setup for PostgreSQL to configuring firewalls and security settings.

### Prerequisites

- **Docker** installed on the RHEL VM.  
  You can follow the official Docker documentation for installation on RHEL:  
  👉 [Install Docker Engine on RHEL](https://docs.docker.com/engine/install/rhel/)

### Steps

1. ### Create a Directory Structure

   Organize your PostgreSQL setup by creating a `services` directory and a subdirectory for PostgreSQL.

   ```bash
   mkdir -p services/postgres
   cd services/postgres
   ```

2. ### Create the `docker-compose.yml` File

   This file defines the Docker services for PostgreSQL.

   ```bash
   touch docker-compose.yml
   vi docker-compose.yml
   ```

   Add the following configuration to the `docker-compose.yml` file:

   ```yaml
   version: '3.9'

   services:
     postgres:
       image: postgres:latest
       container_name: postgresql
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         POSTGRES_DB: testdb
       ports:
         - "5432:5432"
       volumes:
         - pgdata:/var/lib/postgresql/data
       mem_limit: 3g      # Limit memory usage to 3 GB
       cpus: 1.5          # Limit to 1.5 CPUs

   volumes:
     pgdata:
   ```

   - This configuration pulls the latest PostgreSQL image and sets environment variables for the user, password, and database name.
   - The `ports` section maps port 5432 on the host to port 5432 in the container, allowing external connections.
   - The `pgdata` volume ensures data persists even if the container is removed.

3. ### Configure the Firewall

   Allow traffic through the firewall for the PostgreSQL and Nginx proxy ports.

   ```bash
   sudo firewall-cmd --add-port=5432/tcp --permanent
   sudo firewall-cmd --add-port=9856/tcp --permanent
   ```

4. ### Enable Network Connections for Nginx (TCP Streams)

   Since we’re using Nginx’s **TCP stream module**, we need to adjust SELinux settings to allow such connections. Run the following command:

   ```bash
   sudo setsebool -P nis_enabled 1
   ```

   > This enables the `nis_enabled` boolean, which allows services like Nginx to establish outbound TCP connections — necessary for proxying PostgreSQL connections.

5. ### Reload the Firewall

   Reload the firewall to apply these changes.

   ```bash
   sudo firewall-cmd --reload
   ```

6. ### Start PostgreSQL with Docker Compose

   Bring up the PostgreSQL container in detached mode using the Docker Compose **plugin** (note the space between `docker` and `compose`):

   ```bash
   sudo docker compose up -d
   ```

   > This command uses the Docker Compose plugin (recommended for modern Docker installations) to start the PostgreSQL container in the background.

7. ### Access PostgreSQL

   Confirm that PostgreSQL is running by accessing it from within the container:

   ```bash
   sudo docker exec -it postgresql psql -U postgres -d testdb
   ```

   This command opens an interactive `psql` session connected to the `testdb` database as the `postgres` user.

8. ### Install Nginx

   Install Nginx if it is not already installed.

   ```bash
   sudo yum install nginx
   ```

9. ### Configure Nginx as a TCP Proxy

   Nginx can act as a reverse proxy, forwarding requests on port 9856 to PostgreSQL’s port 5432.

   - Create a configuration file for the proxy setup using `sudo` to ensure proper permissions:

     ```bash
     sudo touch /usr/share/nginx/modules/postgres.conf
     sudo vi /usr/share/nginx/modules/postgres.conf
     ```

   - Add the following configuration:

     ```nginx
     stream {
         server {
             listen 9856;
             proxy_connect_timeout 60s;
             proxy_pass localhost:5432;
         }
     }
     ```

10. ### Restart Nginx

    Apply the new configuration by restarting Nginx:

    ```bash
    sudo systemctl restart nginx
    ```

11. ### Test the Connection

    Verify that Nginx is successfully forwarding requests by using `nc` (netcat) from another machine or your VM:

    ```bash
    nc -vz <your_private_ip> 9856
    ```

    > Replace `<your_private_ip>` with the actual private IP address of your RHEL VM. This command will check if the connection to the PostgreSQL service is being forwarded correctly through Nginx.

12. ### Open Port 9856 in VM's Security Settings

    Make sure that your VM's network security settings (e.g., security groups, ingress rules) allow access to port 9856.

13. ### Connect from Another Machine

    From a different VM or machine, connect to PostgreSQL using the Nginx proxy:

    ```bash
    psql -h <your_host> -p 9856 -U postgres -d testdb
    ```

    Replace `<your_host>` with the IP address of your VM. This command should establish a remote connection to the PostgreSQL instance hosted in Docker.

### Summary

With these steps, you have successfully set up a PostgreSQL database in a Docker container on RHEL, configured Nginx as a reverse proxy to forward external traffic, and opened necessary ports. This setup ensures that your database is accessible securely and can handle incoming connections from other machines on the network.
