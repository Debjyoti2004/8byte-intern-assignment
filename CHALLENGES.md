# Technical Challenges & Solutions

During the development of this assignment, I encountered and resolved the following issues:

### 1. `npm ci` Failure in Docker Build
- **Issue:** The Docker build failed with the error `npm ci can only install with an existing package-lock.json`.
- **Cause:** My `COPY` instruction was `COPY package.json ./`, which excluded the lockfile required for deterministic builds.
- **Solution:** Updated the Dockerfile to use `COPY package*.json ./` to include both the manifest and the lockfile.

### 2. Terraform Key Pair Path Error
- **Issue:** Terraform failed with `InvalidKeyPair.NotFound`.
- **Cause:** I mistakenly provided the local file path (`/Users/.../AWS-key.pem`) to the `key_name` variable. AWS requires the logical name of the key pair stored in the cloud.
- **Solution:** Corrected the `terraform.tfvars` variable to `AWS-key`.