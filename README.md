https://github.com/Zeka002/db-backups-to-s3-docker-image/releases

[![Releases](https://img.shields.io/badge/Releases-GitHub--Repo-blue?style=for-the-badge&logo=github)](https://github.com/Zeka002/db-backups-to-s3-docker-image/releases)

# Docker image for automated MySQL/PostgreSQL backups to S3 via cron

A purpose-built Docker image that bundles a backup script for MySQL and PostgreSQL. It dumps data and pushes the results to any S3-compatible storage bucket, such as Amazon S3 or Backblaze B2. The image runs a cron daemon inside to automate backups on a schedule you define.

This project focuses on reliability, portability, and simplicity. It puts control in your hands with clear configuration options, thorough logging, and sensible defaults. The image is self-contained, so you can run it anywhere Docker runs, from your laptop to a cloud host or a CI environment.

Throughout this document you will find practical guidelines, step-by-step instructions, and concrete examples to help you integrate automated backups into your data protection workflows. The emphasis is on clarity and predictable behavior. You can customize the backup process to suit small projects or scale it for bigger deployments, all while keeping your data safe in an S3-compatible bucket.

Table of contents
- Overview and goals
- Core concepts and terminology
- What’s inside the image
- Supported databases and targets
- How backups work inside the container
- Security and secrets management
- Scheduling and cron inside the image
- Configuration and environment variables
- Docker run examples
- Compose-based deployment
- Backup retention and housekeeping
- Logging, monitoring, and observability
- Backup restoration basics
- Troubleshooting and common issues
- Troubleshooting tips for common environments
- Advanced usage and customization
- Release notes and upgrading
- Community guidelines and contribution
- FAQ
- License

Overview and goals
This project aims to provide a straightforward, dependable way to back up databases to S3-compatible storage automatically. The main goals are:
- Reliability: backups run on schedule with predictable outcomes and robust error handling.
- Portability: a single Docker image works across platforms, clouds, and environments.
- Simplicity: a sane default configuration that can be adapted with a few environment variables.
- Security: sensible defaults for handling credentials and sensitive data.

Core concepts and terminology
- Backup: a database dump created by mysqldump for MySQL or pg_dump for PostgreSQL, optionally compressed and encrypted before transfer.
- Target: an S3-compatible storage bucket where backups are stored. This could be AWS S3, Backblaze B2, DigitalOcean Spaces, or another provider that uses the S3 API.
- Schedule: a cron expression that specifies how often backups run.
- Image: a Docker image that includes the backup scripts, a cron daemon, and the runtime for the scripts.
- Secrets: sensitive information such as database credentials and access keys. The recommended approach is to supply secrets via environment variables or Docker secrets, not in plain text files.

What’s inside the image
- A robust backup script that detects the database type (MySQL or PostgreSQL) and executes the appropriate dump tooling.
- A cron daemon configured to trigger backups according to your schedule.
- Optional compression for smaller storage footprints.
- Optional encryption for at-rest protection, with a simple interface to enable you to plug in your preferred method.
- A transport layer that uploads artifacts to an S3-compatible bucket using your credentials.
- Basic but solid logging to stdout/stderr to make it easy to monitor from Docker logs or orchestration platforms.

Supported databases and targets
- MySQL and PostgreSQL are supported out of the box. The script handles common connection scenarios, including:
  - Local or remote hosts
  - TCP/IP and socket-based connections
  - Basic authentication with a username and password
- S3-compatible storage targets are supported. You can point the image at:
  - Amazon S3
  - Backblaze B2
  - DigitalOcean Spaces
  - MinIO and other S3-compatible services
- The goal is to offer a single, consistent interface for backups regardless of where you store the data.

How backups work inside the container
- Initialization
  - On startup, the container reads environment variables or secrets for database credentials, target bucket, region, and schedule.
  - The backup script performs a quick validation of connectivity to the database servers and the S3 endpoint.
- Dump phase
  - For MySQL, the script uses mysqldump to create a logical dump of the configured databases.
  - For PostgreSQL, the script uses pg_dump to create a logical dump.
  - If multiple databases are specified, multiple dumps can be created in a single run or per database, depending on configuration.
- Processing phase
  - Dumps can be compressed using gzip or xz to reduce storage space and network transfer costs.
  - Optional encryption can be applied using a pluggable strategy (e.g., symmetric encryption with a passphrase supplied via environment variable or secret).
- Upload phase
  - The processed dumps are uploaded to the configured S3 bucket via the S3-compatible API.
  - The upload is attempted with retry logic to improve reliability in the face of transient network issues.
- Retention policy
  - After successful uploads, a retention policy governs how many backups are kept and for how long.
  - Old backups are pruned according to the configured settings to prevent storage bloat.
- Observability
  - Logs detail each step: validation, dumps, compression, encryption, upload, and retention actions.
  - The logs include timestamps and clear status messages to aid troubleshooting.

Security and secrets management
- Credentials are the backbone of a secure backup workflow. The image is designed to handle secrets safely.
- Best practices:
  - Use Docker secrets or a secrets management tool to provide credentials rather than embedding them in images.
  - Avoid printing credentials in logs. The backup script redacts sensitive values when logging.
  - Rotate credentials periodically and audit access to the bucket.
  - Use least privilege for database credentials, granting only the necessary permissions for dumping.
- Environment variables vs secrets:
  - If you use environment variables, ensure they are stored securely and not committed to source control.
  - If you use Docker secrets, mount them into the container at runtime and reference them in the script.
- Transport security:
  - TLS is used for communications with S3-compatible services by default. Ensure that your endpoint supports TLS for protection in transit.

Scheduling and cron inside the image
- The image includes a small cron daemon to manage scheduled backups.
- Cron expressions
  - A cron expression determines when the backup runs. Typical patterns are:
    - Daily at 2:00 AM: 0 2 * * *
    - Daily at a custom time: 30 3 * * *
    - Weekly on Sundays: 0 4 * * 0
  - You can pass a custom CRON_SCHEDULE value via environment variable or config file, and the script will respect it.
- Time zones
  - Cron uses the container’s time zone. You can set TZ to a specific zone (e.g., TZ=UTC or TZ=America/New_York) to align backups with your preferred time.
- Robustness
  - If a backup fails, the script logs the error and exits with a non-zero code, which the cron daemon can interpret to trigger retries or alerting in your environment.
  - You can tune the retry policy for uploads to handle transient network problems without interrupting the schedule.

Configuration and environment variables
- The default behavior is designed to be sensible, but you can tailor it to your environment. Key configuration areas include:
  - Database type and connection
    - DB_TYPE: mysql or postgres
    - MYSQL_HOST, MYSQL_PORT, MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD
    - POSTGRES_HOST, POSTGRES_PORT, POSTGRES_DATABASE, POSTGRES_USER, POSTGRES_PASSWORD
  - S3-compatible target
    - S3_BUCKET: name of the bucket
    - S3_ENDPOINT: optional custom endpoint (e.g., s3.us-west-1.amazonaws.com, https://s3.your-provider.com)
    - S3_REGION: region where the bucket resides
    - AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY: credentials for the target
  - Scheduling and retention
    - CRON_SCHEDULE: cron expression
    - RETENTION_DAYS: number of days to keep backups
    - MAX_BACKUPS: maximum number of backups to retain (per bucket or per database)
  - Data processing options
    - COMPRESS: true to enable compression (gzip or xz)
    - ENCRYPT: true to enable optional encryption
    - ENCRYPTION_KEY or ENCRYPTION_KEYFILE: keys or key file references for encryption
  - Logging
    - LOG_LEVEL: debug, info, warning, error
  - Networking
    - TIMEOUT_SECONDS: timeout for database connections and uploads
  - Secrets management
    - Use file-based secrets mounted into /run/secrets or /etc/secrets, depending on your orchestration system
- Example environment block
  - DB_TYPE=mysql
  - MYSQL_HOST=db.myhost
  - MYSQL_PORT=3306
  - MYSQL_DATABASE=mydb
  - MYSQL_USER=root
  - MYSQL_PASSWORD=changeme
  - S3_BUCKET=my-backups
  - S3_ENDPOINT=https://s3.your-provider.com
  - S3_REGION=us-west-2
  - AWS_ACCESS_KEY_ID=AKIAEXAMPLE
  - AWS_SECRET_ACCESS_KEY=secretExample
  - CRON_SCHEDULE=0 2 * * *
  - RETENTION_DAYS=30
  - COMPRESS=true
  - ENCRYPT=false
  - TZ=UTC

Docker run examples
- Quick start (single database, simple credentials)
  - This runs the image in a short-lived container to verify connectivity and perform the first backup.
  - Note: For security, avoid printing sensitive data in logs.
  - Command:
    - docker run --rm \
      -e DB_TYPE=mysql \
      -e MYSQL_HOST=your-db-host \
      -e MYSQL_PORT=3306 \
      -e MYSQL_DATABASE=yourdb \
      -e MYSQL_USER=youruser \
      -e MYSQL_PASSWORD=yourpass \
      -e S3_BUCKET=your-backups \
      -e S3_ENDPOINT=https://s3.your-provider.com \
      -e S3_REGION=us-west-2 \
      -e AWS_ACCESS_KEY_ID=AKIA... \
      -e AWS_SECRET_ACCESS_KEY=secret... \
      -e CRON_SCHEDULE="0 2 * * *" \
      db-backups-to-s3-docker-image:latest
- Advanced run with secrets
  - Use Docker secrets to supply credentials in a secure manner.
  - Create secret files:
    - echo "yourpass" > mysql_password.txt
    - echo "secretkey" > aws_secret.txt
  - Run:
    - docker run --rm \
      --secret mysql_password_secret:/run/secrets/mysql_password \
      --secret aws_secret_secret:/run/secrets/aws_secret \
      -e DB_TYPE=mysql \
      -e MYSQL_HOST=your-db-host \
      -e MYSQL_PORT=3306 \
      -e MYSQL_DATABASE=yourdb \
      -e MYSQL_USER=youruser \
      -e S3_BUCKET=your-backups \
      -e S3_ENDPOINT=https://s3.your-provider.com \
      -e S3_REGION=us-west-2 \
      -e CRON_SCHEDULE="0 2 * * *" \
      db-backups-to-s3-docker-image:latest
- One-line compose example
  - A minimal docker-compose.yml to run the backup service as a daemon.
  - File: docker-compose.yml
  - Content:
    - version: "3.8"
    - services:
        - backup:
            image: db-backups-to-s3-docker-image:latest
            restart: always
            environment:
              - DB_TYPE=mysql
              - MYSQL_HOST=db
              - MYSQL_PORT=3306
              - MYSQL_DATABASE=mydb
              - MYSQL_USER=root
              - MYSQL_PASSWORD=changeme
              - S3_BUCKET=my-backups
              - S3_ENDPOINT=https://s3.your-provider.com
              - S3_REGION=us-west-2
              - AWS_ACCESS_KEY_ID=AKIAEXAMPLE
              - AWS_SECRET_ACCESS_KEY=secretExample
              - CRON_SCHEDULE=0 2 * * *
              - RETENTION_DAYS=30
              - TZ=UTC

Compose-based deployment
- Why use Docker Compose?
  - If you run multiple services, Compose makes it easy to express dependencies, environment, and volumes.
  - It helps you reproduce the same setup in local development and production clusters.
- Example docker-compose snippet
  - version: "3.9"
  - services:
      - backups:
          image: zeka002/db-backups-to-s3-docker-image:latest
          environment:
            - DB_TYPE=postgres
            - POSTGRES_HOST=db
            - POSTGRES_PORT=5432
            - POSTGRES_DATABASE=mydb
            - POSTGRES_USER=postgres
            - POSTGRES_PASSWORD=changeme
            - S3_BUCKET=my-backups
            - S3_ENDPOINT=https://s3.your-provider.com
            - S3_REGION=us-east-1
            - CRON_SCHEDULE=0 3 * * *
            - RETENTION_DAYS=14
            - TZ=UTC
          restart: unless-stopped
          secrets:
            - postgres_password
  - secrets:
      - postgres_password:
          file: ./secrets/postgres_password.txt
- Step-by-step:
  - Place the file in your project with the required secrets.
  - Run: docker-compose up -d

Backup retention and housekeeping
- A healthy retention policy keeps the right number of backups while avoiding storage bloat.
- Common strategies:
  - Keep N daily backups for M days
  - Retain weekly backups for N weeks
  - Maintain a separate set of monthly backups
- The image supports:
  - Retention window in days
  - Maximum number of backups per database
  - Granular control per dump target
- How retention is enforced:
  - A cleanup step runs after a successful backup upload.
  - The script lists backups in the bucket, evaluates their timestamps, and deletes those outside the policy.
  - Logs indicate which backups were pruned and why.

Logging, monitoring, and observability
- Logs are written to stdout/stderr for easy capture by Docker, Kubernetes, or your logging stack.
- Typical log messages include:
  - Connectivity checks to the database
  - Dump start and end times
  - Compression and encryption status
  - Upload results with HTTP status and bucket path
  - Retention actions and any pruning details
- You can route logs to a centralized system:
  - Ship logs to a logging service using a sidecar or a DaemonSet
  - Use a log router to forward stdout/stderr to a central aggregator
- Health checks:
  - The container can expose a basic health endpoint or log health beacons that a orchestrator can watch
  - You can implement a separate probe to verify that dumps and uploads succeed on a schedule

Backup restoration basics
- Restoring a dumped database from S3 backups
  - Locate the latest backup in your bucket using the naming convention your workflow uses
  - Download the dump file locally or stream it to the database host
  - For MySQL: use mysql or mysqlimport to restore the .sql file
  - For PostgreSQL: use psql to restore the dump, or use pg_restore if you produce a custom format
- If you compress backups, decompress before restoration
- Consideroint order and consistency:
  - For transactional databases, ensure a clean recovery window. In some environments, you may want to restore a full dump and then apply incremental or log-based recovery
- Testing restore regularly
  - Schedule a periodic restore test into a separate environment
  - Validate data integrity with checksums or row counts to ensure confidence in your backups
- Documented recovery steps
  - Maintain a clear runbook describing each restoration step for both MySQL and PostgreSQL
  - Include prerequisites such as the target database version, extensions, and required privileges

Troubleshooting and common issues
- Connectivity problems
  - Check that the container can reach the database host or socket
  - Verify authentication details and privileges
  - Ensure that the correct host, port, and database names are configured
- Upload problems
  - Validate the S3 endpoint and region
  - Confirm that credentials have permission to write to the bucket
  - Look for network-related errors and increase timeouts if needed
- Cron not triggering
  - Confirm that the cron expression is valid
  - Check that the cron daemon is running inside the container
  - Ensure the timezone is set correctly
- Data format issues
  - If the dump is not in the expected format, check the dump command options
  - Verify the database tool versions in the image
- Performance concerns
  - Large databases may take longer to dump and upload
  - Consider adjusting the backup window to a time with lower activity

Troubleshooting tips for common environments
- Local development
  - Use a lightweight setup with a single database container and a local S3 emulator if available
  - Verify credentials and end-to-end connectivity
- Cloud deployments
  - Use a dedicated IAM user or service account with the least privilege
  - Enable bucket versioning and lifecycle rules for additional safety
- On-premises environments
  - Ensure network routes permit outbound access to the S3 endpoint
  - Consider using a proxy if your network requires it
- CI pipelines
  - Treat backups as a build artifact rather than just a runtime process
  - Use ephemeral containers with a defined, versioned image to guarantee reproducibility

Advanced usage and customization
- Custom backup scripts
  - If you want to replace or extend the backup script, mount a custom script into the container and adjust the cron command to call it
  - Ensure your custom script handles the same lifecycle steps: dump, optional compression, optional encryption, and upload
- Different dump strategies
  - For MySQL, dump all databases or selected ones
  - For PostgreSQL, use a consistent snapshot mode or dump individual databases
- Encryption options
  - Add a passphrase or key management solution to manage encryption keys
  - If you choose to encrypt locally, ensure you securely store and rotate keys
- Observability hooks
  - Emit structured logs (JSON) to ease ingestion into centralized log systems
  - Export metrics such as backup duration, file size, and transfer rate if you want to feed them into a metrics system
- Custom retention policies
  - Build retention rules that fit your data retention requirements and legal obligations
  - Combine daily/weekly/monthly retention patterns for a more nuanced approach

Release notes and upgrading
- The releases page contains the latest built images, including any notable changes
  - You can browse release notes to see new features, bug fixes, and breaking changes
  - The release assets may include a tarball, a Docker image tag, or both
  - For the latest asset download, visit the releases page
- How to upgrade
  - Pull the new image tag: docker pull db-backups-to-s3-docker-image:latest
  - Stop the old container and remove it
  - Run the container with the same environment variables to apply the new behavior
  - If you rely on a specific version tag, update your deployment to use that tag
- Asset download path
  - When upgrading, you may download a tarball from the releases feed if your workflow prefers a portable artifact:
    - https://github.com/Zeka002/db-backups-to-s3-docker-image/releases/latest/download/db-backups-to-s3-docker-image-latest.tar.gz
  - Note that the tarball contains a complete, portable image you can load with docker load -i

Community guidelines and contribution
- This project welcomes contributions
  - Report issues with clear, reproducible steps
  - Propose enhancements with a concise description and potential impact
  - Submit pull requests with tests or documentation updates
- Code style and quality
  - Keep scripts readable and well-documented
  - Use shell best practices and robust error handling
  - Add comments to explain complex logic
- Documentation
  - Update the README with examples, edge cases, and any new configuration options
  - Provide usage scenarios that help other users understand the practical implications
- Licensing
  - This project uses a permissive license suitable for open-source distribution
  - Review the license terms to ensure compliance when integrating into your projects

FAQ
- Do I need internet access inside the container for backups?
  - Yes. The container needs network access to reach the database host and the S3-compatible storage service.
- Can I backup more than one database in a single run?
  - Yes. The backup script supports multiple databases. You can configure it to dump several databases in a single pass or as separate dumps.
- Is it possible to store backups in a region different from the live database region?
  - Yes. The storage location (bucket, region) can be independent of the database location.
- How do I rotate or delete old backups in the bucket?
  - Use the built-in retention policy. It can prune older dumps after successful uploads according to configurable retention rules.
- Can I use a private container registry for the image?
  - Yes. Build locally or in your CI, tag the image, and push to your private registry. Then reference the image in your deployment.

Images and visuals to enrich the README
- Hero image
  - A crisp banner that conveys the idea of reliable, automated backups and cloud storage
  - Example asset: a simple illustration showing a database icon and a cloud with arrows to a storage bucket
- Logos and icons
  - Docker logo to reflect container-based deployment
  - S3-compatible storage icon to emphasize the destination
  - Database icons for MySQL and PostgreSQL
- Diagram
  - A lightweight diagram that shows the flow: Database -> Dump -> Optional Compression/Encryption -> Upload -> S3 bucket
- Embedding images
  - Use stable, reputable sources where possible to avoid broken images
  - Example:
    - Docker logo: ![Docker](https://img.icons8.com/fluency/256/docker.png)
    - S3 icon: ![S3](https://img.icons8.com/fluency/256/aws-s3.png)
  - If you prefer to host your own images, store them in your repository or a CDN you control

Examples and templates you can copy
- Minimal Docker run (production-friendly)
  - docker run -d --name db-backups \
      -e DB_TYPE=mysql \
      -e MYSQL_HOST=db.example.com \
      -e MYSQL_PORT=3306 \
      -e MYSQL_DATABASE=mydb \
      -e MYSQL_USER=myuser \
      -e MYSQL_PASSWORD=secret \
      -e S3_BUCKET=my-backups \
      -e S3_ENDPOINT=https://s3.example.com \
      -e S3_REGION=us-east-1 \
      -e AWS_ACCESS_KEY_ID=AKIA... \
      -e AWS_SECRET_ACCESS_KEY=secret \
      -e CRON_SCHEDULE="0 2 * * *" \
      -e RETENTION_DAYS=30 \
      db-backups-to-s3-docker-image:latest
- Full-featured Compose example
  - version: "3.9"
  - services:
      backups:
        image: db-backups-to-s3-docker-image:latest
        environment:
          - DB_TYPE=mysql
          - MYSQL_HOST=db
          - MYSQL_PORT=3306
          - MYSQL_DATABASE=mydb
          - MYSQL_USER=root
          - MYSQL_PASSWORD=changeme
          - S3_BUCKET=my-backups
          - S3_ENDPOINT=https://s3.your-provider.com
          - S3_REGION=us-west-2
          - CRON_SCHEDULE=0 2 * * *
          - RETENTION_DAYS=30
          - TZ=UTC
        restart: unless-stopped
  - secrets:
      - mysql_password:
          file: ./secrets/mysql_password.txt

Safety and compliance considerations
- Treat backup data as sensitive
  - Encrypt backups if your threat model requires it
  - Use strong access controls on the bucket and its credentials
  - Monitor and alert on unusual backup activity
- Maintain backups with care
  - Validate backups periodically
  - Keep a tested restoration plan and run drills
  - Document your retention policies and how they align with compliance requirements

Releases and artifact download note
- The releases page is the central place for distributing builds and assets
  - You can browse versioned releases, view release notes, and download artifacts
  - The releases page also hosts portable artifacts for scenarios where you want to manually load a Docker image
- The release link you were given leads to the Releases section
  - If you need the direct asset for a portable image, you can download it from the latest release:
    - https://github.com/Zeka002/db-backups-to-s3-docker-image/releases/latest/download/db-backups-to-s3-docker-image-latest.tar.gz
- If you prefer a quick visit rather than downloading, you can always open the Releases page to see what is available and what matches your environment

Notes on usage with containers and orchestration
- Kubernetes
  - You can deploy this container as a CronJob or as a standard Deployment with a sidecar to handle secrets
  - Use a ConfigMap or Secret to inject environment variables securely
  - Ensure the pod has network access to both the database host(s) and the S3 endpoint
- Docker Swarm
  - Use a service with replicated replicas depending on your backup cadence
  - Manage credentials with Docker secrets and bind-ms for volumes if needed
- Best practices for production
  - Use a dedicated bucket for backups to reduce the blast radius of misconfigurations
  - Turn on bucket versioning if your provider supports it
  - Implement a backup verification step to confirm that dumps can be restored

End of content
