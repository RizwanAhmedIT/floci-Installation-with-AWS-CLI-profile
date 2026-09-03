# Floci Installation & AWS CLI Profile Setup

## 1. Purpose

This document explains how to install and run **Floci** locally and configure the AWS CLI so that a dedicated AWS CLI profile named `dev` communicates with Floci instead of the real AWS environment.

The goal is to maintain a safe separation:

```text
AWS CLI
│
├── dev
│    └── Floci
│         └── http://localhost:4566
│
└── production
     └── Real AWS
```

> **Important:** The `production` profile is intentionally not configured in this document. The current setup is only for Floci development/testing.

---

# 2. What is Floci?

Floci is an AWS emulator that provides AWS-compatible APIs locally.

Instead of:

```text
AWS CLI
   ↓
Real AWS
```

we can use:

```text
AWS CLI
   ↓
http://localhost:4566
   ↓
Floci
   ↓
Local AWS services
```

Floci supports running through Docker, a native binary, or from source. Docker is the recommended installation method.

---

# 3. Prerequisites

Check Docker:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Check AWS CLI:

```bash
aws --version
```

The Floci Docker setup requires Docker 20.10+ and Docker Compose v2+.

---

# 4. Create a Floci Working Directory

Create a dedicated directory:

```bash
mkdir -p ~/floci-test
cd ~/floci-test
```

Purpose:

- Keeps Floci configuration separate from other projects.
- Makes the local Floci data easy to locate.
- Provides a clean location for `docker-compose.yml`.

---

# 5. Create Docker Compose Configuration

Create:

```bash
nano docker-compose.yml
```

Use:

```yaml
services:
  floci:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - ./data:/app/data
```

### Explanation

```yaml
image: floci/floci:latest
```

Uses the Floci Docker image.

```yaml
ports:
  - "4566:4566"
```

Exposes Floci on:

```text
http://localhost:4566
```

```yaml
volumes:
  - ./data:/app/data
```

Persists Floci data in the local `./data` directory.

Floci's official quick start uses port `4566` and supports a local bind mount for `/app/data`.

---

# 6. Start Floci

Run:

```bash
docker compose up -d
```

### Purpose

Starts Floci in detached/background mode.

Check the container:

```bash
docker ps
```

You should see port mapping similar to:

```text
0.0.0.0:4566->4566/tcp
```

Check logs:

```bash
docker compose logs -f floci
```

Press:

```text
Ctrl+C
```

to leave the log view.

---

# 7. Verify AWS CLI Installation

Check:

```bash
aws --version
```

Example:

```text
aws-cli/2.x.x Python/3.x.x Linux/... exe/x86_64
```

Check existing profiles:

```bash
aws configure list-profiles
```

In our initial setup, the existing profile was:

```text
default
```

We did **not** modify the existing `default` profile.

---

# 8. Create the Floci AWS CLI Profile

Create a dedicated profile called:

```text
dev
```

Run:

```bash
aws configure --profile dev
```

Enter:

```text
AWS Access Key ID [None]: test
AWS Secret Access Key [None]: test
Default region name [None]: us-east-1
Default output format [None]: json
```

### Why use `test` credentials?

Floci accepts non-empty/dummy credentials. A real AWS account or real AWS access keys are not required.

Therefore:

```text
Access Key = test
Secret Key = test
```

are intentionally fake credentials.

**Never use real production AWS credentials for Floci.**

---

# 9. Verify the `dev` Profile

Run:

```bash
aws configure list-profiles
```

Expected:

```text
default
dev
```

Then:

```bash
aws configure list --profile dev
```

Expected configuration:

```text
profile       dev
access_key    ****************test
secret_key    ****************test
region        us-east-1
```

The AWS CLI `configure list` command shows the configuration values and where they were obtained from.

---

# 10. Configure the Floci Endpoint

Set the endpoint specifically for the `dev` profile:

```bash
aws configure set endpoint_url http://localhost:4566 --profile dev
```

This adds the endpoint to:

```text
~/.aws/config
```

The resulting configuration is:

```ini
[default]
region = ap-south-1

[profile dev]
region = us-east-1
output = json
endpoint_url = http://localhost:4566
```

AWS CLI supports `endpoint_url` in a profile's shared configuration file for routing requests to alternate endpoints.

---

# 11. AWS Credentials File

The `dev` credentials are stored in:

```text
~/.aws/credentials
```

Expected:

```ini
[dev]
aws_access_key_id = test
aws_secret_access_key = test
```

The AWS CLI separates configuration settings and credentials between `~/.aws/config` and `~/.aws/credentials`. Named profiles allow different configurations to be used independently.

---

# 12. Final AWS CLI Configuration

Our current setup is:

### `~/.aws/config`

```ini
[default]
region = ap-south-1

[profile dev]
region = us-east-1
output = json
endpoint_url = http://localhost:4566
```

### `~/.aws/credentials`

```ini
[dev]
aws_access_key_id = test
aws_secret_access_key = test
```

---

# 13. Test Floci

Run:

```bash
aws s3 ls --profile dev
```

Because `endpoint_url` is configured inside the `dev` profile, we don't need:

```text
--endpoint-url http://localhost:4566
```

The command automatically uses:

```text
dev
 ↓
http://localhost:4566
 ↓
Floci
```

---

# 14. Create an S3 Bucket

Run:

```bash
aws s3 mb s3://floci-test-bucket --profile dev
```

Expected:

```text
make_bucket: floci-test-bucket
```

List buckets:

```bash
aws s3 ls --profile dev
```

Expected:

```text
2026-09-02 13:27:36 floci-test-bucket
```

This bucket exists inside the local Floci environment, not in the real AWS account.

---

# 15. Test File Upload

Create a test file:

```bash
echo "Hello from Floci" > hello.txt
```

Upload:

```bash
aws s3 cp hello.txt s3://floci-test-bucket/ --profile dev
```

List files:

```bash
aws s3 ls s3://floci-test-bucket/ --profile dev
```

Download:

```bash
aws s3 cp s3://floci-test-bucket/hello.txt downloaded.txt --profile dev
```

Verify:

```bash
cat downloaded.txt
```

Expected:

```text
Hello from Floci
```

---

# 16. Useful Floci Commands

### Start Floci

```bash
docker compose up -d
```

### Stop Floci

```bash
docker compose down
```

### Check Floci container

```bash
docker ps
```

### View Floci logs

```bash
docker compose logs -f floci
```

### List AWS CLI profiles

```bash
aws configure list-profiles
```

### Check `dev` configuration

```bash
aws configure list --profile dev
```

### List Floci S3 buckets

```bash
aws s3 ls --profile dev
```

---

# 17. Important: `dev` vs Production

The intended setup is:

```text
                    AWS CLI
                       │
             ┌─────────┴─────────┐
             │                   │
          dev profile       production profile
             │                   │
             ▼                   ▼
          Floci                AWS
             │                   │
 localhost:4566             Real AWS
```

### `dev`

```text
Profile: dev
Credentials: test/test
Endpoint: http://localhost:4566
Purpose: Local development/testing
```

### `production`

```text
Profile: production
Credentials: Real AWS authentication
Endpoint: AWS default endpoint
Purpose: Production AWS
```

**Do not configure the production profile to use the Floci endpoint.**

---

# 18. Why We Don't Use `AWS_ENDPOINT_URL` Globally

Floci's official quick start shows the environment-variable approach:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
```

However, for a machine that will eventually access production AWS, a dedicated AWS CLI profile is safer because the endpoint is isolated to `dev`.

Avoid permanently putting this in `.bashrc`:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
```

Otherwise, you could accidentally route commands intended for another environment to Floci.

Our approach is:

```bash
aws s3 ls --profile dev
```

for Floci.

Later:

```bash
aws sts get-caller-identity --profile production
```

for real AWS.

---

# 19. Production Safety Check

Before using the future `production` profile for any AWS operation, always verify the account identity:

```bash
aws sts get-caller-identity --profile production
```

This should be your first verification step before production operations.

For production work, confirm:

```text
Account ID
User/Role ARN
Expected AWS account
Expected profile
```

Never assume the profile is pointing to the correct AWS account.

---

# 20. Endpoint Override

If you want to explicitly override the configured endpoint for a single command, AWS CLI supports:

```bash
aws s3 ls \
  --profile dev \
  --endpoint-url http://localhost:4566
```

The command-line `--endpoint-url` takes precedence over configured endpoint settings.

This is useful when troubleshooting.

---

# 21. Current Verified Setup

At the time this document was created, the following was successfully tested:

```text
AWS CLI
    │
    ▼
Profile: dev
    │
    ├── Access Key: test
    ├── Secret Key: test
    ├── Region: us-east-1
    └── Endpoint: http://localhost:4566
             │
             ▼
           Floci
             │
             ▼
       S3: floci-test-bucket
```

Verified command:

```bash
aws s3 ls --profile dev
```

Successfully returned:

```text
2026-09-02 13:27:36 floci-test-bucket
```

Therefore, the **Floci + AWS CLI `dev` profile setup is working correctly**.

---

# 22. Quick Setup Reference

For future installations:

```bash
# 1. Create directory
mkdir -p ~/floci-test
cd ~/floci-test

# 2. Create docker-compose.yml
nano docker-compose.yml

# 3. Start Floci
docker compose up -d

# 4. Create AWS CLI profile
aws configure --profile dev

# Use:
# Access Key: test
# Secret Key: test
# Region: us-east-1
# Output: json

# 5. Configure Floci endpoint
aws configure set endpoint_url http://localhost:4566 --profile dev

# 6. Verify
aws configure list --profile dev

# 7. Test
aws s3 ls --profile dev

# 8. Create test bucket
aws s3 mb s3://floci-test-bucket --profile dev

# 9. Verify bucket
aws s3 ls --profile dev
```

## Final expected result

```text
dev profile
    ↓
http://localhost:4566
    ↓
Floci
    ↓
AWS-compatible local services
```

### References

- [Floci Quick Start](https://floci.io/floci/getting-started/quick-start/?utm_source=chatgpt.com)
- [Floci AWS CLI & SDK Setup](https://floci.io/floci/getting-started/aws-setup/?utm_source=chatgpt.com)
- [Floci Installation](https://floci.io/floci/getting-started/installation/?utm_source=chatgpt.com)
- [AWS CLI — Configuration and credential files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html?utm_source=chatgpt.com)
- [AWS CLI — Using endpoints](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-endpoints.html?utm_source=chatgpt.com)